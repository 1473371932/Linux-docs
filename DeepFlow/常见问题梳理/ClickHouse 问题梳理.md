<details>
<summary><strong> 问题：ClickHouse 数据清理机制 </strong></summary>
[Server 默认配置](https://github.com/deepflowio/deepflow/blob/main/server/server.yaml)中通过 `ck-disk-monitor` 参数设置数据清理机制：

- 磁盘空间使用超过 80%，每次触发时清理每个表最老的数据块，直到`磁盘占用 < 80%` 或 `剩余空间 > 100G`
- 建议通过 Server 配置中 `ttl` 相关参数控制数据量，尽可不能不要触发自动清理机制
- 可通过 Server log 中过滤 `drop partition`、`disk us` 关键字查看是否触发清理机制

</details>