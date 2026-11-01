# 本公司服务日志规范（讨论稿）

> 供业务方评审的原型，不是已生效的开发规范；不代表旧服务已改造。

## 适用与目标

适用于今后新开发或修改的本公司服务。已有服务暂按现有日志采集。让客户现场的采集 Agent 能准确识别服务、时间、错误类别和关联事件；让院内小模型分析筛选后的异常日志时有上下文和依据。中间件及 Windows 系统日志由采集侧适配，不要求第三方修改输出格式。

## 建议的记录约定

建议每条记录为一条 UTF-8 JSON 对象（JSON Lines），字段含义与 [OpenTelemetry Logs 数据模型](https://opentelemetry.io/docs/specs/otel/logs/data-model/) 对齐；**不要求引入 OpenTelemetry SDK 或直接输出 OTLP**。日志产生时间和采集端看到的时间是两个不同概念；后者由采集系统补充，不能伪装为服务自己的时间。

| 字段（建议名） | 本公司服务写入规则 | 用途 |
| --- | --- | --- |
| `timestamp` | 必填，带时区的事件发生时间 | 按时间关联；时间不可信时采集侧另记观察时间 |
| `service.name` | 必填、稳定的逻辑服务名 | 跨主机辨认同一服务；参考 [OTel service.name](https://opentelemetry.io/docs/specs/semconv/resource/service/) |
| `severity` | 必填，取 `TRACE`、`DEBUG`、`INFO`、`WARN`、`ERROR`、`FATAL` 之一 | 筛选候选异常；等级并不等于告警 |
| `message` | 必填，简短描述发生的事，不直接嵌入患者资料 | 人工阅读与模型概括 |
| `event.name` | 对可供监控判断的事件必填，稳定的事件类别名；普通诊断日志可省略 | 聚合相同类型的错误，避免依赖变动的文字 |
| `service.version` | 能获取时写入 | 关联新旧版本的异常 |
| `error.code` | 错误事件如有稳定错误码则写入 | 区分错误类型；不要把动态异常文本当错误码 |
| `correlation.id` | 请求、任务或批处理有安全的关联标识时写入 | 串起同一次操作；不写原始会话令牌 |

主机与实例身份可由采集端在确定来源后附加；不得猜测并填入错误身份。同一个错误类型使用稳定的 `event.name`，变化的详情放在独立字段或安全的 `message` 中。发生事件的时间、服务名、错误类型、原始等级须保留原义。业务方若已有请求 ID、日志等级和错误码约定，应先映射，避免双重标准。

### 示例（虚构，无真实医院数据）

```json
{"timestamp":"2026-11-01T09:03:12+08:00","service.name":"report-worker","service.version":"2.4.0","severity":"ERROR","event.name":"DEPENDENCY_UNAVAILABLE","error.code":"STORAGE_TIMEOUT","correlation.id":"job-7f3a","message":"存储操作超时"}
```

此记录**不直接等于正式告警**：确定性故障按规则产生告警；筛选后的日志可以交给院内模型判断是否值得形成标明模型来源、待人工核实的告警。

## 推荐首先覆盖的事件

服务启动、退出、无法启动；关键依赖不可用及恢复；不可重试的任务失败；持续的资源限制或队列处理失败。各产品自行列出稳定事件类别及含义，不把每次普通重试或每条 `ERROR` 都定为正式告警。[OWASP 日志指南](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html#which-events-to-log) 还建议记录与安全相关的认证/授权失败、配置变更等，但是否进入本监控项目的告警范围需单独决定。

## 敏感数据底线

应用写日志前排除或替换患者健康信息、口令、访问令牌、原始会话 ID、连接串及密钥；异常详情和第三方错误文本也可能包含这些内容，不能原样拼接到 `message`。[OWASP 的排除与脱敏建议](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html#data-to-exclude) 是此条的依据。保留排障所需的安全错误码与关联标识。具体脱敏方式、日志保存期限及访问审计不在本原型拍定；院外邮件和公司项目管理系统只接收已确认的摘要，不传原始日志。

## 评审时需要决定

1. 新服务采用上述 JSON Lines 格式，还是只要求字段语义一致、输出格式由各服务选择？
2. 哪些现有日志等级、服务名称和错误码已有公司约定，需要沿用或映射？
3. 哪些事件要成为公司统一的稳定 `event.name`，哪些属于各产品自定义？
4. 现有服务修改时从哪些模块开始遵循，是否允许某些无法结构化的旧日志暂时例外？

## 参考

- [OpenTelemetry Logs 数据模型](https://opentelemetry.io/docs/specs/otel/logs/data-model/)：稳定的逻辑记录语义，不强制具体输出编码。
- [OpenTelemetry 服务语义约定](https://opentelemetry.io/docs/specs/semconv/resource/service/)：逻辑服务身份及版本。
- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)：应用日志的事件属性、相关性及敏感信息排除。
