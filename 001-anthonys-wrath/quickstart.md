# Quickstart — 验证流程（安东尼之怒）

**Feature**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md) | **Date**: 2026-10-23

本文件只写「怎么做、看到什么算通过」；实现细节与代码结构见 tasks 与源码。每次实测的操作、观察、探针输出与 key 清单追加到 `memory/log/`；只有经得起复用、带方法与适用范围的结论才升入 `memory/facts/`（严入阈值见 `memory/README.md`）。

## 1. 前置条件

- **运行期前置（人工一次性，本仓库不提交 jar）**：装好 ModTheSpire 加载器与 BaseMod、StSLib，两种方式任选——Steam 创意工坊订阅（本项目当前采用：mod 位于 `steamapps/workshop/content/646570/`，游戏目录无需 `mods/`），或手动放入游戏目录 `mods/`。判据：ModTheSpire 启动日志的 Mod list 中 `basemod` 与 `stslib` 同时出现、且 BaseMod 完成初始化；安装证据（订阅清单、启动器日志摘录或截图）记入 `memory/log/`。未完成时，一切游戏内验证都不可执行。
- **构建前置**：Java 8 + Maven；游戏本体、ModTheSpire、BaseMod、StSLib 的 jar 放在顶层 `libs/`（不入库）。
- **修改前提**：本功能涉及原版判定点拦截；拦截清单以时序探针（§3）的实测结论为准，不得按猜测实现。

## 2. 构建与加载

1. 在仓库根目录执行构建命令（首个基础任务确定确切命令，通常为 `mvn -q package`），预期 `BUILD SUCCESS` 且 `target/` 下生成 mod jar。
2. 用 `jar tf <jar>` 检查产物：包含 `strings/` 下中英双语资源、诅咒牌贴图、jar 根 `ModTheSpire.json`；不含游戏本体图片与任何占位图的意外泄漏。
3. 把 jar 放入游戏目录 `mods/`（不存在则新建；创意工坊方式下本地 mod 仍从该目录加载），启动游戏：加载日志无异常，mod 列表中出现本 mod，游戏可正常进入主菜单。

## 3. 时序探针（实现前置）

构建一个只有日志输出的探针版本，在游戏内推进到以下场景并打印触发顺序与抽牌堆内容：

- 进入战斗（战斗开始）；
- 抽牌堆耗尽触发重洗；
- 由卡牌或遗物触发的非耗尽重洗；
- 使用「搜寻」类效果直接取牌；
- 在「轮抽（Draft）」（或 Insanity / Sealed Deck）等改写开局牌组的自定义开局下进入战斗；Draft 开局牌组先为空、随后选牌填充。
- 另需一个临时关闭「抽到即负」的探针开关，供 QS-12 在开发期验证 FR-005（发布前移除，见 T035）。

探针必须回答（结论决定实现落点）：

1. 首次洗牌发生在各候选钩子的之前还是之后；哪个介入点能同时覆盖首次洗牌与重洗。
2. 效果直接取牌与普通抽牌是否走同一判定点。
3. 「轮抽（Draft）」等改写开局牌组的选项与开局牌组创建 / 填充的先后顺序。
4. 参照牌「铃铛的诅咒」在「变化」与「移除」两类操作下的实际表现。
5. 本体让参照牌「删不掉」的全部判定点（含 `CardGroup.getPurgeableCards()`、净化之泉 `FountainOfCurseRemoval`，以及是否存在不经这两处的移除路径）。
6. `PostDungeonInitialize` 钩子是否晚于 `AbstractDungeon.initializeCardPools()`；**读档路径是否触发该钩子**（决定诅咒池排除落点）。

探针输出与结论记入 `memory/log/`；未得出结论前，不得进入对应实现（置底、入手判定、不可移除拦截、诅咒池排除）。

**探针版本（开发期实现，T035 移除）**：代码位于 `src/main/java/<modId>/probe/`（`TimingProbe` 自注册、`ProbePatches` 只读观测、`AwProbeCommand` 控制台命令、`ProbeLog` 双写日志）。日志同时输出到 MTS 控制台与文件；文件默认 `%USERPROFILE%\awprobe.log`，可用下列两种方式改到工作区（日志首行会打印实际路径与来源，`ProbeLog.logPath()` 也会把路径写进 `hook.PostInitialize`）：
- `%USERPROFILE%\awprobe.logpath`：文件内写一行**绝对路径**（支持 `#` 注释与 UTF-8/ANSI，缺失或不可写时自动回退默认位置）；
- 环境变量 `AWPROBE_LOG`：写绝对路径（优先级高于指针文件；需重启 Steam 让新环境变量生效）。

推荐指向仓库根 `awprobe.log`（`.gitignore` 的 `*.log` 规则会忽略它），这样游戏退出后可直接读取、无需手工复制。控制台（BaseMod Dev Console，反引号键打开）命令：

```text
awprobe info                      局面 / 牌堆 / customMods / curseCardPool
awprobe purge                     masterDeck.getPurgeableCards() 的输入与输出
awprobe cursepool                 curseCardPool / srcCurseCardPool
awprobe find <text>               列出含该文本的 cardID（排错用）
awprobe addcard <cardID>|bell     masterDeck 加一张副本（bell = CurseOfTheBell）
awprobe bell                      同 addcard bell，免参数
awprobe transform <cardID>|bell   调 AbstractDungeon.transformCard 并记录 transformedCard
awprobe shuffledeck               直接调 player.drawPile.shuffle()（无参洗牌原语）
awprobe shuffleaction             入队真实 ShuffleAction(drawPile, true)（Reboot / DeepBreath 的牌触发非耗尽洗牌路径）
awprobe drawpile                  dump drawPile / discardPile（索引 0 = 底部），不触发洗牌
awprobe sinktest                  DrawPileOrdering 确定性自检（多副本成组/保序、缺席跳过、非牌堆组跳过）
```

- 卡 ID 大小写不敏感，`bell` 是 `CurseOfTheBell` 的别名；Tab 补全在参数位给出真实卡 ID（旧版把子命令词填进参数位，导致 `addcard` 收到 `cursepool`，见 `memory/log/018`）。若控制台仍异常，可改用 BaseMod 自带 `deck add CurseOfTheBell`（它走 `ShowCardAndObtainEffect`，与探针的 `masterDeck.addToTop` 语义不同，日志里会体现）。
- 日志中 `drawPile` 列表**索引 0 = 底部（最后抽到），末位 = 下一次抽到**（`memory/facts/f010`）。

采集场景（2026-10-24 两轮已全部执行，下列保留以便复跑）：

1. 新档自定义模式开两局（勾选「轮抽（Draft）」或「Insanity」，再开一局不勾选任何改写牌组的选项）→ 观察 `initializeStarterDeck`、`PostCreateStartingDeck`、`customMods`；
2. 第一场战斗开局 → `preBattlePrep` / `initializeDeck` / `shuffle` / `StartBattle` / 首次摸牌的先后；
3. 打到抽牌堆耗尽 → `emptyDeckShuffle` 与 `shuffle(discardPile)`；
4. 用「搜寻」类效果直接取牌（**抽牌堆与弃牌堆各取一次**）→ `addToTop` / `addToHand` 调用栈；控制台 `awprobe shuffledeck` 观察无参 `shuffle()` 原语（非耗尽重洗方向）；
5. `awprobe bell`（或 `awprobe addcard bell`）后 `awprobe info`、商店移除 / 休息点移除 / 净化之泉 → `getPurgeableCards`、`removeCard`、`fountain`；
6. `awprobe transform bell` 及变化类事件 → `transformCard` 的输入与 `transformedCard` 输出；
7. **R-006 存活实验**：PowerShell 执行 `New-Item -ItemType File -Path "$env:USERPROFILE\awprobe.inject" -Force`，然后开一局 Draft（新档）→ 日志应出现 `probe.inject | marker=AscendersBane`；看 `StartGame` / `PostDungeonInitialize` / 首战 `preBattlePrep` 的 `masterDeck` 里标记牌是否仍在（哨兵文件用后自动删除，每次实验前重建）；
8. **读档路径**：任一局打到中途 → 存档退出 → 继续 → 观察 `initializeCardPools` 触发而 `PostDungeonInitialize` 不触发（R-007 证据；本牌注册后 T027 再补池内复测）。

跑完后把日志文件交回（若已把日志指到工作区，直接告知路径即可）；结论回写 `research.md` §R-004～§R-008。

> 2026-10-24 两轮采集均已完成（`memory/log/018`、`memory/log/019`）：R-004～R-007 已定稿；R-008 的候选列表排除、商店移除链路与变化行为有实测，净化之泉 / 休息点路径留到实现后的 QS-11。后续回归按 §4 的 QS 清单执行。

## 4. 游戏内验证场景

逐条执行；每条记录「操作 → 观察 → 是否通过」，并按需求号回填 `memory/log/`。

> 2026-10-24 19:09–19:10 首轮验收实测（`mvn package` 产物 18:36；游戏日志 19:08 启动，两局）：**QS-01 / QS-02 / QS-08 通过**。证据：工作区 `awprobe.log`（事件 A#4、A#8~A#12、B#41、B#45~B#50）与 `memory/log/022`；游戏日志中 RunModStrings / CardStrings 无缺键报错，且 BaseMod 记录 `adding [ … anthonyswrath:AnthonysWrath ]` / `adding [ anthonyswrath:AnthonysWrath ]`。QS-01 的屏幕中文文案未留截图，US2 首轮实测时顺带目视复核；QS-14 未执行，留 T041 完整回归；QS-03/04/05 待 US2 实现后执行。

> 2026-10-24 US2 实现完成（T016~T019：`run/DrawPileOrdering` + `patches/vanilla/CardGroupShufflePatch`；离线自检见 `memory/log/023`），**T020 于 2026-10-24 21:22–21:26 实测通过**（`awprobe.log` 235 行；判读记录 `memory/log/026`）：
>
> | 场景 | 结果 | 证据（`awprobe.log`） |
> | :--- | :--- | :--- |
> | QS-03 首洗置底（新档 + Draft） | PASS | #10~#14、#38：首洗后 drawPile 索引 0 |
> | QS-04 牌触发非耗尽重洗 ×2 | PASS | #42~#45、#151~#154：洗前 0 → `ShuffleAction` → `cmd.drawpile` 仍 0 |
> | QS-04 抽牌堆耗尽重洗（先经 Scry 把本牌放入弃牌堆） | PASS | #106~#129：draw=0 时弃牌堆含本牌（索引 9）→ 洗后索引 0 → `emptyDeckShuffle.after` 自 drawPile=size=1 起始终在索引 0 |
> | QS-05 多副本成组 | PASS | #163~#169：17 张首洗后两张副本位于 drawPile 0、1；`awprobe sinktest` 4/4 PASS（含 uuid 相对顺序核对） |
> | QS-17 第二场战斗重新置底 | PASS | #169、#193：第二场 StartBattle 与 drawpile 均为索引 0 |
>
> **读日志注意**：探针的 `patch.shuffle().after` / `patch.shuffle(Random).after` 与 mod 置底后处理的先后**不保证**（本轮采集里无参原语曾先于置底、`Random` 重载曾后于置底），判断置底以 `hook.StartBattle` / `patch.initializeDeck.after` / `patch.emptyDeckShuffle.after` / `awprobe drawpile` 的**最终状态**为准。
>
> **「抽牌堆耗尽重洗」的到达条件（2026-10-24 二次修正，见 `spec.md` Assumptions 与 `research.md` §R-004 Reachability）**：若牌组没有任何「从抽牌堆直接弃牌」的效果，抽空抽牌堆必然先抽到本牌并判负，该分支不可达；但原版 Scry 类效果可把抽牌堆顶若干张牌直接放入弃牌堆（不入手的通道），当剩余张数 ≤ Scry 数时位于底部的本牌也会被弃掉，之后自然耗尽即可触发含本牌的重洗。本次实测正是这样复现的（本牌全程无入手事件）。早先「成品不可达」的记录已作废（`memory/log/024` → 修正 `memory/log/026`）。
>
> **T041 复现耗尽重洗的步骤（成品版）**：用观者或带 Scry 类牌的牌组（如 `CutThroughFate` / `JustLucky`）——① 正常战斗，把抽牌堆抽到只剩少量牌（本牌在索引 0，最后抽到）；② 剩余张数 ≤ 某张 Scry 牌的数量时打出该牌，在 Scry 选择界面把「安东尼之怒」放进弃牌堆；③ 继续抽空抽牌堆，下一次抽牌触发耗尽重洗；④ 用「冰冻之眼」或控制台确认重洗后本牌位于抽牌堆最底部。
>
> **遗留观察**：QS-01 的屏幕中文文案仍无截图/记录，留 T041 或下会话顺手复核；#159/#160 净化之泉把本牌从 masterDeck 删除属 US5（T029-T031）未实现的预期缺口，不是 US2 回归。

| 编号 | 对应需求 | 操作 | 预期 |
| :--- | :--- | :--- | :--- |
| QS-01 | FR-001、FR-014 | 进入自定义模式选项列表 | 可见「负面：安东尼之怒」；勾选前即可看到效果说明文本 |
| QS-02 | FR-002、US1 | 勾选该选项并开始新一局，打开牌组查看界面 | 牌组中恰好一张「安东尼之怒」诅咒牌 |
| QS-03 | SC-001、FR-006 | 进入第一场战斗，用原版遗物「冰冻之眼」（不可用时用开发者控制台）按顺序查看抽牌堆 | 首次洗牌前该牌已在牌组中；首次洗牌结束后位于抽牌堆最底部（含 Draft 的空开局情况） |
| QS-04 | SC-002、FR-006/007、EC-001 | 一场战斗中触发至少两次重洗，其中一次由卡牌 / 遗物效果触发（非耗尽）；「抽牌堆耗尽重洗」可用 Scry 类效果先把本牌从抽牌堆放入弃牌堆来复现（见 §4 步骤） | 每次重洗后该牌均在抽牌堆最底部 |
| QS-05 | FR-007 | 若存在可作用于该牌的复制类效果则优先使用；否则用控制台加入第二张副本，随后洗牌并查看抽牌堆顺序 | 两张副本成组位于底部，且相对顺序不变 |
| QS-06 | SC-003、FR-008/009 | 战斗推进直到自然抽到该牌 | 进入手牌瞬间本局立即结束，结算界面显示本局失败，无需额外操作；不要求结算界面额外说明失败原因 |
| QS-07 | EC-008、FR-008 | 用「搜寻」类效果从抽牌堆或弃牌堆直接取牌 | 同样立即判负 |
| QS-08 | EC-002、FR-003 | 自定义开局选「轮抽（Draft）」+ 勾选该选项，进入战斗 | 诅咒牌仍被加入牌组，且加入时点早于第一次洗牌（置底断言由 QS-03 覆盖） |
| QS-09 | FR-010、EC-004 | 多副本同时入手；判负后继续尝试抽牌/操作 | 只结算一次失败；判负后不再处理抽牌，也不接受可操作结果 |
| QS-10 | SC-004、FR-011、EC-010 | 不勾选该选项，分别开标准模式、每日挑战与自定义模式各一局；用事件/遗物反复产出诅咒 | 牌组与未安装本 mod 时逐张一致；任何来源都不发出本牌；自定义模式列表中该选项可见但未勾选，标准模式 / 每日挑战不出现该选项 |
| QS-11 | FR-012、EC-009 | 依次进入商店移除服务、休息点移除、净化之泉（批量清诅咒） | 本牌不出现在可移除目标中 / 不被删除；任何成功移除均判违规 |
| QS-12 | EC-003、EC-011、FR-005/012 | 用探针版（临时关闭「抽到即负」，仅开发期）把本牌放入手牌：验证不可打出、不可被主动消耗，并与参照牌同条件对照一次变化类效果、一次持「蓝蜡烛」；随后用正常版复测判负路径 | 表现与参照牌一致，本 mod 不增加限制也不放宽；探针开关不进入发布版（T035 移除） |
| QS-13 | FR-013、SC-005、宪法二 | 切换中英两种语言查看牌面；人工比对两份 `strings/` 的 key 集合 | 描述直接写明「不能从牌组中移除」且无关键词样式，措辞与参照牌对应语言一致；key 一一对应、无缺无多 |
| QS-14 | EC-006、FR-001 | 卸载 / 禁用本 mod 后加载曾选择过该选项的旧存档 | 不崩溃；本 mod 的选项不再出现；若崩溃则记录并回 `/speckit-clarify` 重新界定 EC-006 |
| QS-15 | FR-009 | 因抽到该牌判负后回到主菜单 / 角色游戏记录 | 本局计为失败 |
| QS-16 | EC-005 | 主动放弃本局（放弃战斗 / 退出） | 不触发本功能特有的失败判定，按游戏原有流程处理 |
| QS-17 | FR-006 | 不抽到该牌结束第一场战斗，进入第二场战斗后查看抽牌堆 | 该牌仍在抽牌堆最底部 |

## 5. 记录与结论归属

- **流程**：即本文件，可复现、可重跑。
- **记录**：每次实测的操作、观察、截图路径、探针输出、key 清单 → `memory/log/`，发生即写。
- **结论**：只有能说出方法（`verified`）、界定适用范围（`scope`）、且在范围外仍成立的判断才升入 `memory/facts/`。
- SC-005 的「无需阅读代码」只覆盖游戏内界面与操作（牌堆观察、判负结算、牌面与移除界面、未选中时的牌组比对）；资源 key 人工比对与探针类开发期检查不属于该条。

## 6. 完成判据（宪法第五条）

- [ ] 构建通过，产物内容符合 §2；
- [ ] QS-01 至 QS-17 逐项实测通过，并留下 `memory/log/` 记录；
- [ ] plan 的 Constitution Check 与规格 checklists 保持全过；
- [ ] 未选中路径（QS-10）与卸载路径（QS-14）均无回归。
