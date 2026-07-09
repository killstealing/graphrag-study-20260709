# GraphRAG 图数据库篇 - 导入脚本问题排查记录

## 问题描述

在执行 `parallel_batched_import` 并行导入数据到 Neo4j 时，出现以下错误：

```
批次0 （行100-0） 处理失败：{neo4j_code: Neo.ClientError.Statement.ParameterMissing} {message: Expected parameter(s): row}
批次 0 失败：-99 行，耗时：0.83秒，错误：None
进度：1/1 批次完成 (100.0%)
导入完成：总计 1 行，成功0 行，失败-99行
总耗时：0.84秒，平均速度:0.00行/秒
```

---

## 问题排查与修复

### Bug 1：批次起始位置计算错误

**位置：** `process_batch` 函数内，第 376 行

**问题：** 使用加法 `+` 而非乘法 `*`，导致批次起始位置计算错误

```python
# 修改前（错误）
start = batch_idx + batch_size

# 修改后（正确）
start = batch_idx * batch_size
```

**影响：** 第 0 个批次的 `start = 0 + 100 = 100`，超出数据范围，导致实际处理 0 行数据（显示为 `-99` 行）。

---

### Bug 2：Cypher 参数名与 Python 传参关键字不匹配（直接导致报错）

**位置：** `session.run()` 调用处，第 391 行

**问题：** Cypher 语句中引用 `$row`，但 `session.run()` 传入的关键字参数是 `rows=`，Neo4j 找不到名为 `row` 的参数

```python
# 修改前（错误）
result = session.run(
    "UNWIND $row as value " + statement, rows=batch.to_dict("records"))

# 修改后（正确）
result = session.run(
    "UNWIND $row as value " + statement, row=batch.to_dict("records"))
```

**影响：** Neo4j 抛出 `Neo.ClientError.Statement.ParameterMissing` 错误，提示 `Expected parameter(s): row`。

**经验总结：** Neo4j Python Driver 中，Cypher 语句中的参数占位符（如 `$row`）必须与 `session.run()` 方法传入的关键字参数名**完全一致**。

---

### Bug 3：异常返回字典键名与打印时不匹配

**位置：** `except` 块中的返回字典，第 411 行

**问题：** 异常时返回字典使用的键是 `counters`，但打印时取的是 `error` 键，导致错误信息显示为 `None`

```python
# 修改前（错误）
return {
    "batch": batch_idx,
    "rows": end - start,
    "success": False,
    "duration": batch_duration,
    "counters": str(e)   # 键名不匹配
}

# 修改后（正确）
return {
    "batch": batch_idx,
    "rows": end - start,
    "success": False,
    "duration": batch_duration,
    "error": str(e)      # 与打印时 result.get('error') 对应
}
```

**影响：** 不影响导入功能，但失败时错误信息显示为 `None`，无法定位问题。

---

### Bug 4：调用代码拼写错误

**位置：** 最后一个 cell，第 668 行

**问题：** `create_document_nodes(df_documents)1111` 多了 `1111`

```python
# 修改前（错误）
create_document_nodes(df_documents)1111

# 修改后（正确）
create_document_nodes(df_documents)
```

---

## 修复后完整代码参考

```python
def parallel_batched_import(statement, df, batch_size=100, max_workers=8):
    total = len(df)
    batches = (total + batch_size - 1) // batch_size
    start_time = time.time()
    results = []

    print(f"开始并行导入 {total} 行数据，分为{batches} 个批次，每批{batch_size} 条")

    def process_batch(batch_idx):
        start = batch_idx * batch_size          # Bug1 修复：+ 改为 *
        end = min(start + batch_size, total)
        batch = df.iloc[start:end]

        batch_start_time = time.time()
        try:
            with driver.session() as session:
                result = session.run(
                    "UNWIND $row as value " + statement,
                    row=batch.to_dict("records")  # Bug2 修复：rows 改为 row
                )
                summary = result.consume()
                batch_duration = time.time() - batch_start_time

                return {
                    "batch": batch_idx,
                    "rows": end - start,
                    "success": True,
                    "duration": batch_duration,
                    "counters": summary.counters
                }
        except Exception as e:
            batch_duration = time.time() - batch_start_time
            print(f"批次{batch_idx} （行{start}-{end-1}） 处理失败：{str(e)}")

            return {
                "batch": batch_idx,
                "rows": end - start,
                "success": False,
                "duration": batch_duration,
                "error": str(e)               # Bug3 修复：counters 改为 error
            }

    with concurrent.futures.ThreadPoolExecutor(max_workers=max_workers) as executor:
        futures = [executor.submit(process_batch, i) for i in range(batches)]

        for i, future in enumerate(concurrent.futures.as_completed(futures)):
            result = future.result()
            results.append(result)

            if result['success']:
                print(f"批次 {result['batch']} 完成：{result['rows']} 行，耗时：{result['duration']:.2f}秒")
            else:
                print(f"批次 {result['batch']} 失败：{result['rows']} 行，耗时：{result['duration']:.2f}秒，错误：{result.get('error')}")
            print(f"进度：{i+1}/{batches} 批次完成 ({((i+1)/batches*100):.1f}%)")

    successful_rows = sum(r['rows'] for r in results if r['success'])
    failed_rows = sum(r['rows'] for r in results if not r['success'])

    duration = time.time() - start_time
    rows_per_second = successful_rows / duration if duration > 0 else 0

    print(f"导入完成：总计 {total} 行，成功{successful_rows} 行，失败{failed_rows}行")
    print(f"总耗时：{duration:.2f}秒，平均速度:{rows_per_second:.2f}行/秒")

    return {
        "total_rows": total,
        "successful_rows": successful_rows,
        "failed_rows": failed_rows,
        "duration_seconds": duration,
        "rows_per_second": rows_per_second,
        "batch_results": results
    }
```
