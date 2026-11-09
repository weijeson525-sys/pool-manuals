# 说明

- 来源：号池体系代码级暴力拆解报告（2026-11-09，老板亲令），舰队仓 results/W03-TO-W02-pool-system-dissection-20261109.md（commit 97d5141）。
- 本文件为该报告对应章节的单行本，便于分发；内容与报告一致，未改动。
- 凭据值永不进文件；复现前须先修 setup_gateway.py 的 base_url（去掉 /v1），见搭建手册第 4 步。

## 五、自主升级手册（怎么跟上游、改什么、怎么验）

**跟上游**：盯两个仓库的 release/tag——yukkcat/chatgpt2api（出图/轮换/刷新路径）与 QuantumNous/new-api（rc tag；本次基准 v1.0.0-rc.48，commit 7061447）。每次升级先读 CHANGELOG + diff，只看三条线：图片请求路径、账号轮换/失败分类、token 刷新；其余改动先记、不先跟。本次实例：3.2.6 → 3.2.7 的 CHANGELOG 唯一变化是 `image_max_n` 默认 4，不碰上游拒/轮换/额度/GIF/SVG 路径（2026-11-09 09:49 隔离场已升版验证）。

**升级流程（隔离场先行，生产另批）**：① 备份：源码目录 + `data/chatgpt2api.db` + `one-api.db`（本次 3.2.7 升级即留了 `backup-3.2.6-20261109/`）；② 隔离场部署新版；③ 跑全矩阵（每号 10 格）+ 中继复测 + 重启自愈，对照基线（本次基线见 §2.9 矩阵总表）；④ 指标对不上的先回滚备份，再挖原因；⑤ 生产切不切、什么时候切，另报老板批（本次任务只研究不动生产）。

**该改什么（按价值排序，都是本次拆解挖出来的缺口）**：
1. `services/image_failure.py` 的 `upstream_text_reply` 归类：把“号没出图能力”从请求问题改成账号能力问题——`switch_account` 改为真（失败换号）和/或 `verify_account` 改为真（失败记数复核、标异常隔离）。这是根治“坏号一直被轮询到”的最小改动点；改完须重跑全矩阵验证（坏号失败后是否换号、是否被隔离、好号是否不受影响）。
2. 按号出图能力分级：用现成的 `AccountTestService.execute`（`account_test_service.py:50-200`，chat/image 两档真实调用）给每个号打分级（能出图/不能出图/额度档），只把能出图的号放进轮换；分级结果写配置，不进代码。
3. GIF 提示词默认加动画描述：`build_image_prompt()`（`conversation.py:502`）里 gif 请求默认追加明确的动画描述（pulse 类），把真动画率从 17% 往上提；帧数参数不动。
4. new-api `RetryTimes` 与自动禁用：按需把 `RetryTimes` 从 0 提到 ≥1（换渠道重试才生效）；自动禁用默认只禁 401，如需禁 500 类须改 `status_code_ranges.go:14`（改完重测死渠道行为）。
5. 补三件没找到的东西：refresh cron（定义、脚本、日志、禁用人）、watchdog v1.2（设计文件、代码、13/13 用例、QA verdict）、`pool_models.conf`（L1–L5 定义与使用者）——先找到、再谈升级。

**怎么验**：每次改动跑三件套——① 全矩阵（20 号 × 10 格，坏号须复测三格）；② 中继复测（models 200 + chat 200）；③ kill 双进程重启自愈。指标对不上基线就回滚，不硬上。

