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

## 9. 集群内 Zabbix Proxy + Agent（官方 K8s 模板接入）

把 `zabbix` namespace 里已经部署好的 Zabbix Server 当作“外部 Server”，在当前集群里再部署一套 **Zabbix Proxy（active 模式）+ 每节点 Zabbix Agent2**，由 proxy 统一采集后回传 Server，并挂上 Zabbix 官方的 Kubernetes 监控模板，实现集群、节点、控制面组件的全量监控。

### 9.1 架构

```
┌─────────────────────────── zabbix namespace（外部 Server）──────────────────────────┐
│  zabbix-server:10051   zabbix-web:8080   zabbix-postgres:5432                         │
└───────────────────────────────────────▲───────────────────────────────────────────────┘
                                        │ active proxy 回连（10051）
┌─────────────────────────── zabbix-monitoring namespace ──────────────────────────────┐
│  zabbix-proxy（Deployment，1 副本）◄── 配置同步 / 数据上报 ──► zabbix-server            │
│     │                                                                                 │
│     ├─ 被动采集：连接每节点 Agent（nodeIP:10050）                                       │
│     └─ HTTP 采集：调用 K8s API / kube-state-metrics，跑官方模板的发现规则               │
│  zabbix-agent2（DaemonSet，hostNetwork，每节点一个）                                    │
└───────────────────────────────────────────────────────────────────────────────────────┘
```

复用集群已有的 `rancher-monitoring`（kube-state-metrics + prometheus-node-exporter），不再重复部署。

### 9.2 前置条件

- Helm 3（本环境 `~/.local/bin/helm`）；
- `zabbix` namespace 的 `zabbix-server` Service 可达（`zabbix-server.zabbix.svc:10051`）；
- 已安装 rancher-monitoring，存在 Service `rancher-monitoring-kube-state-metrics`（供官方 cluster-state 模板抓取）。

### 9.3 获取官方 chart

```bash
git clone --depth 1 https://git.zabbix.com/scm/zt/kubernetes-helm.git /tmp/kubernetes-helm
# chart 自带的 kube-state-metrics / prometheus-node-exporter 我们禁用，
# 若网络拉不到 prometheus-community 依赖仓库，可把 Chart.yaml 里的 dependencies 整块删掉（禁用时不影响）。
```

本仓库提供的覆盖文件：[deploy/zabbix-proxy/values.yaml](file:///home/zzh/source-code/k8s-monitoring/deploy/zabbix-proxy/values.yaml)。

关键覆盖项：

- 镜像 tag 固定为 `alpine-7.0-latest`，与 Server 7.0.32 一致，避免协议不兼容；
- `kubeStateMetrics.enabled=false`、`prometheusNodeExporter.enabled=false`，复用 rancher-monitoring；
- proxy 为 active 模式（`ZBX_PROXYMODE=0`），`ZBX_HOSTNAME=zabbix-proxy-k8s`，`ZBX_SERVER_HOST=zabbix-server.zabbix.svc`；
- `ZBX_STARTHTTPPOLLERS=8`：官方 K8s 模板大量使用 HTTP agent，默认 1 个轮询器太慢；
- agent `ZBX_SERVER_HOST=0.0.0.0/0` 允许 proxy 被动采集（**不要**同时设 `ZBX_PASSIVESERVERS`，否则 entrypoint 会把 `0.0.0.0/0` 重复写入 `Server=` 导致配置校验失败）；
- agent tolerations 增加 RKE2 控制面的 `CriticalAddonsOnly:NoExecute`，否则 agent 只会调度到 worker 节点；
- proxy `startupProbe.failureThreshold` 放宽，首次启动建 SQLite 库 + 拉全量配置较慢，避免被探针误杀。

部署（release 名取 `zabbix`，这样渲染出的 agent Service 名正好是 `zabbix-zabbix-helm-chart-agent`，匹配节点模板的默认宏 `{$KUBE.NODES.ENDPOINT.NAME}`）：

```bash
export PATH="$HOME/.local/bin:$PATH"
helm install zabbix /tmp/kubernetes-helm -n zabbix-monitoring --create-namespace \
  -f deploy/zabbix-proxy/values.yaml
```

### 9.4 Zabbix Server 侧配置（API）

> Zabbix 7.0 已内置官方 Kubernetes 模板，无需手动导入：`Kubernetes cluster state by HTTP`（templateid `10510`）、`Kubernetes nodes by HTTP`（templateid `10507`）。

1. **注册 active proxy**：

```json
POST /api_jsonrpc.php
proxy.create  {"name":"zabbix-proxy-k8s","operating_mode":0}   // 0=active, 1=passive
```

2. **建立 proxy 组并把 proxy 加入**（7.0 主机必须挂到 proxy 组，不能直接指定单个 proxy）：

```json
proxygroup.create  {"name":"k8s-proxies"}
proxy.update       {"proxyid":"1","name":"zabbix-proxy-k8s","proxy_groupid":"1",
                    "operating_mode":0,"local_address":"0.0.0.0","local_port":"10051",
                    "address":"127.0.0.1","port":"10051","tls_connect":1,"tls_accept":1}
```

> `proxy.update` 要求 `local_address` 非空，必须带上。

3. **取 SA token**（proxy 调 K8s API 用）：

```bash
kubectl get secret zabbix-zabbix-helm-chart -n zabbix-monitoring -o jsonpath={.data.token} | base64 -d
```

4. **创建两台主机并挂模板**（由 proxy 组监控，`monitored_by:2` + `proxy_groupid`）：

| 主机 | 模板 | 必需主机宏 |
| --- | --- | --- |
| `k8s-cluster` | Kubernetes cluster state by HTTP (`10510`) | `{$KUBE.API.TOKEN}`、`{$HTTP.TLS.VERIFY}=none`、`{$KUBE.STATE.ENDPOINT.NAME}=rancher-monitoring-kube-state-metrics` |
| `k8s-nodes` | Kubernetes nodes by HTTP (`10507`) | `{$KUBE.API.TOKEN}`、`{$HTTP.TLS.VERIFY}=none` |

要点：

- `monitored_by: 0`=server / `1`=proxy / `2`=proxy group，真正的 id 放在 `proxyid` 或 `proxy_groupid`；
- `{$HTTP.TLS.VERIFY}` 必须设为 `none`，集群 CA 是自签证书，默认 `full` 会让 proxy 的 HTTP agent TLS 校验失败；
- `{$KUBE.STATE.ENDPOINT.NAME}` 指向已有的 rancher KSM Service 名（模板跨所有 namespace 按 `metadata.name` 过滤 endpoint，所以只填名字即可）；
- 节点模板默认 `{$KUBE.NODES.ENDPOINT.NAME}=zabbix-zabbix-helm-chart-agent`，与 release 名 `zabbix` 渲染出的 Service 一致，无需改；
- 主机仍需一个接口（HTTP 模板也要），填一个 dummy Agent 接口 `127.0.0.1:10050` 即可。

可复现脚本（替换 `TOKEN` 与 API 地址）：

```python
import json, urllib.request
URL = "http://127.0.0.1:18080/api_jsonrpc.php"
def call(m, p, a=None):
    b = {"jsonrpc":"2.0","method":m,"params":p,"id":1}
    if a: b["auth"]=a
    r = urllib.request.urlopen(urllib.request.Request(URL,
        data=json.dumps(b).encode(), headers={"Content-Type":"application/json-rpc"}), timeout=60)
    d = json.loads(r.read())
    assert "result" in d, d
    return d["result"]
a = call("user.login", {"username":"Admin","password":"zabbix"})
gid = call("hostgroup.get", {"filter":{"name":["Kubernetes"]},"output":["groupid"]}, a)[0]["groupid"]
dummy = [{"type":1,"main":1,"useip":1,"ip":"127.0.0.1","dns":"","port":"10050"}]
call("host.create", {"host":"k8s-cluster","name":"K8s cluster state",
    "groups":[{"groupid":gid}],"monitored_by":2,"proxy_groupid":"1",
    "interfaces":dummy,"templates":[{"templateid":"10510"}], "macros":[
        {"macro":"{$KUBE.API.TOKEN}","value":TOKEN},
        {"macro":"{$HTTP.TLS.VERIFY}","value":"none"},
        {"macro":"{$KUBE.STATE.ENDPOINT.NAME}","value":"rancher-monitoring-kube-state-metrics"}]}, a)
call("host.create", {"host":"k8s-nodes","name":"K8s nodes",
    "groups":[{"groupid":gid}],"monitored_by":2,"proxy_groupid":"1",
    "interfaces":dummy,"templates":[{"templateid":"10507"}], "macros":[
        {"macro":"{$KUBE.API.TOKEN}","value":TOKEN},
        {"macro":"{$HTTP.TLS.VERIFY}","value":"none"}]}, a)
```

### 9.5 触发发现 / 查看效果

模板的发现规则是**依赖型**（`master_itemid` 非空，`delay=0` 是正常的，随主 item 定时拉取自动重跑）。刚接入时若想立刻看到结果，可手动触发：

```python
# task.create 的 itemid 在 7.0.32 里必须是单个整数（不能传数组）
for dr in call("discoveryrule.get", {"hostids":[hostid],"output":["itemid"]}, a):
    call("task.create", {"type":6,"request":{"itemid":int(dr["itemid"])}}, a)
```

查看：

- `Monitoring → Hosts`：应出现 `k8s-cluster`、`k8s-nodes`，以及自动发现的每节点主机（`m1/m2/m3/w1/w2`）和控制面组件主机（`API / Scheduler / Controller manager / Kubelet <节点>`）；
- `Monitoring → Latest data`：节点主机的 CPU / 内存 / 磁盘等来自 Agent，`k8s-cluster` / `k8s-nodes` 的集群态指标来自 KSM；
- `Monitoring → Problems`：集群真实告警（Pod 异常、组件健康、etcd 等）。

### 9.6 已验证结果（本环境实测）

- proxy online，从 server 同步全量配置约 4 MB，`version 70032` 与 server 一致；
- agent DaemonSet 调度到全部 5 个节点（含 3 个控制面），`hostNetwork` 监听 `nodeIP:10050`；
- 自动发现 5 个节点主机并按角色分组（control-plane/etcd/master=3、worker-pool=2），节点主机 116 个 item 仅 1 个 unsupported（容器内无包管理器，正常），可用内存等指标约 15s 刷新；
- 自动发现控制面组件主机：3×API server、3×Scheduler、3×Controller manager、5×Kubelet；
- Problems 已产生真实告警：`etcdHighCommitDurations [warning]`、`gitea / cattle-system / plc` 等命名空间的 Pod 异常，全部经 in-cluster proxy 上报。

### 9.7 常见坑

1. **agent 起不来**：`cannot parse the "Server" parameter: address "0.0.0.0/0" specified more than once` —— 别同时设 `ZBX_SERVER_HOST` 和 `ZBX_PASSIVESERVERS`；
2. **agent 只在 worker 节点**：RKE2 控制面带 `CriticalAddonsOnly:NoExecute` taint，values 里要补对应 toleration；
3. **proxy 首次启动 0/1 Ready 反复重启**：建库 + 拉配置超过默认启动探针窗口，调大 `startupProbe.failureThreshold`；
4. **HTTP 指标长时间不出数**：proxy 默认 `StartHTTPPollers=1`，设 `ZBX_STARTHTTPPOLLERS=8`；
5. **主机创建报 `proxyid value must be 0` / `object does not exist`**：7.0 用 proxy 组模型，按 9.4 建组并用 `monitored_by:2 + proxy_groupid`；
6. **K8s API / KSM 报 TLS 错误**：设 `{$HTTP.TLS.VERIFY}=none`；
7. **cluster-state 找不到指标**：确认 `{$KUBE.STATE.ENDPOINT.NAME}` 是实际存在的 KSM Service 名。

### 9.8 清理

```bash
helm uninstall zabbix -n zabbix-monitoring
kubectl delete ns zabbix-monitoring
# Zabbix 侧：删除 k8s-cluster / k8s-nodes 及自动发现的主机、proxy 组和 proxy
```
