# qq-bridge 优化总结（相对原版 v0.2.1-r1）

> 基线：`Derpyu520/qq-bridge` v0.2.1-r1（`git clone` 时的 main）
> 时间：2026-10-27　运行环境：DSH 0.2.0-rc.2（桌面版 profile `desktop`，Web 端口 19387）+ SnowLuma v1.14.20 + QQ 3633618714
> 回归状态：`npm run test:audit` → **38/40 通过**（2 项为原版既有环境性失败，见 §6）

---

## 1. 一句话结论

原版是「**每次开口前把聊天记录重读一遍**」——记忆靠内联快照、权限靠整群白名单、人格靠静态 8,940 字前缀。
现在改成「**记得住、查得到、有立场、会挑时段花钱**」：新增一层可检索的长时记忆（902 行引擎代码）、一套按人拉黑的硬闸门、一层主人立场与口癖画像、一个空闲时段调度器，**没有改任何原有安全边界**。

---

## 2. 代码改动总览

| 文件 | 改动 | 行数 | 做什么 |
| --- | --- | --- | --- |
| `src/bridge.js` | 修改 | **+321 / -2** | 接线：记忆落盘与检索路由、拉黑闸门、立场/风格注入、ownerMention 唤醒、空闲时段闸门 |
| `src/mcp-snowluma-safe.js` | 修改 | **+47** | 注册 `qq_recall` / `qq_digest` 两个工具（关掉则不注册、不占 token） |
| `src/engine/memory/store.js` | 新增 | 367 | append-only JSONL 记忆库 + 召回记录（`recall.json`）+ 主人语料接口 |
| `src/engine/memory/index.js` | 新增 | 239 | 中文 bigram 倒排 + 三要素打分 + 遗忘曲线 + 群情统计 |
| `src/engine/memory/recall.js` | 新增 | 91 | 检索结果的「给模型看的紧凑形状」（限 6 条 / 1,200 字符） |
| `src/engine/persona/owner-style.js` | 新增 | 171 | 主人说话风格统计 + 立场文案 + 画像增量融合 |
| `src/engine/schedule/offpeak.js` | 新增 | 34 | 空闲时段闸门（复用 `model-prices` 的 `isPeakAt`） |
| `scripts/test-*.mjs` | 新增 5 个 | 591 | 97 项离线断言 |
| `scripts/audit-bridge-harness.mjs` | 修改 | +14 / -1 | 把新模块注入测试沙箱（否则 bridge 在 vm 里 `ReferenceError`） |
| `scripts/test-audit.mjs` | 修改 | +9 | 5 个新测试纳入总回归 |
| `config.example.json` / `README.md` | 修改 | +32 / +5 | 配置模板与文档 |

**总计：5 个新模块（902 行）+ 5 个新测试（591 行）+ 已跟踪文件 +428 / -3。**

> 注：`plugins/qq-agent-presets/presets/*.patch.yml` 也显示被改，那是 `node scripts/setup-dsh.mjs` 重新生成的产物，不是手工编辑。

---

## 3. 七项能力：原版 vs 现在

| # | 能力 | 原版 | 现在 |
| --- | --- | --- | --- |
| 1 | **长时记忆** | 只在内存留最近 100 条，正文截到 200 字；重启即丢 | `state/memory/<群或好友>/messages.jsonl` 全量落盘（默认留 90 天，正文 2,000 字），启动异步裁剪 |
| 2 | **旧事召回** | 只能靠唤醒时的内联快照（≤6,000 字符），超了就没有 | 模型可用 `qq_recall(query=…)` 主动检索几天前的旧事；`qq_digest(sinceMin=…)` 拿零成本群情统计（谁在说/高频词/最热时段） |
| 3 | **检索质量** | —— | 抄 **Generative Agents** 的三要素：相关性 × 0.5 + 重要性 × 0.3 + 新近度 × 0.2；重要性由启发式打分（长句/带问号/被 @/主人发言更高） |
| 4 | **遗忘曲线** | 固定半衰期（话题 2 小时、记忆 24 小时粗糙清理） | 抄 **MemoryBank** 的 `R = 2^(-t / (48h × S))`：每被召回一次 `S += 1`（上限 10）⇒ 反复用到的梗更难忘，一次性冷知识自然褪色 |
| 5 | **群里某人烦人** | 只能整群屏蔽（`deny.groups`） | `socialV2.ignore.speakersByGroup`：按人拉黑，**硬闸门**（@ 你、提问、拍一拍都不唤醒），消息仍入库保证上下文完整，`"*"` 通配所有群，**改完不用重启**（mtime 热读） |
| 6 | **主人立场** | 无（只有 `【管理员】` 身份标记） | 唤醒包固定带 `【立场】`（默认站主人这边、但不编造事实）；有人在群里**点名/暗讽主人**时用独立唤醒原因 `ownerMention` 主动介入，**10 分钟限流 1 次**；`loyalty.mode` 可选 `support`（默认）/`naive`（无脑帮腔，不建议） |
| 7 | **主人说话风格** | 无 | 从主人在该会话的发言统计：句长中位数、语气词（呗/啊…）、感叹号/省略号/波浪号习惯、反问率、**只在主人高频出现的口头禅**；压成 ≤150 字一行注入，样本 <6 条不注入。**纯本地统计，0 模型调用** |

另外一处**不花钱但重要**的改动：

| # | 项 | 原版 | 现在 |
| --- | --- | --- | --- |
| 8 | **后台作业时段** | 不分时段，24 小时同价 | 黑话研究这类「不急着做」的作业先过空闲时段闸门：落在价目表峰值（工作日 9-12 / 14-18，单价 ×2）就推迟并打日志，等空闲再跑 ⇒ **这部分成本直接减半** |

---

## 4. 成本与体积影响

| 项 | 变化 | 说明 |
| --- | --- | --- |
| 每轮固定注入 | **+约 250 字符**（≈110 token） | 立场行（~97 字）+ 风格行（≤150 字）。样本不足时只有立场行 |
| 工具面 | **+972 字符**（`qq_recall` 579 + `qq_digest` 393），工具数 33 → **35** | 关掉 `socialV2.tools.recall/digest` 就**完全不注册**，不占 token |
| 写入 | 每条消息一次 append（<1KB），**0 次 API 调用** | 记忆与画像都是纯本地计算 |
| 检索 | 每次 `qq_recall` = 一次工具往返 | 默认限 6 条 / 1,200 字符；唤醒包里写明「每次唤醒最多查两次」 |
| 后台作业 | 峰谷调度后约 **-50%** | 只影响黑话研究这类原本就在跑的作业 |

**没有改动**：推理强度保持 `max`（原版 `config.example.json` 也是 max）、预设 persona 8,940 字未动、未读水位协议未动、会话生命周期未动。

---

## 5. 新增测试（97 项断言）

| 测试 | 断言数 | 覆盖 |
| --- | --- | --- |
| `test-memory-recall.mjs` | 56 | 中文 bigram 分词、三要素排序、复习效应、强度跨进程恢复、主人语料隔离、去重、落盘、裁剪、体积上限、路径穿越拒绝 |
| `test-ignore-speakers.mjs` | 12 | 拉黑不唤醒但入库、不消耗 bootstrap、拍一拍不能绕过、`"*"` 通配与按群并集、**改配置立即生效** |
| `test-owner-style.mjs` | 13 | 句长/语气词/标点/口癖统计、体积上限、**特征不足时宁可不注入**、立场文案、增量融合、重算时机、机器人昵称不算口癖、长群名片截断 |
| `test-loyalty.mjs` | 9 | `ownerMention` 触发、10 分钟限流（按会话独立）、无关谩骂不触发、开关可关、唤醒包内容、样本不足不编造 |
| `test-offpeak.mjs` | 7 | 峰谷判定与 `model-prices` 同源、周末不算峰值、等待时间、闸门放行/拦截/`allowPeak`、价目表异常时退化为放行 |

---

## 6. 真实数据暴露并修掉的 5 个坑

这些不是设计阶段想到的，是跑起来之后被抓出来的：

1. **口头禅抓到日文碎片**：主人 @ 机器人时，它那串日文昵称混进了「主人语料」，画像输出 `「もう」「う嫌」`。→ `analyzeStyle` 增加 `excludeWords`，候选词是机器人昵称子串就跳过。
2. **立场行把群名片当名字**：真实群名片是 20 字的梗名，整行撑到 120+ 字符。→ 名字截到 12 字。
3. **`digest` 高频词是 bigram 碎片**（`我上 上周 周买 买的`）：搜索需要 bigram，但摘要需要词形。→ 拆出 `tokenizeWords()` 仅供摘要用。
4. **主人标记缺失**（历史上所有消息都没这个字段）：风格学习拿不到语料。→ 加字段 + 一次性回填 17 条（按 `uid == ownerQQ`）。
5. **索引裁头后 doc 下标错位**：缓存超过上限裁掉最旧消息时，索引里的下标全部偏移。→ 裁头即作废索引、检索时惰性重建，不再 `addDocument` 硬接。

另外两个**原版既有**的测试失败（与本次改动无关，仅环境问题）：

- `test-voice-system.mjs`：依赖 `docs/guides/VOICE.md`，该文件被 `.gitignore` 排除，仓库里本来就没有；
- `test-preset-local-tools.mjs`：需要读取 Electron 打包后的 `node_modules`（`@deepseek-ai/*` 在 `app.asar` 内），打包形态读不到。

---

## 7. 运维配置差异（不属于代码，但缺了就跑不起来）

| 项 | 原版默认 | 本机设置 | 原因 |
| --- | --- | --- | --- |
| DSH profile | `web` | **`desktop`** | 桌面版带的是 `desktop` profile，`setup-dsh.mjs` 必须显式传 `desktop` |
| `dsh.baseUrl` | `http://127.0.0.1:3080` | **`http://127.0.0.1:19387`** | 桌面版 Web 端口是 19387 |
| `snowluma.homeDir` / `launcherPath` | 空 | `D:\SnowLuma` | token 自愈要读 `config/onebot_<QQ>.json` |
| `snowluma.accessToken` | 空 | SnowLuma 的 **HTTP** token（43 字节） | HTTP 与 WS token 是两个不同的值，WS 由桥接自愈 |
| `ownerQQ` | `null` | `1903813061` | 审批/管理命令/closed-agent 模式 |
| `allow.private` | `[]` | `["1903813061"]` | 空白名单 = 不响应任何消息（fail-closed） |
| `allow.groups` | `[]` | `["798377342"]` | WSCTF命题组 |
| `socialV2.ignore` | 无 | `{"*": ["3222106915"]}` | 全群不回应此人 |
| `dsh.reasoningEffort` | `max` | `max`（保持） | 明确要求推理拉满 |

---

## 8. 未做 / 下一步

按之前评估过的借鉴清单，还没落地的：

| 项 | 为什么没做 |
| --- | --- |
| 模型版风格画像提炼（每 24h 一次，夜里跑） | 形状已留好（`mergeStyle`，`new = 0.3·旧 + 0.7·新`），等统计版攒一周数据再上，成本约 ¥0.002/天 |
| few-shot 原话注入（`【主人原话示例】`） | 每轮 +300~400 字符，等看过统计版效果再决定 |
| persona 拆 core / archival（8,940 → 1,500 字） | 要动 preset + 约 60 条契约断言，单独一轮 |
| 会话轮换 + 摘要播种（M2） | 让上下文真正有界，是最大的一块，需要先有 episode 摘要 |
| 话题板 + 风格档 + 模型摘要（M3） | 依赖 M2 |
| FrugalGPT 式判定器（小模型决定要不要醒） | 要先有埋点数据才知道拦得准不准 |
| 空闲时段调度覆盖更多作业 | 目前只接了黑话研究 |

---

## 9. 回滚方式

全部改动都有开关或可 `git` 回退：

```powershell
# 1) 只关记忆（保留代码）
#    config.json → socialV3.memory.enabled = false  （或 socialV3.enabled = false 全关）

# 2) 只关立场/风格
#    config.json → socialV3.persona.loyalty.enabled = false
#                  socialV3.persona.style.enabled = false

# 3) 只关拉黑名单（清空即可，热读、无需重启）
#    config.json → socialV2.ignore.speakersByGroup = {}

# 4) 关掉两个新工具（连 schema 都不注册，需重启 DSH）
#    config.json → socialV2.tools.recall = false / digest = false

# 5) 彻底回退代码
git checkout -- src/bridge.js src/mcp-snowluma-safe.js scripts/audit-bridge-harness.mjs scripts/test-audit.mjs
Remove-Item -Recurse -Force src/engine
```

改 `config.json` 的注意：`socialV2.ignore` 与 `socialV3.persona` 之外的大部分字段**只在启动时读**，改完要重启桥接；`ignore` 是热读的。

---

## 10. 数据落在哪

```
state/
  memory/
    group_798377342/
      messages.jsonl    # 全量消息（含 owner / imp / s 字段）
      recall.json       # seq → {s, at}：召回强度与时间（append-only 之外的旁路）
    private_1903813061/
      ...
  social-v2.json        # 未读/最近/唤醒配置（原版就有）
  sessions.json         # QQ 会话 ↔ DSH 会话映射（原版就有）
  token-usage.jsonl     # 逐轮用量账本（原版就有，控制台看板读它）
```

记忆库的 `messages.jsonl` 是**纯追加**；想重置某个会话的记忆，直接删掉对应目录即可（下次唤醒会重建，不影响 DSH 侧会话）。
