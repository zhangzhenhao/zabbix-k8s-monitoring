# Kubernetes Alertmanager → Zabbix 7.0 集成

把 Kubernetes 集群中 Alertmanager 的活跃告警接入 Zabbix，用 Zabbix 原生的 Problems / Actions / 通知 / 恢复机制完成告警的展示与发送。

## 1. 集成方案

采用 **pull（拉取）模式**：Zabbix Server 通过 HTTP agent 周期拉取 Alertmanager v2 API，不修改 Alertmanager 配置、无需在集群里额外部署 webhook 服务。

数据流：

```
Alertmanager /api/v2/alerts
        │  HTTP agent（每 1m，可调）
        ▼
am.alerts.raw（TEXT，JS 预处理）
        │  过滤 active → 按 fingerprint 去重 → 字段归一化 → 按 severity 分桶
        ▼
5 条 dependent LLD 规则（critical / error / warning / info / other）
        │  lld_macro_paths 把每条告警的字段映射为 {#FINGERPRINT} 等宏
        ▼
每条告警 2 个 dependent item
  ├─ am.alert.state.<bucket>[fp]   1=告警存在，0=告警已消失
  └─ am.alert.detail.<bucket>[fp]  该告警的归一化明细
        ▼
trigger prototype：last(/.../am.alert.state.<bucket>[fp])=1
        ▼
Zabbix Problems → Actions → 邮件/Webhook/IM 等（完全使用 Zabbix 原生能力）
```

选择 pull 而非 Alertmanager webhook 推送的原因：

- 零侵入：不改 Alertmanager 配置，不需要额外的 trapper 接收端；
- 可控：过滤条件、拉取频率、字段裁剪都在 Zabbix 侧通过宏调整；
- 可观测：Alertmanager 不可达时由 `nodata()` 触发器直接告警。

### Severity 映射

Zabbix 的 trigger priority 不支持 LLD 宏，因此按 severity 拆成 5 条发现规则：

| Alertmanager severity | Zabbix trigger priority |
| --- | --- |
| critical | Disaster (5) |
| error | High (4) |
| warning | Average (3) |
| info | Warning (2) |
| 缺失 / 其他 | Information (1) |

### 恢复语义

- 告警从 Alertmanager 消失后，对应 state item 取值变为 `0`，`last(...)=1` 不再成立，problem **自动恢复**（已用 mock 验证）。
- 发现规则 `lifetime: 7d`：告警消失后对象保留 7 天，方便查看历史并避免指纹抖动导致反复创建；7 天后自动删除。
- master item `nodata(10m)` 触发 WARNING，用于 Alertmanager 不可达 / 拉取失败场景。

## 2. Zabbix 7.0 测试环境（集群内单副本）

[deploy/zabbix/k8s.yaml](file:///home/zzh/source-code/k8s-monitoring/deploy/zabbix/k8s.yaml) 包含 PostgreSQL 16、Zabbix Server、Zabbix Web（nginx），均为单副本，数据库使用 `local-path` 5Gi PVC。

```bash
kubectl apply -f deploy/zabbix/k8s.yaml
kubectl -n zabbix get pods

# 本地访问 Web
kubectl -n zabbix port-forward svc/zabbix-web 18080:8080 --address 127.0.0.1
```

- Web：<http://127.0.0.1:18080>
- 默认账号：`Admin` / `zabbix`
- 当前实测版本：Zabbix 7.0.32

## 3. 前置条件：让 Zabbix 能访问 Alertmanager

Zabbix Server 运行在集群内，可直接使用 Service DNS。先确认 Alertmanager 地址：

```bash
kubectl get svc -A | grep -i alertmanager
```

本环境为 `rancher-monitoring-alertmanager.cattle-monitoring-system.svc:9093`，对应模板宏默认值。若你的 Alertmanager 在其他命名空间 / 端口，或需要走集群外地址，修改主机宏 `{$ALERTMANAGER.URL}` 即可。

## 4. 导入模板

模板文件：[templates/zabbix/k8s_alertmanager_by_http.yaml](file:///home/zzh/source-code/k8s-monitoring/templates/zabbix/k8s_alertmanager_by_http.yaml)

Web UI：`Data collection → Templates → Import`，选择上面的 YAML（勾选 Templates / Items / Triggers / Discovery rules 相关选项）。

导入后创建主机：

1. `Data collection → Hosts → Create host`；
2. Host name 例如 `k8s-alertmanager`，加入主机组 `Kubernetes`；
3. 链接模板 `Kubernetes Alertmanager by HTTP`；
4. **不需要配置 Agent/SNMP 接口**（所有监控项都是 HTTP agent / dependent）。

### 宏

| 宏 | 默认值 | 说明 |
| --- | --- | --- |
| `{$ALERTMANAGER.URL}` | `http://rancher-monitoring-alertmanager.cattle-monitoring-system.svc:9093` | Alertmanager 基础 URL（不带结尾斜杠） |
| `{$ALERTMANAGER.ALERT_FILTER_QUERY}` | `active=true&silenced=false&inhibited=false` | v2 API 查询参数 |
| `{$ALERTMANAGER.POLL_INTERVAL}` | `1m` | 拉取间隔 |
| `{$ALERTMANAGER.NODATA_PERIOD}` | `10m` | 超过该时间无数据则触发可用性告警 |

## 5. 展示与告警发送

- **原始数据**：`Monitoring → Latest data` 中 `am.alerts.raw`，内容为归一化后的
  `{"total":n,"counts":{...},"groups":{"critical":[...],...}}`。
- **发现项**：每条活跃告警生成 `am.alert.state.*`（1/0）与 `am.alert.detail.*`（明细 JSON）。
- **Problems**：每条活跃告警对应一个 problem，名称 `K8s Alert: {#ALERTNAME} [{#SEVERITY}]`，描述中包含 summary/description/namespace/pod/instance/job/startsAt/runbook，runbook 同时写入 problem URL。
- **Tags**：`source=alertmanager`、`severity`、`alertname`、`namespace`、`pod`，可直接用于 Action 条件、告警分组和抑制。
- **发送**：在 `Alerts → Actions → Trigger actions` 中按需创建动作（例如 `tag severity = critical` 才发送），配置 Media types（Email / Webhook / 企业微信 / 飞书等）即可；告警升级、确认、维护期等均沿用 Zabbix 原生流程。

## 6. 已验证结果（本环境实测）

- master item 单次拉取约 66 条活跃告警，归一化负载约 37 KB；
- 自动发现 66 个 state item（64 个为 1）与对应 detail item；
- 生成 64 个 problem：Disaster 4 / Average 57 / Warning 2 / Information 1；
- mock Alertmanager 测试：告警消失后 state 变为 0，problem 自动恢复；
- 停掉 mock 后 `nodata(1m)` 触发器按预期产生 WARNING problem。

## 7. 实现要点与限制

- **TEXT 上限 64 KB**：master item 的归一化 JSON 必须精简，字段已截断（summary 200 / description 300 / runbook 150 字符），超长会被截断并导致 JSON 非法。
- **LLD 宏需显式映射**：7.0 中 JSONPath 发现不会自动把 JSON key 变成 `{#MACRO}`，必须在每条发现规则上配置 `lld_macro_paths`（模板已包含 15 个宏映射）。
- **trigger priority 不支持 LLD 宏**，故按 severity 拆分为 5 条发现规则。
- item prototype 的 `error_handler` 使用 `DISCARD_VALUE`。
- Alertmanager 不可达时 master item 变为 unsupported，依赖项不会被误判恢复，由 nodata 触发器负责提示。
- 若 Alertmanager 启用了 Basic Auth / TLS，需在 master item 上补充认证与 SSL 配置（模板默认未启用）。

## 8. 清理

```bash
kubectl delete -f deploy/zabbix/k8s.yaml
```
