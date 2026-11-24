# 实验 3 记录：第 15 组

评分时依据本文件和 `runs/` 中的原始数据核对。本文件中的每一个数值，都须能在某个 `runs/<名字>/` 的文件中找到：表格用 `scripts/table.sh` 生成或从 `summary.md` 中复制后粘贴，其他数值注明出处（例如“`runs/b2-cpu/summary.md` 缺口表第 1 行”）。不要只贴截图。将 `<填写>` 替换为本组的内容。

标有“预测”的小节须在对应的测量之前写好，写好后不要修改。预测错误不扣分，与实测不一致之处在分析中讨论。

## 1. 基本信息

| 项 | 内容 |
|---|---|
| 小组 | 第 15 组 |
| 仓库地址 | https://github.com/404MaximWang/ops-lab3-15 |
| 主机 | HuaweiCloud ECS with 2vCPU 2GB |
| CPU | General Purpose Processor；2 |
| 内存与 swap | 1775MB；无 |
| 操作系统、内核、systemd | Ubuntu 24.04.5 LTS；6.8.0-147-generic；systemd 255 (255.4-1ubuntu8.18) |
| cgroup | cgroup2fs；cpuset cpu io memory hugetlb pids rdma misc |
| Go、Prometheus、node_exporter | 1.26.9；3.13.4；1.12.1 |
| 分核 | 实例：1；loadgen、lbd、scaler、Prometheus、node_exporter：0 |

## 2. 任务 A：配额，节流与 OOM

### 2.1 A1 配额算术

预测（测量之前写）：配额 0.5 核、周期 100 ms 时，1 个和 2 个线程跑满，每个线程每个周期运行多久、被停多久？本机上 enrolld 的 GOMAXPROCS 会是几，为什么？

各线程各周期运行50ms/25ms，被停止50ms/75ms。cgroups的限额最小。是0.5；按照保底规则，本机GOMAXPROCS是2。

两次 `-burn` 的输出：

```text
cgroup：/sys/fs/cgroup/system.slice/run-r968c22baa560431a9830b0195be99cfb.scope
cpu.max：配额 0.50 核，周期 100ms（每个周期最多 50 ms CPU 时间）
GOMAXPROCS=2，忙线程 1 个，计数 10.0 秒
实际用核 0.50；周期 100 个，被节流 100 个（100%）；被停共 5.08 秒（按 CPU 累计）
每个线程每个周期被停：实测 50.8 ms，算术 50.0 ms
cgroup：/sys/fs/cgroup/system.slice/run-rf9c7b2b141214ecdbf0727a9d1ea0db5.scope
cpu.max：配额 0.50 核，周期 100ms（每个周期最多 50 ms CPU 时间）
GOMAXPROCS=2，忙线程 2 个，计数 10.0 秒
实际用核 0.50；周期 100 个，被节流 100 个（100%）；被停共 14.91 秒（按 CPU 累计）
每个线程每个周期被停：实测 74.5 ms，算术 75.0 ms
```

| 线程数 | 周期数 | 被节流的周期 | 实际用核 | 每线程每周期被停（实测） | 算术 |
|---|---|---|---|---|---|
| 1 | 100 | 100 | 0.50 | 50.8 | 50.0 |
| 2 | 100 | 100 | 0.50 | 74.5 | 75.0 |

### 2.2 A2 节流与 p99

本组写的节流比例查询（与 `deploy/queries-red.txt` 一致）：

```text
throttle_ratio      rate(cgroup_cpu_throttled_periods_total{job="enrolld"}[${window}]) / rate(cgroup_cpu_periods_total{job="enrolld"}[${window}])
```

配额 100% 时测得的每请求 CPU：4.17692 ms（`runs/a2-base/red.csv` 的 `cpu_ms_per_req`）；据此选定的到达率：95 请求/秒，平均约 0.4 核。

预测（配额改为 50% 之前写）：平均用量低于配额，p99 会不会变？节流比例大约多少？会变大；20%。

各次实验的数据（`scripts/table.sh runs/<名字> ok_rps p50_ms p99_ms server_p99_ms gomaxprocs cg_limit_cores cg_cpu_cores throttle_ratio throttled_s_per_s`，每次一张表）：

#### 100%（runs/a2-q100）

| step | target_rps | ok_rps | p50_ms | p99_ms | server_p99_ms | gomaxprocs | cg_limit_cores | cg_cpu_cores | throttle_ratio | throttled_s_per_s |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 95 | 98.0 | 6.88 | 10.96 | 9.95354 | 2 | 1 | 0.412141 | 0 | 0 |
| 2 | 95 | 91.6 | 6.90 | 10.55 | 9.95392 | 2 | 1 | 0.373809 | 0 | 0 |
| all | - | 94.8 | 6.89 | 10.74 | - | - | - | - | - | - |

#### 50%（runs/a2-q50）

| step | target_rps | ok_rps | p50_ms | p99_ms | server_p99_ms | gomaxprocs | cg_limit_cores | cg_cpu_cores | throttle_ratio | throttled_s_per_s |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 95 | 98.0 | 6.94 | 44.86 | 40.6344 | 2 | 0.5 | 0.409849 | 0.335714 | 0.0879525 |
| 2 | 95 | 91.6 | 6.92 | 36.44 | 28.1937 | 2 | 0.5 | 0.372077 | 0.179856 | 0.0371135 |
| all | - | 94.8 | 6.94 | 43.98 | - | - | - | - | - | - |

#### 改法1（runs/a2-mod1，CPUQuota=60%）

| step | target_rps | ok_rps | p50_ms | p99_ms | server_p99_ms | gomaxprocs | cg_limit_cores | cg_cpu_cores | throttle_ratio | throttled_s_per_s |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 95 | 98.0 | 6.89 | 17.97 | 17.0529 | 2 | 0.6 | 0.411769 | 0.0857143 | 0.0158583 |
| 2 | 95 | 91.6 | 6.89 | 11.42 | 9.97365 | 2 | 0.6 | 0.372952 | 0.0214286 | 0.0046515 |
| all | - | 94.8 | 6.89 | 14.66 | - | - | - | - | - | - |

#### 改法2（runs/a2-mod2）

| step | target_rps | ok_rps | p50_ms | p99_ms | server_p99_ms | gomaxprocs | cg_limit_cores | cg_cpu_cores | throttle_ratio | throttled_s_per_s |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 95 | 98.0 | 8.40 | 45.30 | 33.5273 | 1 | 0.5 | 0.404317 | 0.235714 | 0.0325981 |
| 2 | 95 | 91.6 | 8.39 | 44.01 | 30.1308 | 1 | 0.5 | 0.364821 | 0.114286 | 0.0228148 |
| all | - | 94.8 | 8.40 | 44.94 | - | - | - | - | - | - |

### 2.3 A3 OOM

预测（测量之前写）：`/api/report?mb=8&ms=500` 在到达率 10、20、30 时平均和最多各有几个在途请求？RSS 大约多少？在 `MemoryMax=200M` 下会在哪一个阶段被杀？

到达率 10、20、30 时平均在途请求数约为 5、10、15 个；按 Poisson 到达波动估算，突发峰值最多分别约为 10、17、23 个。负载为平均值时，只计算活跃对象，各组占用 53、93、133 MB；但因 Go 默认 GOGC=100 在堆增长到上次存活数据的约 2 倍时才回收，峰值期间总内存大约可达 93 MB、173 MB、253 MB。预计在第 2 阶段后期或第 3 阶段时，系统触发 OOM Kill。

#### 第 1 组：MemoryMax=200M（runs/a3-oom）

| step | target_rps | ok_rps | error_rate | p99_ms | inflight_avg | report_inflight_max | rss_max_mb | heap_max_mb | mem_current_max_mb | mem_limit_mb |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 10 | 9.1 | 0.0000 | 505.27 | 4.73333 | 8 | 137.836 | 121.395 | 132.508 | 200 |
| 2 | 20 | 1.8 | 0.9091 | 504.73 | - | - | - | - | - | - |
| all | - | 5.5 | 0.6228 | 505.21 | - | - | - | - | - | - |

#### 第 2 组：改法一 MemoryMax=313M（runs/a3-mod1）

| step | target_rps | ok_rps | error_rate | p99_ms | inflight_avg | report_inflight_max | rss_max_mb | heap_max_mb | mem_current_max_mb | mem_limit_mb |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 10 | 9.1 | 0.0000 | 505.29 | 4.26667 | 6 | 134.324 | 89.3511 | 129.277 | 313 |
| 2 | 20 | 19.8 | 0.0000 | 504.90 | 9.8 | 19 | 298.445 | 185.497 | 293.461 | 313 |
| 3 | 30 | 0.3 | 0.9888 | 506.49 | - | - | - | - | - | - |
| all | - | 9.8 | 0.5129 | 505.04 | - | - | - | - | - | - |

#### 第 3 组：改法二 MemoryMax=200M + -report-max 11（runs/a3-mod2）

| step | target_rps | ok_rps | error_rate | p99_ms | inflight_avg | report_inflight_max | rss_max_mb | heap_max_mb | mem_current_max_mb | mem_limit_mb |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 10 | 9.1 | 0.0000 | 504.98 | 4.26667 | 7 | 137.797 | 105.355 | 132.727 | 200 |
| 2 | 20 | 16.4 | 0.1742 | 504.94 | 8.73333 | 11 | 186.125 | 153.421 | 181.125 | 200 |
| 3 | 30 | 19.5 | 0.3740 | 504.84 | 9.8 | 11 | 186.246 | 153.433 | 181.277 | 200 |
| all | - | 15.0 | 0.2515 | 504.94 | - | - | - | - | - | - |

被杀的证据（`systemctl show -p ActiveState,Result,ExecMainStatus lab3-a` 和 `journalctl -u lab3-a` 的相关行，`dmesg` 或 `/var/log/kern.log` 中的 oom-kill 行）：

```text
# systemctl show -p ActiveState,Result,ExecMainStatus lab3-a
Result=oom-kill
ExecMainStatus=9
ActiveState=failed

# journalctl -u lab3-a --no-pager | tail -n 5
Nov 23 23:28:43 ecs-3815 systemd[1]: Started lab3-a.service - /home/admin/ops-lab3-15/bin/enrolld -addr 127.0.0.1:8081 -data /home/admin/ops-lab3-15/data/a.
Nov 23 23:28:43 ecs-3815 enrolld[3083421]: 2026/11/23 23:28:43 enrolld v1 监听 127.0.0.1:8081；GOMAXPROCS=2；cgroup /sys/fs/cgroup/system.slice/lab3-a.service；预热 0s；读：work=200000、pool=32、db-ms=2；写：sync=each
Nov 23 23:29:07 ecs-3815 systemd[1]: lab3-a.service: A process of this unit has been killed by the OOM killer.
Nov 23 23:29:07 ecs-3815 systemd[1]: lab3-a.service: Main process exited, code=killed, status=9/KILL
Nov 23 23:29:07 ecs-3815 systemd[1]: lab3-a.service: Failed with result 'oom-kill'.

# dmesg | grep -i oom-kill
[2286273.245531] enrolld invoked oom-killer: gfp_mask=0xcc0(GFP_KERNEL), order=0, oom_score_adj=0
[2286273.245662] memory: usage 204800kB, limit 204800kB, failcnt 78
[2286273.245665] Memory cgroup stats for /system.slice/lab3-a.service:
[2286273.246089] [3083421]     0 3083421   365787    52497    50993     1504         0   512000        0             0 enrolld
[2286273.246093] oom-kill:constraint=CONSTRAINT_MEMCG,nodemask=(null),cpuset=/,mems_allowed=0,oom_memcg=/system.slice/lab3-a.service,task_memcg=/system.slice/lab3-a.service,task=enrolld,pid=3083421,uid=0
[2286273.246120] Memory cgroup out of memory: Killed process 3083421 (enrolld) total-vm:1463148kB, anon-rss:203972kB, file-rss:6016kB, shmem-rss:0kB, UID:0 pgtables:500kB oom_score_adj:0
```

两种改法：按最坏情况定的 limit 是 313MB（算式：(17 * 8 * 2 + 13) * 1.1）；限制并发时 `-report-max` 取 11（依据：11 * 8 * 2 + 13 = 189 MB < 200 MB）。

## 3. 任务 B：伸缩控制环

### 3.1 B0 单个实例的能力

#### runs/b0-one

| step | target_rps | ok_rps | error_rate | p99_ms |
|---|---|---|---|---|
| 1 | 20 | 18.6 | 0.0000 | 7.11 |
| 2 | 40 | 41.5 | 0.0000 | 7.70 |
| 3 | 60 | 63.3 | 0.0000 | 8.48 |
| 4 | 80 | 76.3 | 0.0000 | 14.16 |
| 5 | 100 | 99.8 | 0.0000 | 60.47 |
| all | - | 59.9 | 0.0000 | 38.24 |

#### runs/b0-one-hi

| step | target_rps | ok_rps | error_rate | p99_ms |
|---|---|---|---|---|
| 1 | 100 | 103.5 | 0.0000 | 62.31 |
| 2 | 120 | 114.4 | 0.0000 | 116.25 |
| 3 | 140 | 110.5 | 0.2376 | 1232.93 |
| 4 | 160 | 110.8 | 0.2912 | 1845.54 |
| all | - | 109.8 | 0.1540 | 1506.41 |

单个实例（CPU 配额 25%）的上限约 114.4 请求/秒，按 0.7 的容量线可承担 80 请求/秒；依据：`runs/b0-one-hi/steps.csv` 第 2 阶段（目标 120 req/s）为开始出错前的最大 `ok_rps = 114.4`（`error_rate = 0`，`p99_ms = 116.25 ms`；第 3 阶段目标 140 req/s 即出现 23.76% 错误），`114.4 × 0.7 ≈ 80`。

本组的到达曲线：平时 32、峰值 320 请求/秒（`scripts/plan.sh` 的输出：`32@120s,32~320@30s,320@300s,320~32@60s,32@120s`）。

预测（B1～B3 之前写）：峰值需要几个实例？反应式从爬升开始到新实例就绪大约要多久，缺口大约多长？定时预热最晚何时开始？4个；QPS信号下就绪需要110-130s而CPU信号下需要215-245s；按照硬上限计算，缺口为90-110s；定时预热最晚49s开始。

### 3.2 三种方案的对照

| 方案 | `runs/` 目录 | 参数 | 缺口（秒） | 错误数 | 峰值阶段的 p99（ms） | 实例·分钟 |
|---|---|---|---|---|---|---|
| B1 常驻 | `runs/b1-fixed` | `-mode fixed -replicas 4` | 0 | 0 | 26.99 | 42.0 |
| B2 反应式（信号一） | `runs/b2-cpu` | `-mode reactive -min 2 -max 4 -signal "$CPU"` | 110 | 16653 | 1732.22 | 36.6 |
| B2 反应式（信号二） | `runs/b2-qps` | `-mode reactive -min 2 -max 4 -target 80 -signal "$QPS"` | 100 | 15662 | 1639.01 | 36.8 |
| B3 定时加反应式 | `runs/b3-sched` | `-mode reactive -min 2 -max 4 -schedule "+34s=4,+480s=2" -signal "$CPU"` | 0 | 0 | 27.43 | 40.8 |

### 3.3 控制环各段的实测

以 B2（信号一）为例，从 `summary.md` 的“伸缩决策”和“实例”两张表读出：

| 段 | 起止 | 实测（秒） | L-11 第 10 页的设定 |
|---|---|---|---|
| 指标延迟与等待决策（爬升开始 → 第一次扩容决策） | 120.0 → 156.0 | 36.0 | 指标约 20 s + 决策 ≤ 15 s |
| 拉镜像（创建 → 启动） | 155.6 → 176.6 | 21.0 | 约 20 s |
| 启动、预热与健康检查（启动 → 就绪） | 176.6 → 239.6 | 63.0 | 约 60 s |
| 合计（爬升开始 → 新实例就绪） | 120.0 → 239.6 | 119.6 | 约 2 min |

B2 缺口中的错误：客户端超时 6432 个，lbd 返回的状态码 `502:1654 503:8567`（`steps.csv` 的 `timeout`、`codes`）；这段时间 `lbd-events.csv` 中有 27 次 `down`。

## 4. 任务 C：故障注入

| 注入 | `runs/` 目录 | lbd 的参数 | 判定用时（秒） | 恢复用时（秒） | 注入到判定之间的错误 | 全程错误 | 其他 |
|---|---|---|---|---|---|---|---|
| C1 停止（kill） | `runs/c1-stop` | 默认（1s/500ms, fall=3, rise=2） | 0.3 | 83.3 | 16 | 161 | 判定是 `removed` 还是 `down`：`removed`；补建的实例何时就绪：144.6 s |
| C1 卡住（pause 一台 20 秒） | 同上 | 同上 | 2.8 | 22.3 | 145 | 同上 | 判定是 `removed` 还是 `down`：`down` |
| C2 暂停 2 秒（灵敏） | `runs/c2-sensitive` | `-check-interval 500ms -check-timeout 200ms -fall 1` | 0.5 | 2.8 | 87 | 379 | 缺口 3 s（`p99=13.89 ms`，全站短暂 503，错误放大 12 倍） |
| C2 暂停 2 秒（宽松） | `runs/c2-default` | 默认（1s/500ms, fall=3, rise=2） | 2.6 | 7.1 | 30 | 30 | 缺口 6 s（慢请求 709 个，`p99=1463.27 ms`） |
| C3 变慢（不摘除） | `runs/c3-slow` | 默认（不开启 `-outlier`） | 未判定（-） | - | 0 | 0 | 全站 p99：19.54 ms（单秒最高 167.16 ms） |
| C3 变慢（离群摘除） | `runs/c3-outlier` | `-outlier`（默认 window=10s, factor=3, min=50ms, eject=30s） | 24.8 | 54.8 | 0 | 0 | 全站 p99：14.88 ms（共摘除 `ejected` 2 次） |

预测（C1 之前写）：按 lbd 的参数，一个实例停止或卡住以后多久被判定？其间大约有多少请求出错？停止时，scaler会在0-1s内判定；卡住时，lbd会在2.5-3.5s内判定；按三台机器平分请求计算，前者0-54个请求出错，后者133-187个请求出错。

C1 中停止与卡住的判定用时各是多少，分别是谁发现的？与“间隔 × 失败阈值”相比如何？停止（`kill 8102`）判定用时为 `0.3 s`（`removed`），由 `scaler`（每秒巡检 systemd 单元状态）先于健康检查发现并移除，远快于健康检查的“间隔 × 失败阈值”（`1s × 3 = 3s`）；卡住（`pause 8103 20`）判定用时为 `2.8 s`（`down`），因进程仍存活且为 `active`，只能由 `lbd` 的主动健康检查连续超时 3 次发现，介于 `2.5 ~ 3.5 s` 之间，略短于直接相乘的 `3 s`（首次探测距注入 `0~1 s` + 第 3 次检查 `0.5 s` 超时）。

C3 中把配额调到了多少，依据的用量 U 是多少：调到了 `13%`；依据 `runs/c1-stop` 注入前（前 60s）Prometheus 测得三台实例的 `rate(cgroup_cpu_usage_seconds_total{job="instances"}[30s])` 分别为 `0.1103`、`0.1112`、`0.1105` 核（均值 `U ≈ 0.111` 核），按 `U ÷ 0.85 ≈ 0.130` 取整为 `13%`。

C3 中健康检查为什么全部通过？用 `runs/<名字>/lbd-metrics-end.txt` 或 Prometheus 中按实例的数据，找出是哪一台变慢，并说明依据：`/readyz` 只读内存中的就绪标志位，几乎不消耗 CPU，即便 `8102` 配额被压到 `13%` 发生业务计算节流，健康检查仍可在 `500 ms` 超时内返回 `200 OK`，因此全部通过。从 `runs/c3-slow/lbd-metrics-end.txt` 看各实例总耗时/请求数均值：`8101` 为 `69.900s/12875 ≈ 5.43 ms`、`8103` 为 `70.039s/12876 ≈ 5.44 ms`，而 `8102` 升高到 `84.357s/12875 ≈ 6.55 ms`；在 Prometheus 按 `backend` 查 `histogram_quantile(0.99, sum by (backend, le) (rate(lb_request_duration_seconds_bucket[60s])))`（如 `t=1795525209`），`8101` 和 `8103` 的 p99 为 `17.6 ms` 和 `17.5 ms`，而 `8102` 飙升至 `50.1 ms`（且在 `runs/c3-outlier/lbd-events.csv` 中 `8102` 两次因 `p99=58.6ms / 54.1ms` 被 `ejected`），可确定是 `127.0.0.1:8102` 变慢。

## 5. 任务 D：发布与自愈

### 5.1 D1 滚动发布

预测：按 L-05 第 24 页的公式，3 个实例逐个替换要多久？直接替换时大约多少请求出错？约260s；直接替换约76s，故有12160个请求出错。

| 做法 | `runs/` 目录 | 发布总时长（秒） | 错误数 | 出错的时段 | 说明 |
|---|---|---|---|---|---|
| 直接替换（naive） | `runs/d1-naive` | 72.6（31.0→103.6） | 8943 | 第 29～33 s、35～99 s（共 68 s 缺口） | `SIGKILL` 后原端口重启预热 `62.6s` 且仅间隔 `5s` 替换下一台，导致三台同时陷入预热 `503` 盲区 |
| 排空加就绪（graceful） | `runs/d1-graceful` | 258.0（30.0→288.0） | 0 | 无 | 先起新版本并等就绪（约 `84s`）后再排空停旧，全站无损（代价：发布期间多占 1 台临时实例，总计 25.3 实例·分钟） |

### 5.2 D2 自愈

预测（D2 之前写）：3 台实例轮询均分 `160 req/s`（每台约 `53.3 req/s`），每请求永久泄漏 `16 KB`，每台净泄漏速度约 `0.83 MB/s`；考虑基础 RSS 约 `13 MB` 及 Go GC（`GOGC=100`）会在堆长到存活数据约 2 倍时才回收，存活泄漏数据涨到约 `25~30 MB` 时总内存就可能撞穿 `64 MB` 上限，预计约30s触发一次OOM。由于 3 台实例几乎同时启动且流量均分，它们的 OOM 与重启几乎同步共振；在三台同时重启并经历 `10s` 预热的窗口内，全站没有可用实例，会产生大量 `503` 错误并形成周期性缺口。

| 项 | 数值与出处 |
|---|---|
| 参数（`-restart`、`-args`、`-memory`） | `-mode fixed -replicas 3 -cpus 1 -restart on-failure -memory 64M -args "-warmup 10s -work 1000000 -leak-kb 16"`（`runs/d2-heal/meta.txt`） |
| 每个实例的泄漏速度、预计多久 OOM 一次 | 约 `53.3 req/s × 16 KB/req ≈ 853 KB/s（0.83 MB/s）`；实测每运行约 `64~66 s` OOM 一次（含重启+10s 预热约 14s，完整周期 ~`80 s`） |
| 实测的重启次数（`systemctl show -p NRestarts lab3-inst-…`、`summary.md` 的“实例”表） | `systemctl show -p NRestarts` 三台均为 `6`；`runs/d2-heal/summary.md`“实例”表三台各重启 `6` 次（合计 18 次） |
| 各实例重启的时刻（`instances.csv` 的 restarted 行）与缺口的时刻 | 重启时刻：`70.0s`、`149.0s`、`229.0~229.1s`、`309.0~309.1s`、`389.0~391.0s`、`468.0~471.0s`；缺口时刻：`66~81s`、`146~160s`、`226~240s`、`306~320s`、`386~400s`、`464~479s`（共 86 s） |
| 全程的成功率、错误数 | 成功率 `83.28%`（`64097 / 76966`），错误数 `12869`（`16.72%`，`runs/d2-heal/steps.csv`） |

本组写的告警查询，以及它在这次实验中的值：

```text
sum(changes(process_start_time_seconds{job="instances"}[10m]))
# 按单实例：changes(process_start_time_seconds{job="instances"}[10m])
# 实测值（d2-heal 结束时刻查询）：单实例均为 6，合计 18（结束 4 分钟后实时查询仍为单实例 4、合计 12）
```
预测错误的原因：
- 套用了A3的场景；实际上此处的对象都是活跃的永久泄漏对象，没有临时垃圾可清理，不存在放大效应。
- cgroup计入的基础内存仅有2.64MB而非13MB。

## 6. 分析问题

每题先列出所用数据（引用上面的表或 `runs/` 中的文件），再写结论，并说明结论的成立条件。每题 300～600 字。
**问题 1：配额。** A1 实测的每线程每周期被停时间与 L-09 第 12 页的算术相差多少，差异来自何处？A2 中平均用量低于配额时，p99 为什么变大？两种改法各改变了什么，各有什么代价？A3 中 RSS 的推算与实测是否一致？Go 的垃圾回收对“基线 + 在途数 × 每请求内存”这个估计有什么影响？`MemoryMax` 为什么要和 `MemorySwapMax=0` 一起设？按最坏情况定 limit 与限制并发，各适合什么条件？

- 所用数据：第 2.1 节 A1 表；第 2.2 节 `runs/a2-q100`、`runs/a2-q50`、`runs/a2-mod1`、`runs/a2-mod2` 表（`steps.csv`、`red.csv`）；第 2.3 节 `runs/a3-oom`（含 `dmesg` 与 `journalctl`）、`runs/a3-mod1`、`runs/a3-mod2` 表（`steps.csv`、`red.csv`）。
- 算术差异：+0.8ms/-0.5ms。来自启动与统计边界未对齐完整 100 ms 周期、Go 运行时调度和内核CFS带宽统计抖动。
- p99何故变大：请求的到达率不是平稳的；在大量请求突发时，时间配额被快速用尽，剩余的请求被挂起。
- 两种改法：改法一放宽至60%，使p99和throttle_ratio降低；代价是单实例占用更多资源，同机可同时运行的实例数量减少，日常CPU冗余增大。改法二阻止双核争抢配额，throttle_ratio数值减半，但p50略微提升，p99变化不大，常规延迟上升，且丧失多核并行能力。
- RSS的推算和实测：差距较大。我按照平均在途数推算；但实测值对应突发在途数。GC使得总内存表达式的后一项变为换来的两倍。
- 为什么要一起设置：后者禁止内存页换出，确保超限后快速失败。
- 各适合什么条件：最坏情况适合要求高成功率且能承受平时内存闲置的场景；限制并发适合内存上限固定，允许请求量大时服务降级的场景。

**问题 2：控制环。** 各段延迟的实测值是多少，哪一段最长，能压缩多少？三种方案的缺口与实例·分钟各是多少？两种信号的差别在哪里，为什么？B2 的缺口中除了超时，还有 lbd 返回的 503：过载为什么会让健康检查也失败，这使缺口变得更严重还是更轻？你们会为“开放日”选哪个方案，结论在什么条件下成立？（可选：思考题 3 第四幕中你们的判断，与这里测到的现象是否一致？）

- 所用数据：第 3.2 节三种方案对照表、第 3.3 节控制环各段实测表，以及 `runs/b1-fixed`、`runs/b2-cpu`、`runs/b2-qps`、`runs/b3-sched` 下的 `summary.md`、`steps.csv`、`lbd-events.csv`。
- 延迟：指标与决策36s，拉取镜像21s，启动预热和健康检查63s。启动预热和健康检查最长。分别如何优化：缩短抓取和评估窗口；精简镜像；优化应用冷启动。能压缩的量不多。
- 缺口：B1 0s，42.0实例分钟；B2（CPU） 110s，36.6实例分钟；B2（QPS）100s，36.8实例分钟；B3 0s，40.8实例分钟。
- 两种信号的差别：QPS比CPU缺口少10s，出错请求数量少991个。CPU信号的上限是1，因此反应滞后，一次最多扩容一台，而QPS信号下2台设备被一次性扩容。
- 过载：readyz接口连续3次超时导致失败；这使缺口变得更严重。
- 选B3，在0缺口的同时节省资源。结论在我们已知特定时间段会有大流量时成立。

**问题 3：健康检查。** 停止、卡住、暂停、变慢四次注入中，各是谁、用多久发现的，健康检查看到了什么？检出快与误判少之间怎样取舍？离群摘除解决了什么问题，又带来什么风险？为什么要限制同时摘除的比例？

- 所用数据：第 4 节任务 C 汇总表，以及 `runs/c1-stop`、`runs/c2-default`、`runs/c2-sensitive`、`runs/c3-slow`、`runs/c3-outlier` 下的 `summary.md`、`steps.csv`、`lbd-events.csv`、`lbd-metrics-end.txt`、`marks.csv`。
- 停止是scaler用0.3s发现的；卡住是ldb用2.8s发现的；暂停是ldb用2.6/0.5s发现的；变慢是outliert用24.8s发现的。健康检查：连接被拒绝；响应超时；响应超时；正常。
- 取舍：首先检测短时抖动的时间，并将窗口置于此时间之上；同时使用多个信息源进行检出，不依赖灵敏的检测参数。
- 离群摘除：用业务请求延迟发现假健康，降低p99；带来了抖动造成的误摘除，且共性抖动下它会将大量后端踢出，造成服务不可用。

**问题 4：发布与自愈。** 直接替换时，失败的请求来自哪几步？排空加就绪消除了哪几类失败，代价是什么？D2 中三个实例的重启时刻有什么关系，为什么？这对“有冗余，一台重启不影响服务”的假设意味着什么？自愈解决了什么、掩盖了什么？应当在哪一层、用什么指标告警？

- 所用数据：第 5.1 节 D1 滚动发布表、第 5.2 节 D2 自愈表，以及 `runs/d1-naive`、`runs/d1-graceful`、`runs/d2-heal` 下的 `summary.md`、`steps.csv`、`timeline.csv`、`instances.csv`、`lbd-events.csv`。
- 在旧实例中请求未被排空时强杀；在新实例尚未被预热时启用新实例；三台新实例被lbd同时判定为失败。
- 解决了以上三个失败，其中排空解决了失败1,就绪解决了失败2、3；代价是发布时长增加，且需要额外的实例。
- 几乎同时重启。这使因为三台同时启动，均分流量，在几乎相同的内存泄漏速率下几乎同时OOM。冗余支队独立随机故障成立；面对共因故障和相同的负载，各个副本会同时失效。
- 自愈能做到自动恢复，免去人工恢复，但掩盖了代码本身的问题以及全站周期性不可用的问题。应当在内存资源一侧和进程一侧进行监控和告警，并在内存占用逼近界限时预警。

## 7. 选做记录

| 选做 | 完成情况与证据 |
|---|---|
| O1 GOMEMLIMIT | 已做（`runs/o1-gomemlimit`），见表下说明 |
| O2 两层伸缩 | <未做 / 参数、`runs/` 目录、结论> |
| O3 扩错层 | <未做 / 预测、`runs/` 目录、结论> |
| O4 Kubernetes 的 HPA | <未做 / 环境、配置文件、数据与结论> |
| O5 换负载均衡器 | <未做 / 配置文件、`runs/` 目录、结论> |
| O6 两台主机 | <未做 / 拓扑、`runs/` 目录、结论> |
| O7 迁移到小组项目 | <未做 / 仓库链接、做了什么、结论> |

#### O1 GOMEMLIMIT（`runs/o1-gomemlimit`）

- 预测：开启 `GOMEMLIMIT=170MiB` 后，第 2 阶段可通过高频 GC 压低垃圾堆积，但活跃对象继续升高后仍会 OOM 被杀，且 GC 次数与 CPU 代价明显增加。
- 实测（`scripts/table.sh runs/o1-gomemlimit ok_rps error_rate p99_ms inflight_avg report_inflight_max rss_max_mb heap_max_mb mem_current_max_mb mem_limit_mb gc_per_s cpu_ms_per_req`）：

| step | target_rps | ok_rps | error_rate | p99_ms | inflight_avg | report_inflight_max | rss_max_mb | heap_max_mb | mem_current_max_mb | mem_limit_mb | gc_per_s | cpu_ms_per_req |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 10 | 9.1 | 0.0000 | 505.40 | 4.33333 | 7 | 153.965 | 105.349 | 148.898 | 200 | 1.64286 | 2.00379 |
| 2 | 20 | 19.6 | 0.0076 | 504.99 | 9.73333 | 19 | 172.426 | 153.489 | 167.43 | 200 | 3.57143 | 1.4958 |
| 3 | 30 | 0.0 | 1.0000 | - | - | - | - | - | - | - | - | - |
| all | - | 9.6 | 0.5212 | 505.13 | - | - | - | - | - | - | - | - |

- 结论：
  1. **是否被杀及阶段**：仍被内核 `oom-kill`（`ExecMainStatus=9`），但比无限制的 `runs/a3-oom`（第 2 阶段初即被杀，`ok_rps=1.8`）死得更晚，几乎完整扛过第 2 阶段（`ok_rps=19.6`），在第 2 阶段末尾切换第 3 阶段时撞穿 `200M` 被杀。同等峰值并发 `report_inflight_max=19` 下，内存从 `runs/a3-mod1` 的 `rss_max_mb=298.445 MB` 被压到 `172.426 MB`（`mem_current_max_mb=167.43 MB`）。
  2. **GC 次数与 CPU 代价**：第 1 阶段与 `runs/a3-oom` 相同（`gc_per_s=1.64286`）；第 2 阶段逼近 `170MiB` 后 `gc_per_s` 升至 `3.57143`（约为无 `GOMEMLIMIT` 的 `runs/a3-mod1` 第 2 阶段 `1.85714` 的 1.9 倍），GC 用掉的 CPU 也随之翻倍。
  3. **分工**：`GOMEMLIMIT` 是 Go 运行时的软限制，负责尽量通过加频 GC 压低堆内存，但不杀进程也无法清除活跃对象及非 Go 内存；`MemoryMax` 是内核 cgroup 的硬限制，不负责回收垃圾。

## 8. 成员分工

每位成员一行，只填写 GitHub 用户名，无需填写姓名；行数应与小组人数一致。

| GitHub 用户名 | 负责的部分 |
|---|---|
| 404MaximWang | ABCDE, O1 |
| Fruitydrops |  |
| tywu26 |  |
| HAOYANERIC |  |
| BearZWasTaken |  |
