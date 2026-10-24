# Tasks: 安东尼之怒（Anthony's Wrath）

**Input**: Design documents from `/specs/001-anthonys-wrath/`

**Prerequisites**: [plan.md](./plan.md)（必需）、[spec.md](./spec.md)（用户故事）、[research.md](./research.md)、[data-model.md](./data-model.md)、[quickstart.md](./quickstart.md)

**Tests**: 本项目不生成自动化测试任务——宪法第一条禁止第三方依赖，验证方式为游戏内实测 + `quickstart.md` 的可复现步骤。

**Organization**: 任务按用户故事分组，每个故事可独立实现与验证。路径中的 `<modId>` 由 T001 写入顶层 `pom.xml` 后确定，全项目以该配置为唯一命名依据。

**Format**: `[ID] [P?] [Story] Description`；`[P]` = 可并行（不同文件、无未完成依赖）；`[USn]` = 对应 spec 的用户故事。

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: 建立构建配置与可编译骨架，使后续所有任务有落点。

- [x] T001 在顶层建立 `pom.xml`：Java 8 编译目标、显式 `sourceDirectory` 指向 `src/main/java`、`resources` 指向 `src/main/resources`（均在 `src` 子模块内），写入 modId、Java 包名与游戏补丁 / ModTheSpire / BaseMod / StSLib 的版本属性（`<properties>`，版本唯一事实来源）；游戏本体与 ModTheSpire 依赖以 provided / compileOnly scope 声明（平台产物，不计入第三方依赖），路径 `pom.xml`
- [x] T002 [P] 建立顶层 `libs/` 依赖目录说明并校验忽略规则：确认 `.gitignore` 的 `*.jar` 为最终规则、`git ls-files "*.jar"` 输出为空；不引入任何第三方库，路径 `libs/`、`.gitignore`
- [x] T003 建立 mod 入口类 `src/main/java/<modId>/<ModEntry>.java`（BaseMod 初始化与订阅注册骨架），并补齐 jar 根 `src/main/resources/ModTheSpire.json` 的 `modid` / 游戏版本 / `dependencies` 字段（版本取自 `pom.xml` 属性），使构建命令能产出可加载的 mod jar（2026-10-24 验收：普通终端 `mvn package` 成功产出 `target/anthonyswrath.jar`，内含入口类与资源过滤后的清单；ModTheSpire 加载日志 Mod list 出现 `anthonyswrath (0.1.0)` 并完成初始化、游戏正常进主菜单后干净退出，见 `memory/log/015`）

**Checkpoint**: 构建可产 jar，入口类可被 ModTheSpire 加载。（2026-10-24 通过：构建 + 加载双证据见 `memory/log/015`）

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: 在写任何玩法代码前，用探针把实现落点钉死；文本资源骨架同时就绪。

**⚠️ CRITICAL**: 本阶段未完成，任何用户故事都不得进入实现。

- [x] T004 **前置（一次性人工准备）**：ModTheSpire 加载器与 BaseMod、StSLib 已可用（Steam 创意工坊订阅或手动放入游戏目录 `mods/`），且 ModTheSpire 启动日志的 Mod list 中 `basemod` 与 `stslib` 同时出现、BaseMod 完成初始化；安装证据（订阅清单或启动器日志摘录）记入 `memory/log/`，作为本任务可被执行的判据。编写并运行游戏内时序探针 `src/main/java/<modId>/probe/TimingProbe.java`（仅日志输出）：回答 research.md R-004～R-008 与 quickstart §3 的六个问题——首次洗牌与候选钩子的先后、能覆盖首次洗牌与重洗的置底介入点、效果直接取牌与抽牌的判定点是否同一、「轮抽（Draft）」等改写牌组选项与开局牌组创建 / 填充的先后、参照牌「铃铛的诅咒」在变化 / 移除下的实际表现、`PostDungeonInitialize` 是否晚于 `AbstractCardPools` 初始化；并枚举本体让参照牌「删不掉」的全部判定点，回写 research.md §R-008；探针输出与结论追加到 `memory/log/`（2026-10-24：探针代码已就位 `src/main/java/anthonyswrath/probe/`，含 TimingProbe / AwProbeCommand / ProbePatches / ProbeLog，JDK8 `javac` 编译通过；待用户运行采集数据，见 `memory/log/015`）（2026-10-24 第二轮采集：`awprobe.log` 654 行覆盖 4 局 / 6 战；R-004 首洗与耗尽重洗时序、R-005 两个入手判定点、R-006 开局注入时序、R-007 钩子时序均已实测并回写 `research.md`；遗留 R-008 场景 5/6、R-004 非耗尽重洗 / 多副本、R-005 弃牌堆取牌、R-006 Draft 存活待补采；探针已修 `addcard` 控制台补全事故并加入 R-006 标记注入实验，见 `memory/log/018`）（2026-10-24 完成：第二轮 `awprobe.log` 250 行补齐全部缺口——R-004 无参洗牌原语、R-005 弃牌堆取牌（Meditate → `addToHand`）、R-006 哨兵注入确认 Draft 填充后存活（首战 16 张含 `AscendersBane`）、R-007 读档路径复现（池重建而 `PostDungeonInitialize` 不触发）、R-008 商店移除链路（`getPurgeableCards` + `masterDeck.removeCard(obj)`）与变化行为（`CurseOfTheBell` → `Parasite` / `Doubt`）；六问全部有实测答案，净化之泉路径按 quickstart §5 允许跳过、留实现后 QS-11；见 `memory/log/019`）
- [x] T005 [P] 建立双语文本资源骨架 `src/main/resources/strings/<modId>/`：中英 key 一一对应，格式按 BaseMod 实际加载规则在依赖 jar 就绪后核对；本任务只建空骨架与命名约定（2026-10-24 完成：按 BaseMod 5.56.0 `loadCustomStringsFile` 实测规则落为 `strings/anthonyswrath/<lang>/{cards,run_mods}.json`，language 目录取 `Settings.GameLanguage` 小写值、缺语言回退 `eng`；本仓 5.56.0 的 `findCallingModName()` 恒返回 null，故 JSON key 必须写完整 id（卡=完整 cardID，选项=完整 optionId），不做 modId 前缀；命名约定与核对方法见 `ModStrings` javadoc、`research.md` §R-014 与 `memory/log/021`）
- [x] T006 [P] 建立 `src/main/java/<modId>/patches/vanilla/` 包骨架与命名约定，供 US5 逐条实现；不可移除判定点清单由 T004 的枚举结论直接回写 `research.md` §R-008，本任务不重复承接该回写（2026-10-24 完成：`src/main/java/anthonyswrath/patches/vanilla/package-info.java`，一个原版目标一个类、只判定本牌 cardID、不出玩家可见文本；实际补丁由 T019/T024/T029-T031 填充）

**Checkpoint**: 实现落点全部有实测依据，文本与补丁包骨架就绪。（2026-10-24：T004 已完成，R-004～R-008 结论已定稿，六问全部有实测答案，见 `memory/log/018`、`memory/log/019`；T005 / T006 已完成，见 `memory/log/021`；Phase 2 通过）

---

## Phase 3: User Story 1 - 选择选项并在开局牌组中获得诅咒牌 (Priority: P1) 🎯 MVP

**Goal**: 玩家在自定义模式中可见并勾选本选项；开局牌组获得恰好一张同名诅咒牌（含 Draft 等改写后的开局牌组）。

**Independent Test**: 在自定义模式勾选本选项开一局，打开牌组查看界面核对（quickstart.md QS-01 / QS-02 / QS-08）。

### Implementation for User Story 1

> 2026-10-24 实现批次：T007～T014 已完成并通过 JDK8 `javac` 自检（`memory/log/021`）；T015 的实测待普通终端 `mvn package` 后由用户按 QS-01 / QS-02 / QS-08 执行。实现要点：选项以 `<modId>:AnthonysWrathOption` 注册、`isDailyMod=true` 走 `CustomTrial` 通道以写入 `AbstractPlayer.customMods`（新档与读档均可得，见 `research.md` §R-014）；注入由 `PostCreateStartingDeck` 完成（R-006 落点）；本体负面选项只用红色方括号表达「负面」，文本层不含「负面：」字样，故按 US1 验收场景 1 在中文选项名写入该前缀（英文名按规格保持 `Anthony's Wrath`），结论见 `memory/log/021`。

- [x] T007 [P] [US1] 实现自定义模式选项类 `src/main/java/<modId>/options/AnthonysWrathOption.java`：稳定 `optionId` 形如 `<modId>:AnthonysWrathOption`，显示名与说明文本取自 `strings/`，不写死字符串（2026-10-24 完成：`OPTION_ID = "anthonyswrath:AnthonysWrathOption"`，`create()` 返回 `new CustomMod(OPTION_ID, "r", true)`；名称/说明由 `CustomMod` 构造器按 `OPTION_ID` 从 `RunModStrings` 读取）
- [x] T008 [US1] 在入口类注册该选项（注册接口名以 BaseMod jar 实物为准，并回写 research.md §R-003），确认标识不与本体选项冲突，同名第三方选项通过标识区分、互不覆盖（`EC-007`）（2026-10-24 完成：`AnthonysWrathMod implements AddCustomModeModsSubscriber.receiveCustomModeMods(List<CustomMod>)`；标识带 modId 命名空间且不入 `getActiveNonDailyMods`，不与本体选项同名冲突；R-003 已回写）
- [x] T009 [P] [US1] 在双语资源写入选项显示名「安东尼之怒」与勾选前可见的效果说明文本（`FR-014`），路径 `src/main/resources/strings/<modId>/`；并比对本体负面选项的前缀规则，确认「负面：」前缀由界面槽位提供还是需写入文本、不得重复，结论记入 `memory/log/`（2026-10-24 完成：`zhs/run_mods.json` 名称「负面：安东尼之怒」、`eng/run_mods.json` 名称 `Anthony's Wrath`，两语言均带勾选前可见的说明；比对结论：本体负面选项仅用红色方括号（`NEGATIVE_COLOR = "r"`）表达「负面」、`CustomModeScreen` 与本地化文本都不提供「负面：」字样，故按 US1 验收场景 1 写入中文名，不存在重复前缀；见 `memory/log/021`）
- [x] T010 [P] [US1] 实现诅咒牌类 `src/main/java/<modId>/cards/AnthonysWrathCurse.java`：`CardType.CURSE`、`CardColor.CURSE`、费用 -2、`CardRarity.SPECIAL`、不可升级、无使用效果、`makeCopy()` 返回新实例（`FR-004`、`FR-005`）（2026-10-24 完成：参数与参照牌一致；`upgrade()` 空实现 + CURSE 类型使 `canUpgrade()` 恒 false；`makeCopy()` 返回 `new AnthonysWrathCurse()`）
- [x] T011 [US1] 在 `EditCardsSubscriber.receiveEditCards` 中注册诅咒牌并确认游戏将其识别为诅咒（`FR-004`），路径 `src/main/java/<modId>/<ModEntry>.java`（2026-10-24 完成：`receiveEditCards` → `BaseMod.addCard(new AnthonysWrathCurse())`；`CardColor.CURSE` 由 BaseMod 归入 `curseToAdd`，经 `CardLibrary` 注册，卡牌类别为 `CURSE`）
- [x] T012 [US1] 实现开局注入 `src/main/java/<modId>/run/StartDeckInjector.java`：实现 `PostCreateStartingDeckSubscriber`，基于 `optionId` 判定本局是否选中，选中则加入恰好一张；注入点在牌组创建之后、首次洗牌之前，具体先后按 T004 结论执行（`FR-002`、`FR-003`；覆盖 `EC-002`：Draft 等改写后的开局牌组；R-006 哨兵实验已确认该钩子的注入牌在 Draft 填充后存活且早于首洗） （2026-10-24 完成：`StartDeckInjector.injectIfSelected` 在 `PostCreateStartingDeck` 追加一张，`alreadyContainsCurse` 防重复；新档判定为 `CardCrawlGame.trial instanceof CustomTrial` + `AbstractPlayer.customMods` 含标识；读档不触发该钩子故不会重复注入）
- [x] T013 [P] [US1] 在双语资源写入诅咒牌名称与描述初版（名称：中文「安东尼之怒」/ 英文 `Anthony's Wrath`），路径 `src/main/resources/strings/<modId>/`；描述定稿由 T033 按参照牌措辞覆盖完成（2026-10-24 完成：`cards.json` 初版，描述按参照牌 `CurseOfTheBell` 措辞写作「不能被打出。 NL 不能从牌组中移除。」/ `Unplayable. NL Cannot be removed from your deck.`；T033 定稿）
- [x] T014 [US1] 选项标识容错（路线 a）：从存档恢复后的自定义模式项列表读取时忽略未知标识，保证本 mod 加载时对其它未知项健壮；EC-006 的「本 mod 未加载」场景由本体 / BaseMod 负责、QS-14 只作验证（实测崩溃则回 `/speckit-clarify`），路径 `src/main/java/<modId>/options/`（2026-10-24 完成：判定集中在 `run/RunState.isOptionSelected()`，只做 `contains(optionId)` 且对 `null` 列表返回 false，未知标识天然被忽略；`customMods` 为静态字段，另以 `Settings.isTrial || trial instanceof CustomTrial` 排除标准局残留）
- [x] T015 [US1] 实测并记录 US1：选项可见 / 显示名 / 效果说明（QS-01，`FR-014`）、开局牌组恰好一张（QS-02）、改写开局牌组注入（QS-08，Draft 场景，仅验加入与时机）、未加载本 mod 后读档不崩溃（QS-14）；结果追加到 `memory/log/`（2026-10-24 验收：QS-01 / QS-02 / QS-08 通过——两局为铁甲战士普通自定义局与观者 + Draft 局；`awprobe.log` A#4 证明选中态可读、A#8→A#9 注入前 deck=10、注入后恰好 1 张且 A#10~A#13 保持 1 张；B#41 选中态含 Draft、B#45 空牌组、B#46 注入 1 张、B#50 Draft 填充 15 张后 masterDeck=16 仍含本牌、B#52 首洗晚于注入约 12 s；QS-01 另由游戏日志 RunModStrings/CardStrings 无缺键报错与 BaseMod `adding [ … ]` 记录佐证，中文文案目视复核留 US2 首轮顺手完成；**QS-14 未执行**，按 R-012 列入 T041 完整回归必做项；详见 `memory/log/022`）

**Checkpoint**: 用户故事 1 可独立验证——选项可见、选中后牌组含牌、卸载容错。（2026-10-24 通过：QS-01 / QS-02 / QS-08 实测通过，证据 `memory/log/022`；QS-14 卸载容错未执行，已列入 T041 必做项）

---

## Phase 4: User Story 2 - 每次洗牌后诅咒牌位于抽牌堆底部 (Priority: P2)

**Goal**: 战斗开始洗牌、抽牌堆耗尽重洗、以及卡牌 / 遗物触发的非耗尽洗牌后，该牌都在抽牌堆最底部；多副本成组且相对顺序不变。

**Independent Test**: 一场战斗中按顺序查看抽牌堆（「冰冻之眼」优先），覆盖三次洗牌路径（quickstart.md QS-03 / QS-04 / QS-05）。

### Implementation for User Story 2

> 2026-10-24 实现批次：T016～T019 已完成并通过 JDK8 `javac` 全量编译与离线补丁目标自检（18/18，见 `memory/log/023`）；**T020 已于 2026-10-24 21:22–21:26 实测通过**（`awprobe.log` 235 行，判读记录 `memory/log/026`）。实现要点：选 R-004 的补丁路线——两个 `CardGroup.shuffle` 原语各挂 postfix，按 `DRAW_PILE` / `DISCARD_PILE` 类型过滤后把本牌移到被洗组索引 0 端；多副本按洗后相对顺序成组置底，缺席直接跳过。注意：20:0x 曾把「抽牌堆耗尽重洗」记为成品不可达，已被 T020 实测证伪（Scry 类效果提供不入手的抽牌堆→弃牌堆通道），spec / quickstart / research 已按二次修正回写。

- [x] T016 [US2] 实现置底逻辑 `src/main/java/<modId>/run/DrawPileOrdering.java`：按 R-004 在两个 `CardGroup.shuffle` 原语（`shuffle(Random)` 与 `shuffle()`）的 postfix 上介入，把本牌移到被洗组索引 0 端（即抽牌堆底部）（`FR-006`）（2026-10-24 完成：`DrawPileOrdering.sinkCurseToBottom(CardGroup)` 只处理 `DRAW_PILE` / `DISCARD_PILE` 类型（战斗开始的临时组由 `initializeDeck` 构造为 `DRAW_PILE`）并附 `player.drawPile` / `player.discardPile` 身份兜底；索引 0 = 底部经字节码复核——`initializeDeck` 与 `EmptyDeckShuffleAction`→`Soul.shuffle` 两条搬运路径都按索引 0→尾 `drawPile.addToTop`；JDK8 `javac` 通过，见 `memory/log/023`）
- [x] T017 [US2] 在入口类接入该逻辑，覆盖战斗开始首次洗牌、抽牌堆耗尽重洗与卡牌 / 遗物触发的非耗尽重洗（`FR-006`、`EC-001`）（2026-10-24 完成：按 R-004 选补丁路线，接入点即 T019 的 `@SpirePatch` 类——ModTheSpire 加载时扫描 jar 内 `@SpirePatch` 自动应用，无需在入口类注册订阅（与 T004 探针补丁同机制）；三条路径收敛于两个原语：战斗开始 `initializeDeck` 洗临时 `DRAW_PILE` 组、耗尽重洗 `EmptyDeckShuffleAction` 洗 `discardPile`、牌触发 `ShuffleAction`（`Reboot` / `DeepBreath`）洗 `drawPile` 与 `ShuffleAllAction` 洗 `discardPile`。V2.3.4 全 jar 静态清点仅 8 处 `CardGroup.shuffle` 调用点，其余 4 处洗 `UNSPECIFIED` 暂存组（已被类型过滤跳过）；无直接对牌堆 `group` 的 `Collections.shuffle`。说明写在入口类 javadoc，证据见 `memory/log/023`）
- [x] T018 [US2] 处理多副本：全部副本成组置于底部且相对顺序不变（`FR-007`），并在找不到该牌时直接跳过、不做额外判定（`EC-003`）（2026-10-24 完成：按洗后相对顺序收成连续块放到索引 `0..k-1`；组内无本牌时返回 false 且不改动（EC-003）；确定性自检命令 `awprobe sinktest`（4 用例：成组/幂等/缺席/非牌堆组跳过，含 uuid 序核对）随 T020 执行）
- [x] T019 [US2] 若介入点落在原版方法上，则把对应补丁放入 `src/main/java/<modId>/patches/vanilla/`，只判定本牌 cardID（2026-10-24 完成：`patches/vanilla/CardGroupShufflePatch.java`，两个内部类分别 postfix `shuffle()`（`paramtypez = {}`）与 `shuffle(Random)`，只调用 `DrawPileOrdering` 判定本牌 cardID；离线复刻 MTS 目标解析自检 18/18 OK——含两条新补丁精确命中对应重载，`memory/log/023`）
- [x] T020 [US2] 实测并记录 US2：首次洗牌置底（QS-03，含 Draft 空开局牌组）、两次以上重洗（含非耗尽，QS-04）、多副本成组（QS-05）、第二场战斗重新置底（QS-17），覆盖 `SC-001`、`SC-002`；结果追加到 `memory/log/`（2026-10-24 21:22–21:26 探针版实测**全部通过**，`awprobe.log` 235 行、判读见 `memory/log/026`：QS-03 #10~#14/#38；QS-04 非耗尽重洗两次 #42~#45、#151~#154，**耗尽重洗也实测通过** #106~#129（经 Scry 先把本牌放入弃牌堆，本牌全程无入手事件）；QS-05 #163~#169 两张副本在 drawPile 0/1 + `awprobe sinktest` 4/4 PASS；QS-17 #169/#193 第二场仍在索引 0。**范围二次修正**：耗尽分支在成品中可达但依赖 Scry 类效果，`spec.md` / `quickstart.md` / `research.md` §R-004 已回写，`memory/facts/f018` 标 stale、`f019` 记录新结论。另 #159/#160 净化之泉删除本牌属 US5（T029-T031）未实现的预期缺口；QS-01 屏幕文案仍无记录，留 T041）

**Checkpoint**: 用户故事 2 可独立验证——三条洗牌路径后该牌均在底部。（**2026-10-24 通过**：T016~T019 实现与离线自检见 `memory/log/023`；T020 实测三条路径全部通过、多副本成组并经确定性自检，见 `memory/log/026`）

---

## Phase 5: User Story 3 - 抽到诅咒牌立即判负 (Priority: P3)

**Goal**: 该牌被抽入手中（或经效果直接取得）时本局立即判负并计入失败记录；只结算一次，之后不再处理抽牌与操作。

**Independent Test**: 等待自然抽到该牌、再用「搜寻」类效果取牌各一次，核对结算界面与游戏记录（quickstart.md QS-06 / QS-07 / QS-09 / QS-15）。

### Implementation for User Story 3

> 2026-10-24 实现批次：T021～T024 已完成并通过 JDK8 `javac` 全量编译、离线 MTS 补丁目标自检 22/22 与一次性守卫离线测试 8/8（`memory/log/027`）；**T025 游戏内实测待用户执行**（步骤见 `quickstart.md` §4 的 2026-10-24 US3 段）。实现要点：判负落点复用本体失败流程——按 `AbstractPlayer.damage` 字节码的死亡分支（V2.3.4 偏移 1102~1136）执行 `isDead=true` → `AbstractDungeon.deathScreen = new DeathScreen(getMonsters())` → `currentHealth=0` → `loseBlock()`，不调用 `damage()`（否则蜥蜴尾 / 妖精药水等复活资源会拦下判负，违反 FR-008）；两个入手判定点落 `patches/vanilla/CardGroupHandEntryPatch`（`addToHand` postfix + 目标为手牌的 `addToTop` postfix，R-005 / f013）；一次性守卫 `RunState.tryMarkDefeated()`，终局短路为 `patches/vanilla/AbstractPlayerDrawPatch`（`draw(int)` prefix 在判负后整段跳过，EC-004）；`StartGame` 订阅只负责每局起点重置运行期标志（T027 将在此钩子追加诅咒池排除）。

- [x] T021 [US3] 实现入手判定 `src/main/java/<modId>/run/DefeatOnDraw.java`：在取得动作完成时立即触发本局失败，走游戏本体失败流程（`FR-008`、`FR-009`）（2026-10-24 完成：`DefeatOnDraw.onCurseEnteredHand(AbstractCard)` 只判定本牌 cardID；`RunState.tryMarkDefeated()` 置一次性守卫后，按本体 `AbstractPlayer.damage` 的死亡分支执行 `player.isDead=true` → `new DeathScreen(AbstractDungeon.getMonsters())` → `currentHealth=0` → `loseBlock()`；`DeathScreen` 构造器承担 screen=DEATH、角色失败统计、失败 metrics、存档处理与返回主菜单按钮，FR-009 的可见失败结算与失败记录均走本体流程。**否决** `player.damage(...)`：同一方法的复活分支（Lizard Tail / Fairy Potion / Mark of the Bloom，偏移 960~1101）会把致命伤变成回复，判负将不成立。判负落点与选择理由见 `research.md` §R-005 Implemented 与 `memory/log/027`）
- [x] T022 [US3] 覆盖「效果直接取牌」路径（`EC-008`）：按 R-005 实测实现两个判定点（`CardGroup.addToHand` 与目标为手牌的 `CardGroup.addToTop`）；抽牌堆检索走后者、弃牌堆取牌（Meditate）走前者，两判定点已覆盖全部实测通道（`memory/facts/f013`）（2026-10-24 完成：`patches/vanilla/CardGroupHandEntryPatch` 两个 postfix 静态内部类；`addToHand` 覆盖普通抽牌 / 临时牌 / 弃牌堆取牌，`addToTop` 仅在 `__instance == player.hand` 时判定，覆盖抽牌堆检索与 Retain；两处均只把命中的本牌交给 `DefeatOnDraw`，其它卡牌行为不变。离线自检确认两个补丁精确命中目标方法）
- [x] T023 [US3] 加入本局一次性守卫与终局短路：失败只结算一次（`FR-010`），判负后后续抽牌在入口整段跳过、不接受可操作结果（`EC-004`）（2026-10-24 完成：`RunState` 新增 `defeated` / `defeatGuard` 与 `isDefeated()` / `tryMarkDefeated()` / `resetDefeatState()`；多副本同时入手时第二次判定直接返回，只结算一次；`patches/vanilla/AbstractPlayerDrawPatch` 以 `SpireReturn` prefix 在 `isDefeated()` 时跳过 `AbstractPlayer.draw(int)` 整个方法体（后续抽牌入口级短路）；判负后本体 `screen=DEATH` 只更新 `DeathScreen`、不再更新战斗与操作，玩家无法再操作。守卫语义离线测试 8/8 通过）
- [x] T024 [US3] 在入口类接入判定组件；若判定点需拦截原版抽牌动作，补丁放入 `patches/vanilla/`，否则优先使用 BaseMod 既有钩子（2026-10-24 完成：判定点与短路均为原版方法拦截，按约定放 `patches/vanilla/`（`CardGroupHandEntryPatch` / `AbstractPlayerDrawPatch`），由 ModTheSpire 加载时自动应用，入口类不额外注册；入口类唯一的 US3 变更是 `implements StartGameSubscriber` 并在 `receiveStartGame()` 调 `RunState.resetDefeatState()`——新档与读档都触发该钩子（R-007），供 T027 在此追加诅咒池排除；javadoc 已写明该分工）
- [ ] T025 [US3] 实测并记录 US3：自然抽到即负（QS-06，`SC-003`）、搜寻取牌判负（QS-07）、多副本只结算一次与终局短路（QS-09）、失败记录（QS-15）、主动放弃不触发特判（QS-16，`EC-005`）；结果追加到 `memory/log/`（2026-10-24：**待用户实测**，jar 已由用户重建并放入 `mods/`；步骤与预期见 `quickstart.md` §4 的 2026-10-24 US3 段，证据判读用 `awprobe.log` 新增的 `patch.deathScreen.*` 两条日志）


**Checkpoint**: 用户故事 3 可独立验证——抽到即负、只结算一次、失败入记录。（2026-10-24：T021~T024 实现完成，`memory/log/027`；T025 待用户实测，通过后本 Checkpoint 才算成立）

---

## Phase 6: User Story 4 - 未选择选项时无任何影响 (Priority: P4)

**Goal**: 未勾选本选项时，牌组、抽牌堆与随机诅咒来源都不出现该牌；标准模式与每日挑战同样不受影响。

**Independent Test**: 不勾选开三局（标准 / 每日 / 自定义），逐张比对牌组，并用事件 / 遗物反复产出诅咒观察（quickstart.md QS-10）。

### Implementation for User Story 4

- [ ] T026 [US4] 让所有注入 / 置底 / 判负组件在未选中时短路：统一以存档恢复后的自定义模式项列表 + `optionId` 为准（`FR-011`），路径 `src/main/java/<modId>/run/`
- [ ] T027 [US4] 把本牌排除出随机诅咒池：在 `StartGameSubscriber.receiveStartGame` 中移出（R-007 实测：新档与读档都触发且晚于 `initializeCardPools`；`PostDungeonInitialize` 读档不触发，不可作唯一落点），钩子不可用时改为 `patches/vanilla/` 补丁；记录所选落点（`EC-010`）
- [ ] T028 [US4] 实测并记录 US4：三种模式未选中时牌组逐张一致、随机诅咒来源不发出本牌（QS-10，`SC-004`）；结果追加到 `memory/log/`

**Checkpoint**: 用户故事 4 可独立验证——未选中路径零扰动。

---

## Phase 7: User Story 5 - 诅咒牌无法从牌组中移除 (Priority: P5)

**Goal**: 商店移除、休息点移除、净化之泉与各类移除事件都无法删掉该牌；牌面按参照牌措辞写明不可移除，中英双语一致。

**Independent Test**: 依次进入三类移除场景，并切换中英语言目视核对牌面（quickstart.md QS-11 / QS-12 / QS-13）。

### Implementation for User Story 5

- [ ] T029 [US5] 实现不可移除拦截 ①：在 `patches/vanilla/` 中过滤 `CardGroup.getPurgeableCards()` 的返回结果，覆盖商店移除、休息点、遗物与事件候选列表（`FR-012`、`EC-009`）
- [ ] T030 [US5] 实现不可移除拦截 ②：令净化之泉 `FountainOfCurseRemoval` 的批量删除循环跳过本牌，路径 `src/main/java/<modId>/patches/vanilla/`
- [ ] T031 [US5] 按 T004 枚举出的其余移除路径逐条补齐拦截，只判定本牌 cardID，不改变原版对其它卡牌的处理
- [ ] T032 [US5] 对齐「变化」与「消耗类效果」行为（路线 b）：在探针版临时关闭「抽到即负」后验证手牌中不可打出、不可被主动消耗，并与参照牌同条件对照（含「蓝蜡烛」一次），不增加限制也不放宽（`EC-003`、`EC-011`）；开关仅开发期存在、随 T035 移除，发布版不含
- [ ] T033 [P] [US5] 在双语资源定稿牌面描述（覆盖 T013 初版）：按参照牌「铃铛的诅咒」措辞直接写明「不能被打出」「不能从牌组中移除」，不使用关键词样式或颜色标记（`FR-013`）
- [ ] T034 [US5] 实测并记录 US5：三类移除场景（QS-11）、变化 / 蓝蜡烛对照（QS-12，探针版临时关闭判负）、中英牌面文本与 key 一一对应（QS-13），覆盖 `SC-005`；结果追加到 `memory/log/`

**Checkpoint**: 用户故事 5 可独立验证——所有移除入口都拒绝该牌，文本与参照牌一致。

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: 发布前清理、跨故事验收与三层提交规范。

- [ ] T035 [P] 移除探针代码 `src/main/java/<modId>/probe/`（含临时关闭「抽到即负」的验证开关），确认发布版本不包含调试输出与任何调试开关（探针结论已入 `memory/log/`）
- [ ] T036 [P] 替换诅咒牌占位贴图，并在 `src/` 层建立自己的 `.gitignore` 忽略 `placeholder/`；确认 `git -C src ls-files` 不含占位图
- [ ] T037 [P] 人工比对中英双语资源文件的 key 集合（无缺无多），并核对代码中无玩家可见字符串字面量（宪法第二条）；两份 key 清单与结果写入 `memory/log/`
- [ ] T038 构建与加载验收：构建命令成功，`jar tf` 确认产物内含双语资源、贴图与 jar 根 `ModTheSpire.json`，mod 能被 ModTheSpire 正常加载且游戏可启动、日志无异常
- [ ] T039 验证记录机制定稿：确认「流程 / 记录 / 结论」三级在实战中够用，边界不清处就地修正；记录追加到 `memory/log/`
- [ ] T040 里程碑校验三层锁定与提交顺序（宪法第七、九条）：三段 gitlink 与各层 HEAD 一一对应、`.gitmodules` 地址可解析；提交顺序为先层后顶层
- [ ] T041 按 [quickstart.md](./quickstart.md) 完成一次完整回归（QS-01 至 QS-17，覆盖 `SC-001` 至 `SC-005`），逐条记录结果到 `memory/log/`，作为宪法第五条「完成标准」的证据（**QS-14 卸载 / 禁用后读旧档为必做项**：T015 未执行，发布前不得跳过；QS-01/02/08 已于 2026-10-24 通过，见 `memory/log/022`）

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: 无依赖，可立即开始
- **Foundational (Phase 2)**: 依赖 Phase 1；阻塞全部用户故事（T004 探针结论决定实现落点）
- **运行期前置（跨故事）**: T004 及 T015 / T020 / T025 / T028 / T034 / T038 / T041 等一切游戏内实测任务，均以 plan 的「运行期前置」完成（加载器与 BaseMod / StSLib 可用，且启动日志 Mod list 同时列出 `basemod` 与 `stslib`）为前提；前置未完成时不得开始，也不得标记完成。
- **User Stories (Phase 3–7)**: 均依赖 Phase 2；按 P1 → P2 → P3 → P4 → P5 顺序推进，也可在人员充足时并行（除 T004 探针为共享前置）
- **Polish (Phase 8)**: 依赖全部用户故事完成

### User Story Dependencies

- **US1 (P1)**: Phase 2 后可开始，无其它故事依赖（MVP）
- **US2 (P2)**: Phase 2 后可开始；依赖 US1 已注册的诅咒牌
- **US3 (P3)**: Phase 2 后可开始；依赖 US1 的诅咒牌与 US2 的置底（否则无法稳定等到该牌被抽到）
- **US4 (P4)**: Phase 2 后可开始；与其它故事无代码依赖，但需在 US1 注册完成后验证
- **US5 (P5)**: Phase 2 后可开始；依赖 US1 的卡牌注册与 T004 的移除路径枚举

### Within Each User Story

- 数据模型 / 组件先于集成；先实现再实测；每个故事以 `memory/log/` 记录收尾，故事完成后再进入下一优先级。

### Parallel Opportunities

- Phase 1 的 T002 可与 T001 并行
- Phase 2 的 T005、T006 可与 T004 并行：两者只建骨架，探针结论（含 §R-008 判定点清单）由 T004 回写，不阻塞骨架建立
- US1 中 T007、T009、T010、T013 可并行（不同文件）
- US5 中 T033 可与 T029–T032 并行（资源文件与代码文件分离）
- Polish 中 T035、T036、T037 可并行

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. 完成 Phase 1 + Phase 2（探针是一切实现的前置）
2. 完成 Phase 3（US1）
3. **停下验证**：按 QS-01 / QS-02 / QS-08 独立验收 US1
4. 再按优先级依次交付 US2 → US3 → US4 → US5

### Incremental Delivery

- 每个用户故事完成即实测并记录，不把验证积压到 Phase 8；
- Phase 8 的完整回归（T041）只做一次，作为发布前证据；
- 任何探针结论变化都可能影响多个故事，须先回写 `research.md` 再改实现。

---

## Notes

- `[P]` 任务 = 不同文件且无未完成依赖
- 涉及原版拦截的任务（T019、T024、T027、T029–T031）一律落在 `patches/vanilla/`，且只判定本牌
- 任务编号即执行顺序建议；跨故事依赖以「User Story Dependencies」为准
- 每个任务完成后可按逻辑分组提交；涉及三层的提交遵守「先层、后顶层」的顺序
