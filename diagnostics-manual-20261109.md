# 说明

- 来源：号池体系代码级暴力拆解报告（2026-11-09，老板亲令），舰队仓 results/W03-TO-W02-pool-system-dissection-20261109.md（commit 97d5141）。
- 本文件为该报告对应章节的单行本，便于分发；内容与报告一致，未改动。
- 凭据值永不进文件；复现前须先修 setup_gateway.py 的 base_url（去掉 /v1），见搭建手册第 4 步。

## 四、诊断手册（症状 → 查哪 → 怎么修）

| 症状 | 查哪 | 结论与修法 |
|---|---|---|
| 出图 400 `upstream_text_reply`，返回上游中文/英文拒绝文本 | 该号全矩阵：文字/SVG 是否 PASS；`call_records` 里 served_by 是不是同一号 | 号本身没出图能力（上游拒因未核实）。换号无用（400 不换号）；修法：验活定级，把该号标坏号隔离（手动）；或改 `image_failure.py` 的 `upstream_text_reply` 归类（改为可换号/计入账号能力，见 §五） |
| 出图 429 `insufficient_quota`，随后 503 `no_available_account` | `/api/accounts` 该号 success + remaining；reset_after | 额度耗尽。等 reset_after，或换有额度号；不要反复重试（失败不扣额度但会把号标限流） |
| 出图 503 `no_available_account`（0.1s） | 各号状态：是否全“禁用”/“限流”；是否刚经历额度耗尽连锁或上游 403 | 选号阶段无可用号。恢复可用号（检查限流原因），不要当成单个号坏了 |
| edit 502 `image_stream_timeout`（~122s） | 重试一次 | 瞬时上游问题（疑似网关固定流超时阈值，成功 edit 35–50s）。重试通常 PASS |
| 出图 502 `upstream_connection_failed` / 503 | 重试一次；查出口代理是否通 | 瞬时连接问题。重试通常 PASS；若是出口代理故障（10:06 事件），先修代理再重启服务 |
| text 格瞬时上游 403（HTML 页）随后多格空账号失败 | 35 秒后复测 text/svg/img2 | 瞬时上游事件，不要直接判 BAD。复测三格再定 |
| GIF 全是静态重复 | 提示词里有没有明确的动画描述 | 改提示词（加 “pulsing bigger then smaller, smooth animation” 类描述）；不要调帧数（帧数只有弱梯度） |
| SVG 400/502 `upstream did not return a valid SVG document` | 上游返回的正文里有没有完整 `<svg…</svg>` | 模型没按指令返回 SVG。重试；SVG 不区分好坏，不要换号 |
| 中继 400 `Model price is not configured` | `SelfUseModeEnabled` 是否为 true | 只开 `SelfUseModeEnabled`（小写 `self_use_mode` 是死键）；普通模式须给每个模型配定价 |
| 中继 405 | 渠道 `base_url` 是否带 `/v1` 或尾斜杠 | 去掉 `/v1` 与尾斜杠（纯字符串拼接，无归一化） |
| 中继 503 `No available channel for model …` | 渠道 `models` 白名单有没有该模型；token 的 group 与渠道 group 是否匹配 | 补模型名或改 group |
| 中继失败不重试 | `RetryTimes` 是否为 0 | 默认 0（恰好 1 次尝试）。想让网关换渠道重试，显式设 `RetryTimes` ≥ 1（每次重试可能重新计费预扣） |
| 服务停了 / 连不上 38083/38084 | 先查出口代理（`hatch-egress-proxy:3128` 是否通），再查进程是否在 | 10:06 事件证明：出口故障会让双服务停机。代理恢复后用现成脚本重启（yukkcat 6 秒、new-api 2 秒 healthy）；不要把代理故障期间的 502 计入账号失败 |
| 某号 `status` 长期“异常” | 是否 `invalid_grant` / 401 吊销；`auto_remove_invalid_accounts` 是否默认 true（直接删除） | 吊销是事后处理。查 `_apply_invalid_token_state` 留下的 `invalid_count` 与 `last_remote_check_result`；需补号走新号验活流程 |
| 模型列表不对（少模型/多模型） | new-api `/v1/models` 是本地聚合（渠道表模型名）；yukkcat 模型来自 catalog（`model_catalog_service.py:118-169`） | 两边模型名要对上；1.8.8 无 `/v1/models` 路由，生产实际怎么补模型列表未核实 |

