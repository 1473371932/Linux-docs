<details>
<summary><strong> 问题：Agent 没采集到某些已经支持的协议 </strong></summary>

随着 DeepFlow 支持解析的协议越来越多，为了优化资源/存储消耗，在 v6.5 版本后默认[仅采集部分协议](https://www.deepflow.io/docs/zh/features/l7-protocols/overview/#%E6%94%AF%E6%8C%81%E7%9A%84%E5%BA%94%E7%94%A8%E5%8D%8F%E8%AE%AE)：
1. 通过 [deepflow-ctl](https://www.deepflow.io/docs/zh/best-practice/agent-advanced-config/) 命令行工具配置 agent-group-config
2. [添加对应协议](https://www.deepflow.io/docs/zh/configuration/agent/#processors.request_log.application_protocol_inference.enabled_protocols)的解析

</details>


<details>
<summary><strong> 问题：Agent 不同部署方式支持的组配置项是否有差异 </strong></summary>

1. 不同部署方式仅在部署流程中使用的组配置项会有差异，详看部署文档
1. 部署后数据采集、自身相关的配置无参数差异

</details>
