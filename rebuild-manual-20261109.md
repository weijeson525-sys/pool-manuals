# 说明

- 来源：号池体系代码级暴力拆解报告（2026-11-09，老板亲令），舰队仓 results/W03-TO-W02-pool-system-dissection-20261109.md（commit 97d5141）。
- 本文件为该报告对应章节的单行本，便于分发；内容与报告一致，未改动。
- 凭据值永不进文件；复现前须先修 setup_gateway.py 的 base_url（去掉 /v1），见搭建手册第 4 步。

## 三、从零复现搭建手册（隔离环境，一步步搭通）

目标：在一台干净机器上复现出与本机隔离场等价的环境（yukkcat 38083 + new-api 38084），只用实验号，不碰生产。前置：Python 3.13 + uv、Go 编译好的 new-api 二进制或官方 release、实验号凭据（0600 文件保管，永不进仓）。

**第 1 步：取源码（二进制与源码对得上才用）**
- yukkcat：`git clone https://github.com/yukkcat/chatgpt2api`，checkout v3.2.6（本机基准 commit 385bed19）；或用本机 tarball `~/workspace/tools/yukkcat-pilot/chatgpt2api-v3.2.6.tar.gz`。核对：`api/ai.py` 须有 `/v1/images/generations`（137 行），`services/image_failure.py` 须有 `switch_account = (status_code != 400)`（37-42 行）。
- new-api：官方 release v1.0.0-rc.48 二进制，核对 SHA-256 前缀 `27fc243a`；源码对照 clone `https://github.com/QuantumNous/new-api`，checkout tag v1.0.0-rc.48（commit 7061447）。

**第 2 步：起 yukkcat（端口 38083，仅回环）**
- 目录内建 venv（`uv sync` 或等价），`.env` 里放 `CHATGPT2API_AUTH_KEY`（0600）；`DATA_DIR=data`（SQLite `data/chatgpt2api.db`）。
- 启动：`setsid .venv/bin/uvicorn main:app --host 127.0.0.1 --port 38083`，30 秒内轮询 `GET /v1/models`（带 key）到 200 为止。现成脚本：`~/workspace/tools/yukkcat-pilot/start-yukkcat.sh`（已含 .env 加载与健康检查；注意 `uv run` 不自动加载 .env，须显式 source）。
- 验通：`GET /v1/models` 200（9 个模型）→ `POST /v1/chat/completions`（gpt-5.6）200。

**第 3 步：接实验号（每号先真实出图验活）**
- 用 yukkcat 的账号导入（API/UI）把实验号接进 `accounts` 表；一次只接 2 个，测通了再扩。
- 每号隔离验活：其余号批量置“禁用”，只留目标号“正常”，跑 10 格矩阵（文字、SVG、出图 img2/flare、GIF 2/4/6/10 帧、pulse、编辑）；img2 与 flare 任一 PASS 即 GOOD；初判 BAD 须复测 text/svg/img2 三格再定（瞬时上游 403 会造成假 BAD）。工具：`~/workspace/tools/pool-dissection/shard_d.py`（probe / isolate-test / expand / matrix-one / matrix-all / relay / complete-cells）；证据落 `~/workspace/tools/pool-dissection/evidence/`（每号 `acct_*_matrix.json` + 每格 raw json）。
- 只把能出图的号记为好号；坏号特征只有上游 400 `upstream_text_reply` 拒绝本身（元数据/JWT/套餐/额度/远程预检都不能判定）。

**第 4 步：起 new-api（端口 38084，仅回环）**
- 二进制 + `SESSION_SECRET`（0600 文件读入环境变量，不进日志）；启动：`setsid ./new-api-v1.0.0-rc.48 --port 38084 --log-dir logs`，30 秒内轮询 `/api/status` 到 200。现成脚本：`~/workspace/tools/yukkcat-pilot/newapi-test/start-newapi.sh`。注意：数据目录就是工作目录（`one-api.db` 落在启动 cwd），换目录启动会丢库——脚本 `cd` 到测试目录是关键。
- 首次建联（照 `setup_gateway.py` 流程，但**先修一个 bug**：该脚本原文 `base_url` 写的是 `http://127.0.0.1:38083/v1`，带 `/v1` 会拼出双前缀 405；改成 `http://127.0.0.1:38083` 再跑）：登录 root → 改随机密码（0600 文件）→ 建测试 token（响应不含 key，须走 `POST /api/token/:id/key` 或读 `one-api.db` 的 `tokens.key`）→ 建渠道 `{"mode":"single","channel":{type:1, key, base_url, models, group:"default", status:1, priority:0}}`。
- 开自用模式：只开 `SelfUseModeEnabled=true`（小写 `self_use_mode` 是死键，不用开）。
- 验通：`GET /v1/models` 200（9 模型，本地聚合）→ `POST /v1/chat/completions` 经 38084 → 38083 200。

**第 5 步：自愈与收尾**
- 重启自愈：kill 双进程 → 各自启动脚本重拉（yukkcat 4 秒、new-api 2 秒恢复，账号与调用记录完好）。只验证过进程级；VM 换机级自愈未核实。
- 收尾：测试结束把所有号恢复“正常”（注意 `restore_all_normal()` 会把额度已空的限流号也改回正常，状态字段不能当额度依据）；证据目录做字段名扫描（`access_token`/`refresh_token`/`id_token`/`sk-ant-`/`password` 零命中）再归档。

