# Quickstart — 验证流程（安东尼之怒）

**Feature**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md) | **Date**: 2026-10-23

本文件只写「怎么做、看到什么算通过」；实现细节与代码结构见 tasks 与源码。每次实测的操作、观察、探针输出与 key 清单追加到 `memory/log/`；只有经得起复用、带方法与适用范围的结论才升入 `memory/facts/`（严入阈值见 `memory/README.md`）。

## 1. 前置条件

- **运行期前置（人工一次性，本仓库不提交 jar）**：装好 ModTheSpire 加载器与 BaseMod、StSLib，两种方式任选——Steam 创意工坊订阅（本项目当前采用：mod 位于 `steamapps/workshop/content/646570/`，游戏目录无需 `mods/`），或手动放入游戏目录 `mods/`。判据：ModTheSpire 启动日志的 Mod list 中 `basemod` 与 `stslib` 同时出现、且 BaseMod 完成初始化；安装证据（订阅清单、启动器日志摘录或截图）记入 `memory/log/`。未完成时，一切游戏内验证都不可执行。
- **构建前置**：Java 8 + Maven；游戏本体、ModTheSpire、BaseMod、StSLib 的 jar 放在顶层 `libs/`（不入库）。
- **修改前提**：本功能涉及原版判定点拦截；拦截清单以时序探针（§3）的实测结论为准，不得按猜测实现。

## 2. 构建与加载

1. 在仓库根目录执行构建命令（首个基础任务确定确切命令，通常为 `mvn -q package`），预期 `BUILD SUCCESS` 且 `target/` 下生成 mod jar。
2. 用 `jar tf <jar>` 检查产物：包含 `strings/` 下中英双语资源、诅咒牌贴图 `anthonyswrath/img/cards/AnthonysWrath.jpg`、jar 根 `ModTheSpire.json`；不含游戏本体图片与任何占位图的意外泄漏（如 `placeholder/`）。
3. 把 jar 放入游戏目录 `mods/`（不存在则新建；创意工坊方式下本地 mod 仍从该目录加载），启动游戏：加载日志无异常，mod 列表中出现本 mod，游戏可正常进入主菜单。

> **2026-11-05 产物实测（T038）**：`target/anthonyswrath.jar` 与游戏 `mods/anthonyswrath.jar` 同哈希 `BBF6C534…D3AC`（113,394 B，01:17:18）；含两张贴图、四份双语资源与根 `ModTheSpire.json`。`jar tf` 另有一个**空** `placeholder/` 目录条目（旧 `target/classes` 构建残留，无图片，不构成占位图泄漏；用 `mvn clean package` 重建即消失）。

## 3. 时序探针（实现前置）

> **2026-11-04（T035）**：探针（`probe/` 与 `awprobe` 控制台命令、临时关闭判负的开关）已从发布版删除，本节保留为历史复现说明——需要复跑探针时请从 Git 历史取回 `probe/` 或回退 T035 的删除提交。**发布版的 T041 回归按 §4 的 QS 表用游戏内手段执行**（冰冻之眼 / 开发者控制台 / 本体存档）；§4 中标为 `awprobe` 的段落是探针版实测记录，不是发布版可执行步骤。

构建一个只有日志输出的探针版本，在游戏内推进到以下场景并打印触发顺序与抽牌堆内容：

- 进入战斗（战斗开始）；
- 抽牌堆耗尽触发重洗；
- 由卡牌或遗物触发的非耗尽重洗；
- 使用「搜寻」类效果直接取牌；
- 在「轮抽（Draft）」（或 Insanity / Sealed Deck）等改写开局牌组的自定义开局下进入战斗；Draft 开局牌组先为空、随后选牌填充。
- 另需一个临时关闭「抽到即负」的探针开关，供 QS-12 在开发期验证 FR-005（发布前移除，见 T035；已于 2026-11-04 随 T035 删除，QS-12 的验证见 §4 的 T034 结果块）。

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

> 2026-10-24 US3 实现完成（T021~T024：`run/DefeatOnDraw` + `patches/vanilla/CardGroupHandEntryPatch` + `AbstractPlayerDrawPatch` + `StartGame` 重置；离线自检见 `memory/log/027`），**T025 已于 2026-10-29 00:04–00:33 实测通过**（`awprobe.log` 727 行、6 局；判读记录 `memory/log/028`）。本轮使用的 jar 为 `target/anthonyswrath.jar` / `mods/anthonyswrath.jar`（2026-10-28 16:10:55，55 600 bytes，SHA256 `5B49071B…4AB8`）。执行本段前仍需用普通终端 `mvn package` 重建并覆盖 `mods/anthonyswrath.jar`（沙箱内 Maven 不可用，事实 f008），并删除旧 `awprobe.log`。
>
> **T025 步骤**（全部使用「观者 + 本选项」新档；若用其它角色亦可，步骤不变）
>
> 1. **QS-06 自然抽到即负**：进第一场战斗，先用 `awprobe drawpile`（或「冰冻之眼」）确认本牌位于抽牌堆索引 0（底部）；正常推进回合、不断抽牌直到本牌入手。预期：本牌进入手牌瞬间播放死亡动画并进入失败结算（显示本局失败），无需任何额外操作。证据：`awprobe.log` 中 `patch.addToHand … card=anthonyswrath:AnthonysWrath` 之后出现**恰好一次** `patch.deathScreen.before` / `patch.deathScreen.after` 成对记录。
> 2. **QS-07 搜寻取牌**：新开一局（或读档），用 `awprobe addcard Seek`（Defect 的「搜寻」）/ `awprobe addcard Hologram`（「全息影像」）把检索牌加入 masterDeck——**下一场战斗才进入牌组**；`awprobe find <文字>` 可核对 ID。两种取牌路径任选其一或都做：
>    - **抽牌堆检索（Seek）**：抽出「搜寻」后打出，网格会列出整个抽牌堆，选中「安东尼之怒」并确认。预期：取牌完成瞬间判负；`awprobe.log` 出现 `patch.addToTop(hand) … card=anthonyswrath:AnthonysWrath`，其后接一次 `patch.deathScreen.*`。
>    - **弃牌堆检索（Hologram）**：先按上文 T041 的 Scry 步骤把本牌从抽牌堆放入弃牌堆，再打「全息影像」从弃牌堆取回本牌。预期：同样判负；日志出现 `patch.addToHand … card=anthonyswrath:AnthonysWrath` 后接一次 `patch.deathScreen.*`。
>    - **不要用** Secret Technique / Secret Weapon 作样本：它们只能检索技能 / 攻击，本牌是诅咒，取不到。
> 3. **QS-09 多副本只结算一次 + 终局短路**：用 `awprobe addcard anthonyswrath:AnthonysWrath` 加入第二张副本（下一场战斗生效；首洗后两张副本成组位于 drawPile 索引 0、1，见 QS-05）。核心判据：**本局内**（自该局 `hook.PreStartGame` 起至下一次 `hook.PreStartGame` 前）`patch.deathScreen.*` 只成对出现一次；判负后不再出现新的 `patch.draw(int).before`。注意不要按「整份日志只出现一次」判定——QS-06 / QS-07 的日志与它共用同一份 `awprobe.log`（2026-10-29 实测后修正判据，见下方结果块）。
>    - **基本做法**：把抽牌堆抽到只剩两张（两张本牌），再打一次抽 ≥2 张的牌（如 `awprobe addcard Skim`，「快速浏览」抽 3）或自然抽牌。第一张入手即判负，终局短路使后续抽牌不再发生，不会出现第二次 `deathScreen`。
>    - **可选（同一动作内两张同时入手，更严格）**：用 `awprobe addcard ThirdEye`（「第三只眼」，Scry 3）在抽牌堆只剩两张本牌时把两张一起放进弃牌堆；再打 `awprobe addcard Meditate`（「冥想」，从弃牌堆取至多 2 张）一次取回两张。预期：仍然只有**一次** `deathScreen`，第二张的判定被一次性守卫拦下。
>    - 判负后死亡结算界面接管输入，尝试任何操作应无响应。
> 4. **QS-15 失败记录**：在失败结算界面点「返回主菜单」，到「角色统计 / 游戏记录」确认本局计为失败（loss 计数 +1），游戏日志含 defeat metrics。
> 5. **QS-16 主动放弃不触发特判**：新开一局，进战斗后不抽到本牌即「保存并退出」/放弃本局。预期：按本体流程处理；该流程中**没有由本牌触发**的 `patch.deathScreen.*`（判据见下方「判据修正」第 2 条；本体放弃流程自身会构造一条 `DeathScreen`，不算违规）。
> 6. 顺手复核 QS-01 中文选项屏幕文案（历次遗留项）。
>
> **判负证据线**：`patch.deathScreen.before` 的 `isDead=… hp=…` 与 `patch.deathScreen.after` 的 `screen=DEATH` 成对出现即判负落点生效；本牌入手后没有这两行则落点未生效（回传日志排查）。探针的 `patch.addToHand` 行可能早于本 mod 的判负后处理，判读以 `patch.deathScreen.*` 与后续是否还有 `patch.draw(int).before` 为准。
>
> **测试可行域变化（US3 之后）**：探针版「抽到本牌不判负」的前提已消失；复现 QS-04 的「抽牌堆耗尽重洗」必须先按上文 T041 步骤用 Scry 把本牌从抽牌堆放入弃牌堆，否则抽空抽牌堆会先抽到本牌并判负。US5（T034）验证 FR-005 时需要临时关闭判负，用 `awprobe defeatoff` / `awprobe defeaton`（开发期开关，随 T035 移除）。

> **T025 结果（2026-10-29 00:04–00:33 实测通过，证据 `awprobe.log` 727 行 + 用户提供两张游戏记录截图 + 游戏日志 + `runs/WATCHER/*.run`；判读记录 `memory/log/028`）**
>
> | 场景 | 结果 | 证据 |
> | :--- | :--- | :--- |
> | QS-06 自然抽到即负 | PASS | #73 `patch.addToHand … card=anthonyswrath:AnthonysWrath`（caller `CardGroup.addToHand <- AbstractPlayer.draw`）→ #74 `patch.deathScreen.before`（`isDead=true`，caller 链含 `CardGroup.addToHand`）→ #75 `after screen=DEATH`，恰好一次 |
> | QS-07 抽牌堆检索（Seek） | PASS | #144 `patch.addToTop(hand) … card=anthonyswrath:AnthonysWrath`（caller `BetterDrawPileToHandAction`；即「搜寻」的 `SeekAction` 路径）→ #145/#146 成对 |
> | QS-07 弃牌堆检索（Hologram） | PASS | #359 `patch.addToHand … card=anthonyswrath:AnthonysWrath`（caller `BetterDiscardPileToHandAction`）→ #360/#361 成对 |
> | QS-09 多副本只结算一次 + 终局短路 | PASS | #494 第一张入手 → #495/#496 成对；#498 同一次「冥想」取回的第二张只产生 `patch.addToHand`、**没有**第二次 `deathScreen`（一次性守卫生效）；判负后至下一次 `PreStartGame` 没有新的 `patch.draw(int).before` |
> | QS-15 失败记录 | PASS | 截图两条游戏记录（`00:28-5` / `00:30-5`）与 `runs/WATCHER/1793204894.run`、`1793205055.run` 一致：`floor_reached=1`、`victory=false`、`score=5`、牌组含两张本牌；`1793205200.run` 为主动放弃局（同样记负） |
> | QS-16 主动放弃不触发特判 | PASS（判据已修正） | #308/#309 与 #540/#541 的 `deathScreen` 由本体放弃流程 `ConfirmPopup.yesButtonEffect` 构造（本体行为：放弃即失败结算并记负）；全程没有 caller 链含 `CardGroup.addToHand` / `addToTop` 的、由本牌触发的 `deathScreen` |
>
> **判据修正（2026-10-29 实测发现原判据不可执行）**
> 1. QS-09 原判据「整份 `awprobe.log` 中 `patch.deathScreen.*` 只成对出现一次」与本段其它步骤共用同一份日志矛盾（QS-06 / QS-07 也会产出成对记录）。正确判据为**按局**：自该局 `hook.PreStartGame` 起至下一次 `hook.PreStartGame` 前，`patch.deathScreen.*` 只成对出现一次。
> 2. QS-16 原判据「日志中 `deathScreen` 计数保持 0」与本体行为矛盾：核塔 1 的「放弃本局」本身就会构造 `DeathScreen` 并记负（本次实测 `ConfirmPopup.yesButtonEffect` 调用链 + 运行记录双证）。正确判据为**由本牌触发**的 `deathScreen` 为 0，即没有任何一条 `deathScreen` 的 caller 链含 `CardGroup.addToHand` / `CardGroup.addToTop`，且该局没有本牌入手事件。
>
> **第三方 Mod 干扰（判读陷阱，已排除）**：创意工坊 Mod「快速重开按钮」（`quickRestart`，创意工坊 ID `1805046408`）的「评分重开」在 `receivePostRender` 中调用 `RestartRunHelper.scoreAndRestart()` → 构造 `DeathScreen`（`isDead=false`、探针可见 caller 为 `BaseMod.publishPostRender`，因为探针的 caller 过滤只保留 `com.megacrit` / `basemod` 帧）。本次 #185、#405 即此类；游戏日志同一时刻有 `QuickRestart: Scoring and run restart has been initialized.`，且这两次都没有本牌入手事件。判读以「caller 链是否含本牌入手」+「该局是否有本牌入手记录」为准（详见 `memory/log/028`、事实 f022）。

> 2026-10-29 US4 实现完成（T026：`RunState.beginRun()` 每局捕获一次选中态，置底与判负组件按 FR-011 短路；T027：卡牌类加 BaseMod `@NoPools` + `CardLibraryGetCursePatch` 过滤两个 `CardLibrary.getCurse` 重载；离线自检 26/26 见 `memory/log/029`），**T028 已于 2026-10-29 01:52–02:03 实测通过**（`awprobe.log` 293 行 / 186 事件 / 4 局，判读记录 `memory/log/031`；结果块见本段末尾）。本轮 jar 由普通终端重建：`target/anthonyswrath.jar` 与 `mods/anthonyswrath.jar` 同哈希 `C63C9B46…2DC81`（65 064 B，2026-10-29 01:43:11）；T034 沿用该构建，无需再次重建，执行前删除旧 `awprobe.log`。
>
> **T028 步骤**（QS-10，覆盖 SC-004；三局都不勾选本选项）
>
> 0. **加载自检**：启动游戏后先确认 Mod list 含 `anthonyswrath (0.1.0)`、MTS 控制台 / 游戏日志没有本 mod 的 patch 报错。本轮 T027 / T029 首次使用 `@SpirePatch2`（MTS 3.30.3 支持；若加载器报 `PatchingException` 或 mod 未出现在 Mod list，先停止测试并回传日志）。
> 1. **标准模式**：主菜单「标准模式」开一局（任意角色，不要选自定义）。进首战后依次执行 `awprobe info`、`awprobe cursepool`、`awprobe curseprobe`（默认采样 50 次，两个 `getCurse` 重载各半）。预期：`masterDeck` 与未安装本 mod 时逐张一致（无 `anthonyswrath:AnthonysWrath`）；`cursepool` / `srcCurseCardPool` 都不含本牌；`curseprobe` 输出 `PASS … noArgHits=0 avoidHits=0 poolsClean=true`。随后正常推进到宝箱 / 诅咒钥匙 / 随机诅咒事件，确认不会出现本牌（QS-10 场景 4）。
> 2. **每日挑战**：主菜单「每日挑战」开一局，重复第 1 步的三条命令。预期同上；自定义模式选项不出现在每日挑战（QS-10 场景 2）。
> 3. **自定义模式（勾选本选项之外的自定义组合）**：自定义模式开一局，选项列表确认「安东尼之怒」可见但**未勾选**，开始后重复第 1 步三条命令。预期：牌组无本牌、`drawPile` 的 `tracked={none}`（置底不介入）、正常战斗不会因本牌判负（本牌不存在）、随机诅咒来源不出本牌。
> 4. **对照（勾选本选项）**：同一份构建再开一局勾选本选项的自定义局，执行 `awprobe curseprobe`。预期仍 `PASS`（勾选与否都不应被随机诅咒来源发出；本牌只能来自开局注入）；`awprobe info` 确认 masterDeck 恰好一张本牌。
> 5. 结论写入 `memory/log/`：三局 `masterDeck` 前后对照 + 三次 `curseprobe` 输出（QS-10 / SC-004）。

> **T028 结果（2026-10-29 01:52–02:03 实测通过，证据 `awprobe.log` 293 行 / 186 事件 / 4 局 + 4 个 `runs/` 记录 + 游戏日志与 MTS 进程日志；判读记录 `memory/log/031`）**
>
> | 场景 | 结果 | 证据 |
> | :--- | :--- | :--- |
> | 加载自检 | PASS | MTS 日志 `Mod list` 含 `anthonyswrath (0.1.0)`；`CardLibraryGetCursePatch` 两个重载、`CardGroupGetPurgeableCardsPatch`、`FountainOfCurseRemovalPatch` 等均显示 `- SpirePatch2` 并已应用；两处 `getCurse` 调用被改写为 `return ($r)…Postfix(...)`，证明返回值替换在运行期生效；无本 mod 的 MTS 补丁异常（游戏日志仅第三方 Loadout 的 INFO 级 “Error patching …”，历次已记录且不崩溃） |
> | 标准模式未选中（#4 起） | PASS | #8/#9 注入钩子前后 `masterDeck` 均 10 张且逐张相同、无本牌；#43 `info` 同；#45 `cursepool` / `srcCurseCardPool` 均 10 张原版诅咒、不含本牌；#47 `curseprobe PASS samples=50 noArgHits=0 avoidHits=0 poolsClean=true`；#16/#17 洗牌 `tracked={none}`（置底未介入）；`runs/WATCHER/1793210208.run`：标准局、牌组 10 张、无本牌 |
> | 每日挑战未选中（#50 起） | PASS | #50 `enabledMods=[Insanity, Time Dilation, Midas]`；#54/#55 钩子前后 `masterDeck` 均 50 张（Insanity 随机开局）且逐张相同、无本牌；#88/#90/#92 同标准局输出；`runs/DAILY/1793210256.run`：`is_daily=true`、`daily_mods` 一致、牌组 50 张无本牌 |
> | 自定义模式未勾选（#95 起） | PASS | #99/#100 钩子前后均 10 张、逐张相同、无本牌；#134 `tracked={none}`；#136/#138 同标准局输出；`runs/WATCHER/1793210297.run`：`is_trial=true`、牌组 10 张无本牌 |
> | 对照：勾选本选项（#141 起） | PASS | #146 `masterDeck` 10→11、恰好一张本牌；#153/#154 首洗后 `tracked` 由 @7 → @0（置底生效）；#180 `info` 11 张含恰好一张；#182/#184 同上 PASS；`runs/WATCHER/1793210631.run`：`is_trial=true`、牌组 11 张含恰好一张本牌 |
>
> **判读注意（2026-10-29）**
> 1. 四局均以「放弃本局」结束：四次 `patch.deathScreen.*` 的 caller 链都是 `ConfirmPopup.yesButtonEffect <- SettingsScreen.update`（本体放弃流程），**不是**由本牌触发；本牌入手事件 0 次（控制局本牌位于抽牌堆底部，未抽到）。
> 2. 本段未实际推进到宝箱 / 诅咒钥匙 / 随机诅咒事件；随机诅咒来源的判据由 `curseprobe` 源级采样承担（4 局 × 50 × 两个 `getCurse` 重载 = 400 次调用，0 命中，两池双清）。本体所有随机诅咒来源都汇聚到这两个重载（事实 f023），该判据比单次事件观察更严格；但「事件 / 遗物实际产出」路径仍列入 T041 完整回归的 QS-10 必做项。
> 3. 模式归属由 `runs/` 记录交叉确认（`is_daily` / `is_trial` / `daily_mods`），不单凭 `enabledMods` 猜测；四局起始时间与 `playtime` 精确对应（89 / 35 / 26 / 297 秒）。
> 4. `awprobe.log` #3 的 `registered command:` 行是 `TimingProbe` 硬编码的旧子命令列表，**不能**据此判断 jar 为旧构建（该文本随 T035 删除）。

> 2026-10-29 US5 实现完成（T029：`CardGroupGetPurgeableCardsPatch` 过滤全部「可移除候选」；T030：`FountainOfCurseRemovalPatch` + `run/CurseRemovalGuard` 括号式保护净化之泉批量删除；T031：静态枚举确认其余移除路径已被本体自身过滤覆盖，无需额外补丁；T032：`awprobe defeatoff/defeaton` 开发期开关；T033：中英文本定稿核对；离线自检见 `memory/log/030`），**T034 已于 2026-11-04 实测通过**（沿用 T028 的同一构建，无需再次重建；结果块见下方，判读记录 `memory/log/032`）。
>
> **T034 步骤**（QS-11 / QS-12 / QS-13）
>
> 0. **加载自检**：同 T028 第 0 步（确认 mod 加载与 patch 无报错，重点看 `@SpirePatch2` 的 `getCurse` / `getPurgeableCards` 补丁）。
> 1. **候选列表（QS-11，T029）**：任意「观者 + 本选项」局（牌组含本牌）中执行 `awprobe purgetest`。预期 `PASS inDeck=true inPurgeable=false`。再进入商店移除界面（地图走到的商店）：候选网格中不应出现「安东尼之怒」；持有 Peace Pipe 时在休息点的移除候选同样不应出现；遗物 / 事件类移除（如 Empty Cage / Astrolabe / Cleric）候选同样不含本牌。
> 2. **净化之泉（QS-11，T030）**：用控制台 `event Fountain_of_Cleansing` 进入「净化之泉」事件（控制台命令见 BaseMod；若不可用则自然刷出），选择「清除全部诅咒」。预期：本牌不被删除——事件结束后执行 `awprobe info` / `awprobe purgetest`，`masterDeck` 仍含本牌；`awprobe.log` 中没有 `patch.removeCard(obj) … card=anthonyswrath:AnthonysWrath` 行。注意 `patch.fountain.before/after` 与 mod 保护补丁的先后由 MTS 决定，**判读以事件结束后的 `awprobe` 命令输出为准**。
> 3. **变化 / 蓝蜡烛对照（QS-12，T032；需要开发期开关）**：先 `awprobe defeatoff` 关闭「抽到即负」（探针构建专用，随 T035 移除）。进战斗后正常抽牌直到本牌入手（或用 `awprobe addcard anthonyswrath:AnthonysWrath` 后再抽），确认本牌停在手牌而不是结束本局。
>    - **不可打出**：本牌与参照牌（`awprobe bell` 加入后抽到）在手中都不可打出（费用 -2 的诅咒，点击无响应 / 不可用提示）；两者表现一致。
>    - **蓝蜡烛对照**：控制台 `relic BlueCandle`（或自然获得蓝蜡烛）后，本牌与参照牌都可以按本体规则以 1 点生命打出并在战斗中消耗；两者表现一致。
>    - **主动消耗对照**：`awprobe addcard FiendFire`（恶魔之焰，消耗手牌）后打出，本牌与参照牌同样被消耗、不写回牌组。
>    - **变化对照**：`awprobe transform anthonyswrath:AnthonysWrath` 与 `awprobe transform bell`——两者结果都只会是「其它随机诅咒」，本牌与参照牌都**不会**作为变化结果出现（`awprobe transform` 只计算并记录结果，不写回牌组，与 T004 相同）。`awprobe curseprobe` 也应保持 PASS。
>    - 验证结束后执行 `awprobe defeaton` 恢复判负，并按 QS-06 的方式再自然抽一次确认判负回归（只结算一次）。
> 4. **文本与 key（QS-13，T033）**：切换中英文查看牌面：中文描述「不能被打出。 NL 不能从牌组中移除。」、英文 `Unplayable. NL Cannot be removed from your deck.`，与参照牌「铃铛的诅咒」同款措辞、无关键词样式或颜色标记；两张 `strings/` 的 key 一一对应（离线已核对，见 `memory/log/030`）。（**2026-11-07 更新**：该旧文案漏记本牌特有的两项效果，已由 T044 修订；QS-13 的新判据见下方 T044 段与 §4 QS 表，旧文案仅作历史记录。新文案已于 2026-11-07 由用户重构建后目视复核通过。）

> **T034 结果（2026-11-04 23:18–23:56 实测，判读见 `memory/log/032`）**
>
> 证据：`awprobe.log` 609 行/两段启动（第一段 23:18–23:45 整局，第二段 23:46–23:48 仅建局）；`runs/WATCHER/1793807122.run`（`is_trial=true`、`floor_reached=18`、`playtime=1607`、`victory=false`、`killed_by=Slime Boss`、`master_deck` 7 张含本牌、`relics` 含 `Blue Candle` / `relic_stats.Blue Candle=[1]`）；MTS 进程日志（Mod list 含 `anthonyswrath (0.1.0)`，T029/T030 补丁均应用且 `@SpirePatch2` 生成返回值替换）；jar 哈希 `C63C9B46…2DC81`（与 T028 同构建，实测前后未变）。
>
> | 项目 | 结果 | 证据摘要 |
> | :--- | :--- | :--- |
> | QS-11 候选列表 | PASS | `purgetest` 两次 PASS（#15/#146）；商店 #50/#51、Transmogrifier #52–#55、Peace Pipe #116–#118/#151–#152、Empty Cage #119–#122、Astrolabe #123–#132、Cleric #133–#135 的候选均由 `getPurgeableCards` 产生且实际移除的不是本牌；用户目视候选列表正常、本牌不在其中 |
> | QS-11 净化之泉 | PASS | #136–#141 三次 `buttonEffect(0)` 的 `patch.fountain.*` 显示本牌被 `CurseRemovalGuard` 临时移出（预期观测）；事件后 #143 `info` masterDeck=7 含本牌、#144–#146 `purgetest` PASS；run 记录 `event_choices` Fountain `cards_removed=[]` |
> | QS-12 判负开关/手牌停留 | PASS | #148 `defeatoff`；#188/#364 本牌两次入手均无 `deathScreen` |
> | QS-12 蓝蜡烛 | PASS | run 记录 `Blue Candle:[1]`；#229 本牌经 `AbstractPlayer.useCard` 从手牌消耗（1 次触发） |
> | QS-12 恶魔之焰 | PASS | #359 控制台加 `Fiend Fire`；#364 本牌在手；#399 打出后 #400 `hand=0`、`exhaust 0→6`、`discard=0`（手牌含本牌全部消耗） |
> | QS-12 变化对照 | PASS | #417 本牌→`Injury`；#422 `CurseOfTheBell`→`Regret`；#425 `curseprobe` PASS |
> | QS-12 判负回归 | PASS | #427 `defeaton`；#469/#470 boss 战本牌入手触发一次 `deathScreen`（caller `DeathScreen.<init> <- CardGroup.addToHand <- AbstractPlayer.draw`），run 记录 `victory=false` |
> | QS-13 文本 | PASS（旧文案） | 用户 2026-11-04 目视中英牌面正常；T033 离线 key 核对（`memory/log/030` §2.5）。**2026-11-07 已由 T044 修订为四效果齐全文案，用户重构建后目视确认正常 → 新判据 PASS（见 T044 段与 `memory/log/038`）** |
>
> **留证缺口（不阻塞）**：参照牌「铃铛的诅咒」的「不可打出 / 蓝蜡烛 / 恶魔之焰」同条件对照未单独留证（bell 在 23:44 后才入牌组，boss 战与本牌同时抽到即判负）；关闭依据 = 本牌侧行为与本体诅咒一致 + 源码零拦截（T031 枚举）+ EC-011 不可达性，明细见 `memory/log/032` §3.1。如需补做，须在 T035 删除探针后的重建覆盖前用现探针 jar 复跑。
> **判读注意**：`patch.fountain.before/after` 与保护补丁的先后由 MTS 决定（本批两条日志都显示 suspend 后的 6 张，属预期）；`awprobe addcard FiendFire` 报 unknown 是因本体卡 ID 为 `Fiend Fire`（含空格），改用 BaseMod 控制台后成功，不影响判据。

> **T038 构建与加载验收结果（2026-11-05 01:17–01:22 实测通过，判读见 `memory/log/036`）**
>
> 用户普通终端 `mvn package` 产物：`target/anthonyswrath.jar` = 游戏 `mods/anthonyswrath.jar`，113,394 B，2026-11-05 01:17:18，SHA256 `BBF6C534…D3AC`。`jar tf` 核对：`anthonyswrath/img/cards/AnthonysWrath.jpg`（250×190）、`AnthonysWrath_p.jpg`（500×380）、`strings/anthonyswrath/{eng,zhs}/{cards,run_mods}.json`、根 `ModTheSpire.json`（0.1.0，无 `${...}` 占位符）、无 `probe/` 类；空 `placeholder/` 目录条目见 §2 注。MTS 日志：Mod list 含 `anthonyswrath (0.1.0)`；6 个补丁类 / 10 个补丁处理组全部应用（含 3 个 `@SpirePatch2` 返回值替换：两个 `getCurse` 重载 + `getPurgeableCards`），无本 mod 补丁异常；游戏正常进主菜单 → 建自定义局（`postCreateStartingDeck adding [... anthonyswrath:AnthonysWrath]`）→ 退出时 `Clean shutdown successful`。
>
> **两处修复的目视复核**：用户确认选项名「安东尼之怒」（无「负面：」前缀）与单卡视图大图均正常（T042/T043）；日志中 8 次出现 `Finding texture: anthonyswrath/img/cards/AnthonysWrath_p.jpg` 且无对应 `No image` 告警，佐证 `_p` 大图被实际加载。
>
> **边界**：本次仅完成 T038 与两处目视复核；**T041（QS-01 至 QS-17 完整回归）未被覆盖**——该局仅到建局/查看卡面阶段，未发生战斗、洗牌、移除或卸载动作。

> **T044 牌面文本修订（2026-11-07，用户反馈；实现记录 `memory/log/037`、目视复核 `memory/log/038`）**
>
> 用户 2026-11-07 确认完整回归无功能问题后反馈：牌面描述与参照牌「铃铛的诅咒」完全相同，未记载本牌特有的「每次洗牌后置于抽牌堆底部」与「抽到即判负」两项效果。已修订双语 `cards.json`（spec 层 FR-013 同步校正）：
>
> | 语言 | DESCRIPTION |
> | :--- | :--- |
> | 中文 | 「不能被打出。 NL 不能从牌组中移除。 NL 每次洗牌后置于抽牌堆底部。 NL 抽到这张牌时死亡。」 |
> | 英文 | `Unplayable. NL Cannot be removed from your deck. NL On shuffle, bottom of your draw pile. NL When drawn, you die.` |
>
> **复核结果（2026-11-07，T044 PASS）**：用户重构建后游戏内切换中英查看该牌（总览小卡 + 单卡视图），确认新牌面正常——四项效果可见、无关键词样式、无文字溢出。复核步骤保留如下供复跑：`mvn clean package` → 覆盖 `mods/anthonyswrath.jar` → 进游戏切换中英语言查看该牌：四项效果全部可见、无关键词样式、无文字溢出牌面；key 集合两语言一一对应（离线已核对）。
>
> **离线估算（方法见 `memory/log/037`）**：按本体 `AbstractCard` 的渲染常量（描述字体 24px、英文换行宽 237、中文换行宽 216 = 每行 9 个全角字、牌面坐标宽 300 / 高 420）与 jar 内字体度量，新文案中文约 5 个描述行、英文约 6 个描述行；参照牌旧文案为中文 2 行 / 英文 3 行，本体在超过 5 行时仅在开发模式打印 `has lots of text` 告警。**牌面实际高度/换行须以游戏内目视为准**；若英文行数过高或换行不理想，可再压缩英文措辞（中文为玩家主语言，优先保持中文表述完整）。

| 编号 | 对应需求 | 操作 | 预期 |
| :--- | :--- | :--- | :--- |
| QS-01 | FR-001、FR-014 | 进入自定义模式选项列表 | 可见「安东尼之怒」（不带「负面：」文字前缀）；勾选前即可看到效果说明文本 |
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
| QS-13 | FR-013、SC-005、宪法二 | 切换中英两种语言查看牌面；人工比对两份 `strings/` 的 key 集合 | 描述以纯文本逐项写明四项效果：不能被打出、不能从牌组中移除、每次洗牌后置于抽牌堆底部、抽到即判负；无关键词样式，中英语义一一对应；key 一一对应、无缺无多 |
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

- [x] 构建通过，产物内容符合 §2（2026-11-05 T038）；
- [x] QS-01 至 QS-17 经用户 2026-11-07 完整回归确认通过（用户口述确认「都没有问题」，未逐条回传明细；证据边界见 `memory/log/037`）；
- [x] plan 的 Constitution Check 与规格 checklists 保持全过（2026-11-07 复核）；
- [x] 未选中路径（QS-10）与卸载路径（QS-14）均无回归（在上述用户确认范围内）。

> **2026-11-07 牌面文本修订的复核结论**：T044 已修订双语描述（FR-013 / QS-13 判据同步更新）；用户重构建后游戏内目视确认新牌面正常（四项效果可见、无溢出），**QS-13 新判据 PASS、T044 勾选**，见 `memory/log/038`。全部 QS 判据至此均为通过状态。
