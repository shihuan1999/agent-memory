# openJiuwen agent-memory — OpenHarmony riscv64 移植

> 本分支（`riscv64-ohos`）只存放 **OHOS riscv64 适配层**（说明 / launcher / 打包脚本）。
> 原始代码保留在本 fork 默认分支（跟随上游 openJiuwen-ai/agent-memory，Sync fork 即可更新）。

| 项 | 值 |
|---|---|
| 状态 | 已上机验证：memory_server :8000 长期运行，自研 hash embedding + SQLite 向量库（IR 评测超基线） |
| 上游基线 | develop @ 9bf45fa4154fc448cbe6a126958c50b9f26a1685 |
| brew 包 | `hbrew/riscv/openjiuwen-agent-memory`（bottle 在 riscv-bin Release） |
### 适配方式

rvcompute 网关无 embedding 模型，设备端把上游 `APIEmbedding` 替换为**本地哈希 embedding**
（离线、确定性），store_factory/.env/接口全部复用上游代码（零侵入，见
`ohos-deploy/` 与包内 `libexec/memory/start_memory_server.py` 头注释）。

### 冒烟 / API

```bash
agent-memory-server          # HTTP :8000（存取/检索长期记忆）
agent-memory-warmup          # 预热索引
```

应用侧接口契约与调用示例：
[openjiuwen-ohos-port/docs/API.md](https://github.com/shihuan1999/openjiuwen-ohos-port/blob/main/docs/API.md)

### ohos-deploy/（设备端 launcher，随瓶分发）

- `openjiuwen-agent-memory`
- `1.0.0`
- `['agent-memory-server', 'agent-memory-warmup']`

## 通用运行环境（OHOS riscv64 适配要点）

所有 openJiuwen 组件在设备上共享同一套运行约定（详见各 launcher）：

- `unset PYTHONHOME PYTHONPATH` 后重建 `PYTHONPATH`（hdc shell 会泄漏 /system 的 3.10 环境）；
- `LD_PRELOAD=libriscvflush.so`（riscv64 icache flush shim，openjiuwen 基础包提供）；
- `TMPDIR=/data/tmp`、`SSL_CERT_FILE=/etc/ssl/certs/cacert.pem`（musl 无系统 CA bundle）；
- 解释器优先级：**/data/python312（3.12.14）→ brew python 3.11**；3.12 优先时
  PYTHONPATH 需前置 `/data/python312/lib/python3.12/site-packages`
  （**ohos-312-patch**，否则注入的 3.11 编 pydantic_core 等二进制在 3.12 下无法加载）；
- pip 在线源：tuna（主）+ aliyun（备），musllinux_1_2_riscv64 轮可直装，
  用 `pipm` 安装（自动把 musl 后缀 .so 改成设备 gnu 后缀）；
- 运行日志写 cwd 下 `logs/`（launcher 已 cd $HOME/ojw-run 并建目录）。

## 设备实机验证（2026-09-03，K3 pico / OpenHarmony 6.1 / python 3.12.14）

- 18 个代码仓中 11 个已完成设备实机验证（含本仓）；
- 5 个源码 vendor 仓本轮验证：agent-runtime / agent-protocol / agent-tools / skillhub / agent-dx
  全部通过（边界记录见各仓章节）；
- 验证脚本与记录：openjiuwen-ohos-port 仓 + workspace/ojw-py312-musepaper2/。

## 安装与更新

```bash
# 设备上（已配置 Harmonybrew）
. /data/harmonybrew/hbrew-env.sh
brew install hbrew/riscv/<formula>     # 见下表
```

上游更新流程：Sync fork（默认分支保持上游原样）→ 在新基线上重估本分支说明与
launcher 兼容性 → 重建 bottle → 更新本 README 基线记录。
