# 容器化应用接入 SigNoz 可观测平台实践

当应用逐步演进为多个容器和微服务后，登录服务器逐个执行 `docker logs` 已经很难满足故障定位需求。日志分散、上下文缺失、时间格式不一致，以及无法关联指标和链路，都会显著增加排障成本。

本文以一个经过脱敏的 Docker Compose 应用为例，介绍如何使用 Fluent Bit 采集容器日志并发送到 SigNoz，同时说明常见问题和生产落地建议。文中的应用名称、服务名称、镜像版本、地址、目录和查询示例均为通用占位符，不对应任何真实生产环境。

## 一、为什么需要统一可观测平台

传统日志排查通常依赖：

```bash
docker logs -f <container_name>
```

这种方式适合临时调试，但在生产环境中存在明显局限：

- 多个服务的日志分散在不同容器中。
- 容器重建后，历史日志可能难以追踪。
- 无法按服务、日志级别、请求 ID 等字段统一检索。
- 无法基于错误日志配置实时告警。
- 日志、指标和调用链相互割裂。

SigNoz 是一个基于 OpenTelemetry 的可观测平台，可以统一处理日志、指标和链路追踪。其自托管架构通常由 OpenTelemetry Collector、ClickHouse 和 SigNoz 查询与展示服务组成。

对于希望采用开放标准、需要自托管，并且希望逐步建立日志、指标、链路统一观测能力的团队，SigNoz 是一个值得考虑的方案。

## 二、整体接入架构

本文采用如下日志链路：

```text
业务容器 stdout / stderr
          ↓
Docker Fluentd Logging Driver
          ↓
Fluent Bit
          ↓
OpenTelemetry Collector
          ↓
ClickHouse
          ↓
SigNoz Logs / Dashboard / Alert
```

| 组件 | 职责 |
| --- | --- |
| 业务容器 | 将应用日志输出到标准输出和标准错误 |
| Docker Logging Driver | 将容器日志转发给 Fluent Bit |
| Fluent Bit | 接收、清洗、补充字段并转发日志 |
| OpenTelemetry Collector | 接收和处理遥测数据 |
| ClickHouse | 存储并查询日志、指标和链路数据 |
| SigNoz | 提供查询、Dashboard、告警和关联分析 |

## 三、接入前的设计选择

### 1. 采集协议怎么选

Fluent Bit 到 SigNoz 通常有两条接入路线。

#### 路线一：Fluent Forward

SigNoz 官方文档支持让 Fluent Bit 使用 Forward 协议，将日志发送到启用了 `fluentforward` receiver 的 OpenTelemetry Collector。

优点：

- 配置简单。
- 不需要手动拼装 OTLP JSON。
- 适合已有 Fluent Bit 或 Fluentd 环境。

#### 路线二：OTLP/HTTP

新版 Fluent Bit 提供原生 OpenTelemetry output，可以将日志发送到 OpenTelemetry HTTP endpoint。

优点：

- 使用 OpenTelemetry 标准数据模型。
- 方便逐步统一日志、指标和链路。
- 减少自定义格式转换逻辑。

对于受旧版 Fluent Bit 或历史配置限制的环境，也可以通过 Lua 把日志转换为 OTLP JSON，再使用 HTTP output 上报。但这更适合作为兼容方案，不建议长期依赖大量单行 Lua 代码。

### 2. 日志字段怎么规划

在接入平台前，建议至少统一以下字段：

| 字段 | 用途 |
| --- | --- |
| `service.name` | 区分应用服务 |
| `deployment.environment` | 区分生产、测试等环境 |
| `service.version` | 定位版本相关问题 |
| `log.level` | 按 ERROR、WARN 等级筛选 |
| `trace_id` | 关联调用链 |
| `request_id` | 跟踪单次业务请求 |

日志标签应该保持稳定，避免直接使用动态容器 ID 作为主要查询字段。

## 四、配置 Docker Compose 日志转发

假设应用包含 API、Worker 和 Web 三个服务，可以为每个容器配置 `fluentd` 日志驱动，将日志发送到 Fluent Bit：

```yaml
services:
  api-service:
    image: example/api-service:${APP_VERSION}
    logging:
      driver: fluentd
      options:
        fluentd-address: ${FLUENT_BIT_HOST}:24224
        fluentd-async-connect: "true"
        fluentd-retry-wait: "1s"
        fluentd-max-retries: "120"
        tag: app-api

  worker-service:
    image: example/worker-service:${APP_VERSION}
    logging:
      driver: fluentd
      options:
        fluentd-address: ${FLUENT_BIT_HOST}:24224
        fluentd-async-connect: "true"
        tag: app-worker

  web-service:
    image: example/web-service:${APP_VERSION}
    logging:
      driver: fluentd
      options:
        fluentd-address: ${FLUENT_BIT_HOST}:24224
        fluentd-async-connect: "true"
        tag: app-web
```

这里使用环境变量代替真实地址和镜像版本：

```dotenv
APP_VERSION=<application-version>
FLUENT_BIT_HOST=<fluent-bit-host>
```

`fluentd-async-connect` 可以避免 Fluent Bit 暂时不可用时阻塞容器启动，`tag` 则用于标识日志来源并映射为 `service.name`。

需要注意：远程日志服务不可用时，Docker logging driver 的行为可能影响业务容器。生产环境应测试 Collector 故障、网络中断和磁盘压力下的表现。

## 五、Fluent Bit 接入方案

### 1. 推荐方案：转发给 OpenTelemetry Collector

Fluent Bit 可以接收 Docker logging driver 发来的 Forward 日志，再转发到 OpenTelemetry Collector：

```ini
[SERVICE]
    Log_Level  info

[INPUT]
    Name       forward
    Listen     0.0.0.0
    Port       24224

[FILTER]
    Name       modify
    Match      app-*
    Remove     container_id
    Remove     source

[OUTPUT]
    Name       forward
    Match      app-*
    Host       ${OTEL_COLLECTOR_HOST}
    Port       ${OTEL_FLUENT_FORWARD_PORT}
```

OpenTelemetry Collector 侧需要启用 `fluentforward` receiver，并将它加入 logs pipeline：

```yaml
receivers:
  fluentforward:
    endpoint: 0.0.0.0:24224

processors:
  batch: {}

exporters:
  otlp:
    endpoint: <signoz-ingestion-endpoint>
    tls:
      insecure: true

service:
  pipelines:
    logs:
      receivers: [fluentforward]
      processors: [batch]
      exporters: [otlp]
```

具体 exporter 和端口应根据 SigNoz 部署方式调整，不要直接照搬示例中的占位符。

### 2. 新版 Fluent Bit：使用 OpenTelemetry output

新版 Fluent Bit 提供原生 OpenTelemetry output，可以将日志发送到 OTLP HTTP endpoint：

```ini
[OUTPUT]
    Name      opentelemetry
    Match     app-*
    Host      ${OTEL_COLLECTOR_HOST}
    Port      4318
    Logs_URI  /v1/logs
    Tls       Off
```

不同 Fluent Bit 版本支持的参数可能不同，部署前应对照对应版本官方文档验证配置。

### 3. 旧版本兼容方案：Lua 转换 OTLP JSON

某些旧环境无法直接使用原生 OpenTelemetry output，可能需要使用 Lua 将日志转换为 OTLP JSON，再通过 HTTP output 上报：

```ini
[FILTER]
    Name   lua
    Match  app-*
    Call   to_otlp
    Code   function to_otlp(tag,ts,rec) local s=rec.app or tag or 'unknown-service'; local l=rec.log or rec.message or ''; local t=tostring(os.time())..'000000000'; local o={resourceLogs={{resource={attributes={{key='service.name',value={stringValue=s}}}},scopeLogs={{logRecords={{timeUnixNano=t,body={stringValue=l}}}}}}}}; return 2,ts,o; end

[OUTPUT]
    Name          http
    Match         app-*
    Host          ${OTEL_COLLECTOR_HOST}
    Port          4318
    URI           /v1/logs
    Format        json_stream
    Json_Date_Key false
    Tls           Off
    Header        Content-Type application/json
```

这段配置解决两个常见兼容问题：

- 某些旧版本对多行 Lua 和缩进解析较严格，需要将 `Code` 保持为单行。
- `timeUnixNano` 必须以纯数字字符串表达，避免被序列化为科学计数法后导致 Collector 解析失败。

这类手工 OTLP 封装比较脆弱。条件允许时，建议升级 Fluent Bit，或让 OpenTelemetry Collector 接收 Fluent Forward 数据并负责标准化。

## 六、日志结构化与脱敏

日志进入平台只是第一步。要提高检索和告警质量，还需要结构化处理。例如，业务日志可以采用 JSON 输出：

```json
{
  "timestamp": "2026-01-01T12:00:00Z",
  "level": "ERROR",
  "message": "request processing failed",
  "request_id": "example-request-id",
  "trace_id": "example-trace-id"
}
```

然后在 Fluent Bit 或 OpenTelemetry Collector 中完成：

- JSON 解析和字段重命名。
- 添加 `service.name` 和环境标签。
- 删除无价值或高基数字段。
- 对敏感字段进行删除、掩码或 hash。

不建议把密码、Token、Cookie、身份证号、手机号等敏感内容写入日志。日志平台扩大了数据可见范围，也会放大敏感信息泄露风险。

## 七、服务重启与功能验证

修改 Docker Compose 日志驱动后，需要重建或重启相关容器：

```bash
docker compose down
docker compose up -d
docker compose restart fluent-bit
```

检查 Fluent Bit：

```bash
docker logs -f <fluent-bit-container> --tail 50
```

正常状态通常包括 Fluent Bit 启动成功、Forward input 正常监听、没有配置语法错误，以及日志转发请求成功。

检查 OpenTelemetry Collector：

```bash
docker logs -f <otel-collector-container> --tail 100
```

重点关注 receiver 是否成功启动、是否存在数据格式错误、exporter 是否持续重试，以及 Collector 是否因内存压力触发限流。

进入 SigNoz Logs 页面后：

1. 扩大查询时间范围，避免日志因时间窗口过小而不可见。
2. 按 `service.name` 筛选不同服务。
3. 搜索一条已知测试日志。
4. 检查时间戳、正文和资源字段是否正确。
5. 验证 ERROR 日志能否触发测试告警。

## 八、常见问题排查

| 问题现象 | 常见原因 | 处理建议 |
| --- | --- | --- |
| Fluent Bit 启动失败，提示缩进错误 | 旧版本配置解析严格，或内嵌 Lua 换行错误 | 使用标准缩进；兼容场景将 Lua Code 保持为单行 |
| 上报返回 HTTP 400 | OTLP JSON 结构错误或时间戳格式错误 | 检查 `/v1/logs` 请求体；避免纳秒时间戳被转为科学计数法 |
| Fluent Bit 无法收到日志 | Docker logging driver 地址或网络错误 | 检查容器配置、端口监听、防火墙和路由 |
| Collector 收到数据但 SigNoz 无日志 | pipeline/exporter 配置错误，或查询时间范围不正确 | 检查 Collector 日志并扩大查询时间范围 |
| 日志服务名混乱 | tag、容器名和 `service.name` 缺少统一规则 | 建立稳定的服务命名规范 |
| 查询性能下降 | 高基数字段过多或日志量失控 | 删除无价值字段，控制保留周期和索引策略 |
| 网络中断后日志丢失 | 缓冲和重试策略不足 | 配置文件系统缓冲，并验证故障恢复能力 |

## 九、从日志平台演进为可观测平台

日志接入完成后，可以逐步补齐另外两类遥测数据：

- 使用 OpenTelemetry SDK 或自动插桩采集 Trace，并在日志中注入 `trace_id`。
- 采集请求量、错误率、P95/P99 延迟、CPU、内存、队列堆积和 Collector 状态等指标。
- 配置 ERROR 日志、错误率、延迟和 Collector exporter 失败告警。

告警需要有明确负责人和处置动作，否则很容易演变为无人关注的噪声。

## 十、SigNoz 与其他可观测平台对比

不同平台面向的重点和使用成本不同。下面是适合技术选型初期使用的概括性比较，实际能力和授权模式应以各产品最新官方文档为准。

| 维度 | SigNoz | Splunk Observability Cloud / Splunk Platform | Grafana 可观测栈 | Elastic Stack |
| --- | --- | --- | --- | --- |
| 核心定位 | OpenTelemetry 原生的统一可观测平台 | 企业级日志分析、安全分析与全栈可观测能力 | Grafana、Loki、Tempo、Prometheus/Mimir 等组件组合 | 搜索分析、日志管理与可观测能力 |
| 数据采集 | OpenTelemetry 优先，也支持多种日志采集器 | 支持 Splunk Distribution of OpenTelemetry Collector 及丰富采集生态 | Prometheus、OpenTelemetry、Alloy 等 | Elastic Agent、Beats、Logstash、OpenTelemetry 等 |
| 存储与查询 | 自托管常用 ClickHouse | 托管平台及 Splunk 数据平台能力 | 根据组件分别存储指标、日志和链路 | Elasticsearch |
| 部署方式 | 云服务或自托管 | 以商业产品和托管服务为主 | 云服务或自建组合栈 | 云服务或自托管 |
| 运维复杂度 | 单平台体验较统一，自托管仍需维护 ClickHouse 和 Collector | 商业平台能力成熟，需评估许可和数据成本 | 灵活度高，但多组件组合与容量规划更复杂 | 搜索能力强，需要维护集群、索引和生命周期 |
| 适合场景 | 希望基于 OpenTelemetry 快速构建统一可观测平台 | 大型企业、复杂日志分析、安全与治理要求高的场景 | 已有 Prometheus/Grafana 体系的团队 | 重视全文检索并已有 Elastic 技术栈的团队 |

选择平台时，可以从以下问题出发：

1. 是否要求自托管和数据本地化？
2. 是否已经采用 OpenTelemetry？
3. 团队能否维护 ClickHouse、Elasticsearch 或多组件 Grafana 栈？
4. 日志量、保留周期和查询复杂度如何？
5. 是否需要安全分析、审计、SIEM 或复杂数据治理？
6. 是否可以接受商业许可和托管服务成本？

如果团队希望使用 OpenTelemetry 统一日志、指标和链路，并倾向于较轻量的自托管方案，可以重点评估 SigNoz。如果企业已经深度使用 Splunk，或需要成熟的日志分析、安全治理和大规模商业支持，Splunk 可能更合适。如果已有 Prometheus 和 Grafana 生态，继续扩展 Loki、Tempo 或 Mimir 往往迁移成本更低。

## 十一、生产落地建议

1. 优先使用 OpenTelemetry、OTLP 和 Fluent Forward 等开放协议，降低未来更换后端平台的成本。
2. 在主机或集群侧部署采集 Agent，在中心侧部署 Collector Gateway，完成批处理、过滤、脱敏和路由。
3. 设置明确的缓冲、重试、背压和丢弃策略，并演练后端不可用场景。
4. 提前规划每日日志量、峰值写入速率、保留周期、存储容量和归档策略。
5. 在采集链路中删除敏感字段，并对不同团队设置日志访问边界。

## 十二、总结

容器化应用接入 SigNoz 的核心，不只是把日志从 Docker 转发到一个网页中，而是建立一条稳定、标准化、可演进的遥测数据链路。

对于新环境，建议优先使用 Fluent Forward 或 Fluent Bit 原生 OpenTelemetry output，减少手工转换。旧环境可以通过 Lua 和 HTTP output 完成兼容接入，但应把它视为过渡方案。最终目标是让日志、指标和链路基于 OpenTelemetry 形成统一的数据基础，为故障定位、性能优化和容量治理提供可靠依据。

## 参考资料

- [SigNoz Technical Architecture](https://signoz.io/docs/architecture/)
- [SigNoz FluentBit to SigNoz](https://signoz.io/docs/userguide/fluentbit_to_signoz/)
- [SigNoz Logs Management Overview](https://signoz.io/docs/logs-management/overview/)
- [Fluent Bit OpenTelemetry Output](https://docs.fluentbit.io/manual/data-pipeline/outputs/opentelemetry)
- [Splunk Distribution of the OpenTelemetry Collector](https://help.splunk.com/en/splunk-observability-cloud/manage-data/splunk-distribution-of-the-opentelemetry-collector)
