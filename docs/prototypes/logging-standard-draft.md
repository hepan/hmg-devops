# 本公司服务日志规范（讨论稿）

> 供业务方评审的原型，不是已生效的开发规范；不代表旧服务已改造。

## 适用与目标

适用于今后新开发或修改的本公司服务；已有服务暂按现有日志采集，不要求立即改造。目标是让客户现场的采集 Agent 能识别服务、时间、等级及关键运维事件，让院内小模型分析筛选后的异常日志时有上下文和依据。第三方中间件及 Windows 系统日志由采集侧适配，不要求第三方修改输出格式。

## 建议的记录约定

暂不统一 JSON 结构或输出格式。各服务沿用适合自身的文本或结构化日志，但应让采集 Agent 能从服务配置与日志中识别以下语义；具体字段名及映射由各产品记录。[OpenTelemetry Logs 数据模型](https://opentelemetry.io/docs/specs/otel/logs/data-model/) 仅作为语义参考，不要求引入其 SDK 或输出 OTLP。日志产生时间与采集端观察时间须区分，后者由采集系统补充。

| 信息语义（示例名，非统一字段名） | 本公司服务记录要求 | 用途 |
| --- | --- | --- |
| `timestamp` | 必填，带时区的事件发生时间 | 按时间关联；时间不可信时采集侧另记观察时间 |
| `service.name` | 必填、稳定的逻辑服务名 | 跨主机辨认同一服务；参考 [OTel service.name](https://opentelemetry.io/docs/specs/semconv/resource/service/) |
| `severity` | 必填，原等级或可映射到 `TRACE`、`DEBUG`、`INFO`、`WARN`、`ERROR`、`FATAL` 的等级 | 筛选候选异常；等级并不等于告警 |
| `message` | 必填，简短描述发生的事，不直接嵌入患者资料 | 人工阅读与模型概括 |
| 稳定事件类别（如 `event.name`） | 关键运维事件记录稳定类别名；普通诊断日志可省略 | 聚合同类错误，不依赖变化的文字 |
| `service.version` | 能获取时写入 | 关联新旧版本的异常 |
| `error.code` | 错误事件如有稳定错误码则写入 | 区分错误类型；不要把动态异常文本当错误码 |
| `correlation.id` | 请求、任务或批处理有安全的关联标识时写入 | 串起同一次操作；不写原始会话令牌 |

主机与实例身份可由采集端在确定来源后附加；不得猜测并填入错误身份。关键运维事件使用稳定类别，变化的详情放在安全的描述或其他已有字段中。发生时间、服务名、错误类型与原始等级须保留原义；已有请求 ID、等级和错误码约定应先映射，避免双重标准。不同日志格式能否稳定提取这些信息是后续需验证的风险。

### 示例（虚构的现有文本格式，非强制模板）

```text
2026-11-01T09:03:12+08:00 ERROR report-worker DEPENDENCY_UNAVAILABLE code=STORAGE_TIMEOUT correlation=job-7f3a 存储操作超时
```

此记录**不直接等于正式告警**：确定性故障按规则产生告警；筛选后的日志可以交给院内模型判断是否值得形成标明模型来源、待人工核实的告警。

## 公司统一的关键运维事件

只有下列事件类别需要跨产品采用相同的稳定名称；其他诊断日志不强制分类，各产品可自行定义额外类别并说明含义：

| 事件类别 | 产生时机 |
| --- | --- |
| `SERVICE_STARTED` | 服务完成启动并可接受自身职责范围内的工作 |
| `SERVICE_STOPPED` | 服务有序退出；进程突然崩溃可能来不及写此事件，仍由进程状态监控识别 |
| `SERVICE_START_FAILED` | 服务启动失败 |
| `DEPENDENCY_UNAVAILABLE` | 服务发现自身依赖不可用，记录依赖类别与安全错误码 |
| `DEPENDENCY_RECOVERED` | 原先不可用的依赖恢复可用 |
| `TASK_FAILED_FINAL` | 任务已终止且不会自动重试 |

事件类别不是告警规则：重复重试及普通 `ERROR` 不自动转成告警。安全审计事件（如认证失败、授权失败和配置变更）可能需要单独规范，不在本监控项目的统一事件清单内；参见 [OWASP 建议的事件范围](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html#which-events-to-log)。

## 敏感数据底线

应用写日志前排除或替换患者健康信息、口令、访问令牌、原始会话 ID、连接串及密钥；异常详情和第三方错误文本也不得未经检查就拼接到描述中。[OWASP 的排除与脱敏建议](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html#data-to-exclude) 是此条的依据。保留排障所需的安全错误码与安全关联标识。具体脱敏机制、保存期限和访问审计留待开发前决定；院外邮件和公司项目管理系统只接收状态摘要，不传原始日志。

## 各产品交付的日志说明

每个新开发或修改的自有服务提供脱敏的正常及异常日志样例，说明时间、服务身份、等级、描述、关键事件类别及可选错误码／关联标识在自身格式中如何识别；列出日志来源、现有等级及错误码与本规范语义的映射。无法稳定提取时须显式记为待解决的采集限制，而不是默认为正常。已有服务不因本建议稿而必须立即改造。

JSON 序列化开销、各格式的采集可靠性及旧系统兼容性均未经过运行验证；本建议稿不承诺性能或自动适配。

## 参考

- [OpenTelemetry Logs 数据模型](https://opentelemetry.io/docs/specs/otel/logs/data-model/)：稳定的逻辑记录语义，不强制具体输出编码。
- [OpenTelemetry 服务语义约定](https://opentelemetry.io/docs/specs/semconv/resource/service/)：逻辑服务身份及版本。
- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)：应用日志的事件属性、相关性及敏感信息排除。
