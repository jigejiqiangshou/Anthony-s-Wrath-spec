# Research — 安东尼之怒（Anthony's Wrath）

**Feature**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md) | **Date**: 2026-10-23

本文件记录实现前需要定型的技术决策、依据与替代方案。凡标注「待探针」的项，均在实现前由游戏内时序探针实测确定，**不得**在未实测的情况下按猜测实现。

## R-001 运行环境与工具链

- **Decision**: Java 8 + Maven（编译与运行目标均为 Java 8）；构建命令在顶层目录执行，产物为单个 mod jar。
- **Rationale**: 游戏运行时为 Java 8；沿用仓库已就绪的工具链，不引入 Gradle 等第二套构建工具。
- **Alternatives considered**: Gradle（否决：需重配全部现有约定，收益为零）；更高 Java 版本（否决：与游戏运行时不一致）。

## R-002 构建与依赖布局

- **Decision**: 顶层 `pom.xml` 显式声明 `sourceDirectory` 指向 `src/main/java`、`resources` 指向 `src/main/resources`（两者位于 `src/` 子模块内）；游戏本体、ModTheSpire、BaseMod、StSLib 的 jar 只放顶层 `libs/`，`*.jar` 一律不入库；版本键写入 `pom.xml` 的 `<properties>` 作为唯一事实来源。其中游戏本体与 ModTheSpire 以 provided / compileOnly 方式仅作平台产物参与编译，不视为第三方依赖。
- **Rationale**: 满足宪法第八条（构建配置归属顶层）与宪法引言（版本可独立查得），同时让 `.gitignore` 的 `*.jar` 规则保持最终效力，不需要白名单例外。
- **Alternatives considered**: 把构建配置放进 `src/` 子模块（否决：违反宪法第八条）；在代码里硬编码版本号（否决：违反宪法引言）。

## R-003 扩展入口（BaseMod 订阅者）

- **Decision**: 入口类实现并订阅：`EditCardsSubscriber.receiveEditCards`（注册诅咒牌）、`EditStringsSubscriber.receiveEditStrings`（加载双语文本）、`AddCustomModeModsSubscriber.receiveCustomModeMods(List<CustomMod>)`（注册自定义模式选项）、`PostCreateStartingDeckSubscriber.receivePostCreateStartingDeck(PlayerClass, CardGroup)`（开局注入）、`StartGameSubscriber.receiveStartGame`（随机诅咒池排除；读档路径也触发，见 R-007）；初始化阶段完成 `BaseMod.subscribe(this)`。
- **Rationale**: 全部走 BaseMod 公开订阅点，避免不必要的原版补丁；只有订阅点覆盖不到的判定才进 `patches/vanilla/`。
- **Alternatives considered**: 全部用补丁实现（否决：补丁面积大、升级游戏版本时易碎）。
- **Verified（2026-10-24，实物核对）**: `libs/BaseMod.jar` 内实有 `basemod/interfaces/AddCustomModeModsSubscriber.class`，方法为 `receiveCustomModeMods(java.util.List<com.megacrit.cardcrawl.screens.custom.CustomMod>)`；对应发布端 `BaseMod.publishAddCustomModeMods(List<CustomMod>)`。旧文档中的 `CustomModeMods` / `receiveAddCustomModeMods` 不存在，作废。方法：`jar tf` + `javap`，证据见 `memory/facts/f009` 与 `memory/log/014`。

## R-004 洗牌后置底介入点（2026-10-24 探针实测；剩余项见末）

- **Decision**: 在洗牌原语上做后置介入：`patches/vanilla/` 对 `CardGroup.shuffle(Random)` 与 `CardGroup.shuffle()` 各挂一个 postfix；当被洗的组是抽牌堆 / 弃牌堆（战斗开始时是 `initializeDeck` 内部的临时 DRAW_PILE 组）且组内存在本牌时，把本牌移到**该组索引 0 端**。实测两条洗牌路径都从洗后组索引 0 开始消费，因此这一处理最终落在抽牌堆底部。`StartBattle` / `OnPlayerTurnStart` 只覆盖首次洗牌，不能作为唯一介入点。
- **Rationale**: 规格要求「每次洗牌后」都在底部（FR-006/007、EC-001）；洗牌后牌序已定，此时移动不依赖后续搬运实现，且两个洗牌原语是全部路径的公共汇合点。
- **Verified（awprobe.log 654 行；证据 `memory/log/018`，抽牌方向与常量见 `memory/facts/f010`）**:
  - 首次洗牌 6 次：`CardGroup.shuffle <- initializeDeck:1031 <- preBattlePrep:1982`，作用对象是 `initializeDeck` 新建的临时 DRAW_PILE 组（`groupTag` 显示 `other(type=DRAW_PILE)`），此时 `player.drawPile` 仍为空；`initializeDeck` 之后按洗后顺序把牌填入抽牌堆。
  - 耗尽重洗 4 次：`CardGroup.shuffle <- EmptyDeckShuffleAction.update:49`，作用对象是 `player.discardPile`；该 action 每次 `update()` 只搬 1 张，搬入顺序 = 洗后弃牌堆索引 0→尾，抽牌堆最终顺序与洗后弃牌堆逐张一致（两次独立核对）。
  - 钩子时序：`StartBattle` 在首洗之后、起手摸牌之前（早 0.8–0.9 s），`PlayerTurnStart` 紧随其后；中途重洗发生在 `PlayerTurnStart` 之后数秒，故两个钩子都不覆盖。
  - 抽牌方向：`getTopCard()` 返回 `group.get(size-1)`，探针 10/10 实测「dump 末位先被抽走」⇒ **drawPile 索引 0 = 底部（最后抽到），末位 = 下一次抽到**；`ProbeLog.ids()` 原注释「index 0 = top」已修正。
- **Remaining（补采前 T016 不得定稿）**: ①非耗尽重洗（卡牌 / 遗物触发）本会话 0 次；无参 `CardGroup.shuffle()` 本会话 0 次，仅 `awprobe shuffledeck` 可直接造出；②多副本成组置底与相对顺序（FR-007、QS-05）无场景；③端到端置底断言要等本牌注册后（QS-03/QS-04）。
- **Alternatives considered**: 只在 `StartBattle` 置底（否决：不覆盖中途重洗，实测重洗发生在该钩子之后）；只挂 `shuffle(Random)`（否决：无参重载是独立实现而非转调，必须两处都挂）；在每次抽牌前动态校正作为唯一手段（否决：不满足「洗牌后即在底部」的可观察性）。

## R-005 入手判定点（2026-10-24 探针实测；弃牌堆取牌待补）

- **Decision**: 两个判定点，不做「共用单一判定点」：① `CardGroup.addToHand` postfix（普通抽牌与临时牌入手）；② `CardGroup.addToTop` postfix 且目标组是 `player.hand`（效果直接取牌、Retain 回手）。两处都按 cardID 命中本牌即判负，并置本局一次性守卫。`PostDrawSubscriber` 只覆盖普通抽牌，不能作为唯一判定点。
- **Rationale**: FR-008、EC-008 要求「效果直接取牌」同样立即判负；EC-004 与 FR-010 要求只结算一次、后续不再处理抽牌。
- **Verified（awprobe.log；证据 `memory/log/018`）**:
  - 普通抽牌 80 次全部走 `addToHand <- AbstractPlayer.draw:2071 <- draw:2098 <- DrawCardAction.update:146`，并触发 `PostDraw`；
  - 临时牌 6 次走 `addToHand <- ShowCardAndAddToHandEffect.<init>:67 <- MakeTempCardInHandAction.addToHand:102`，不触发 `PostDraw`；
  - 「搜寻」取牌 2 次只走 `removeCard(drawPile) <- SkillFromDeckToHandAction.update:54/:78` + `addToTop(hand) <- ...:55/:79`，**完全绕过 addToHand、PostDraw 与 draw(int)**；
  - Retain 回手 10 次走 `addToTop(hand) <- RestoreRetainedCardsAction.update:26`；判定点必须按 cardID 过滤，避免把普通回手当成本牌入手。
- **Remaining**: 弃牌堆→手中的取牌路径未实测（预期同走 `addToTop`，需补采确认）；判负落点（`AbstractDungeon.fail` / 结算界面）与「只结算一次」守卫属 QS-06/QS-09，不属本探针。
- **Alternatives considered**: 只挂 `PostDrawSubscriber`（否决：取牌不触发）；只挂 `addToHand`（否决：EC-008 通道绕过）；回合结束扫描手牌（否决：不满足「同一瞬间结束」）。

## R-006 开局注入时机（2026-10-24 探针实测；Draft 存活实验待补）

- **Decision**: 仍定在 `PostCreateStartingDeckSubscriber.receivePostCreateStartingDeck`；但 T012 定稿前必须先用探针的标记注入实验确认「注入的牌不会被随后的 Draft 填充清除」。若被清除，备选落点为 `preBattlePrep` 前缀 + 一次性守卫（同样需要实测）。
- **Rationale**: FR-003、EC-002；不实测无法判断注入牌在「空牌组 → 轮抽填充」过程中的存活，不能按猜测实现。
- **Verified（awprobe.log；证据 `memory/log/018`）**:
  - Draft 局 `initializeStarterDeck` 前后、`PostCreateStartingDeck`、`StartGame`、`StartAct`、`PostDungeonInitialize` 看到的 `masterDeck` 全部为 0（`PostCreateStartingDeck` 收到的是空牌组）；
  - 填充发生在 `PostDungeonInitialize`（15:52:31.962）之后、首战 `preBattlePrep`（15:52:43.257）之前，窗口 11.295 s；首战时牌组 15 张且全部是 Draft 牌（无普通打击/防御）；
  - `PostCreateStartingDeck` 比首次洗牌早约 11.3 s，「注入早于第一次洗牌」成立（FR-003）；
  - 普通对照局 `PostCreateStartingDeck` 即收到 10 张标准牌组；读档路径不触发 `initializeStarterDeck` / `PostCreateStartingDeck`（牌组来自存档 ⇒ 注入器不会重复加牌，对 FR-002 有利）。
- **Remaining**: 注入牌的 Draft 存活 —— 在 `%USERPROFILE%\awprobe.inject` 哨兵文件存在时，`TimingProbe` 在 `PostCreateStartingDeck` 注入一张 AscendersBane 并记日志；下一轮看 `StartGame` / `PostDungeonInitialize` / 首战 `preBattlePrep` 的 masterDeck 是否仍在。
- **Alternatives considered**: 在第一次战斗开始时补齐注入（否决：会被存档与开局校验遗漏）；`preBattlePrep` 前缀（保留为存活失败时的备选）。

## R-007 随机诅咒池排除（2026-10-24 探针实测；落点改为 StartGame 钩子）

- **Decision**: 用 `StartGameSubscriber.receiveStartGame` 把本牌从 `AbstractDungeon.curseCardPool` 移出：实测新档与读档都触发该钩子，且都在 `initializeCardPools()` 重建池之后。`PostDungeonInitializeSubscriber` **不能**作为唯一落点（读档不触发）。若后续版本观测到 StartGame 时点变化，退回 `patches/vanilla/` 对 `AbstractDungeon.initializeCardPools()` 打 postfix（其内部调用 `addCurseCards()` 填充池）。
- **Rationale**: 原方案只用 PostDungeonInitialize，读档路径会让本牌重新进入随机诅咒池，违反 FR-011、EC-010。
- **Verified（awprobe.log；证据 `memory/log/018`，反汇编见 `memory/facts/f011`）**:
  - 4 次启动（新档 2、读档 2）全部执行 `initializeCardPools`，`curseCardPool` 稳定为 10 张原版诅咒、顺序固定；
  - 新档：`PostDungeonInitialize` 晚于池初始化 10–43 ms（局1 +43 ms、局2 +10 ms）；
  - 读档（floor=2 / floor=3）：`PostDungeonInitialize` **0 次**，而 `initializeCardPools` 照常执行；
  - `StartGame` 在新档与读档都触发，且都在池重建之后（新档 5–16 ms 后、读档 3 ms 后）。
- **Remaining**: 本牌注册后按 T027 复测「存档→退出→继续」后池内不含本牌（探针阶段无法用真实本牌验证）。
- **Alternatives considered**: 只靠 `PostDungeonInitialize`（否决：读档不触发）；依赖卡牌类别或稀有度自然排除（否决：本体按卡牌库全量收录，无法自然规避）。

## R-008 不可移除的实现方式

- **Decision**: 杀戮尖塔 1 没有可启用的「不可移除属性」，本体自身也是靠硬编码 cardID 名单保护参照牌「铃铛的诅咒」。因此把本牌加入同款拦截点：`CardGroup.getPurgeableCards()` 的返回结果过滤掉本牌（覆盖商店移除、休息点、遗物与各类事件候选列表），净化之泉 `FountainOfCurseRemoval` 的批量删除循环跳过本牌；探针枚举出的其它移除路径一并补齐。拦截只判定本牌 cardID，不改变原版对其它卡牌的处理。
- **Rationale**: 依据本机 `desktop-1.0.jar` V2.3.4 反汇编结论（`memory/facts/f005`、过程 `memory/log/007`）；只有与参照牌落在同一批判定点，才能保证「不加限制、不放宽」。
- **Alternatives considered**: 给卡牌加自定义属性并在每个入口判断（否决：仍要拦截同一批入口，且多一套机制）。
- **Status（2026-10-24 第一轮采集）**: 本次日志未触发 `getPurgeableCards` / `FountainOfCurseRemoval` / `transformCard` / `removeCard(String)`，商店移除、休息点移除、净化之泉、变化四类场景均未跑（`awprobe addcard` 因控制台补全事故失败，见 `memory/log/018`）；拦截清单仍以 `memory/facts/f005` 的反汇编结论为准，待第二轮场景 5/6 补采后按实测增删。

## R-009 诅咒牌参数与文本对齐

- **Decision**: 诅咒牌继承 `CustomCard`：`CardType.CURSE`、`CardColor.CURSE`、费用 -2（本体「不可打出」判定依据）、稀有度取参照牌实际使用的 `CardRarity.SPECIAL`、不可升级、无使用效果、`makeCopy()` 返回新实例；名称与描述取自双语 `strings/`，描述按参照牌措辞直接写明「不能被打出」「不能从牌组中移除」，不使用关键词样式或颜色标记。
- **Rationale**: FR-004/005/012/013 要求与本体诅咒及参照牌行为一致；硬编码文本会违反宪法第二条。
- **Alternatives considered**: 复用参照牌文本 key（否决：命名空间与卸载行为不可控）。

## R-010 资源与占位贴图

- **Decision**: 中英双语文本放 `src/main/resources/strings/<modId>/`；开发期贴图放 `src/main/resources/placeholder/`，由 `src/` 层自己的 `.gitignore` 忽略，发布前替换为自制资源；`ModTheSpire.json` 随构建产出放在 jar 根。
- **Rationale**: 宪法第二条要求文本外置；不提交游戏本体图片（仓库纪律虽未写入宪法，但按既有约定执行）。
- **Alternatives considered**: 先提交本体贴图、后替换（否决：会污染仓库历史）。

## R-011 三层验证记录

- **Decision**: 可复现的流程写进本功能的 `quickstart.md`；每次实测的记录（操作、观察、探针输出、key 清单）追加到 `memory/log/`；只有经得起复用的结论才升入 `memory/facts/`，且必须带方法与适用范围。
- **Rationale**: 宪法第六条；`memory/README.md` 的严入标准。
- **Alternatives considered**: 把实测记录直接写进规格（否决：污染需求文档，且无法容纳过程数据）。

## R-012 未加载 mod 的旧档容错（路线 a）

- **Decision**: EC-006 的容错由游戏本体 / BaseMod 对未知自定义模式项的既有处理提供；本 mod 不承诺、也无法在「mod 未加载」时执行自身代码。验证方式为 QS-14（卸载 / 禁用本 mod 后加载旧档）；实测若崩溃，则停止该路线并回到 `/speckit-clarify` 重新界定 EC-006。
- **Rationale**: 未加载的 mod 没有执行机会，容错只能在加载器 / 本体侧；把它写成 mod 侧实现任务不可执行。mod 侧保留「读取自定义模式项列表时忽略未知标识」的健壮性即可。
- **Alternatives considered**: 在 mod 内实现「未见过的选项标识容错」（否决：覆盖不到本 mod 未加载的读数场景，只能防其它来源的未知项）。

## R-013 FR-005 的验证路线（路线 b）

- **Decision**: 本牌入手即判负、无法自然停留在手牌，故用**探针版临时关闭「抽到即负」**在开发期实测「不可打出 / 不可被主动消耗」，并与参照牌同条件对照（含持「蓝蜡烛」的一次）；正常版只验判负。该开关只存在于探针构建，随 T035 一并移除，发布版不含。
- **Rationale**: 直接观察不可打出需要牌停在手牌，正常版不可达；探针 + 参照牌对照兼顾可验证性与发布安全；本轮选择路线 b。
- **Alternatives considered**: 只做参照牌对照 +「不可达」论证（否决：对 FR-005 的直接验证不足）；在发布版保留调试开关（否决：违背发布洁净与不可被利用）。

## 风险与兜底（Risks and Fallbacks）

- **置底介入点覆盖不全**（最可能出问题）：首洗与耗尽重洗已实测覆盖；非耗尽重洗与多副本成组置底待补采；必要时在抽牌前加一道兜底校正，两道防线可并存，重复校正无副作用。
- **效果直接取牌的判定点**：R-005 实测确认不与抽牌共用（`addToHand` vs `addToTop`），按两个判定点实现；弃牌堆取牌路径待补采确认。
- **判负调用方式不正确**（误记放弃或不计失败）：以本体失败流程为准，实测同时核对结算界面与游戏记录。
- **注入晚于牌组固定**：R-006 实测确认 `PostCreateStartingDeck` 早于 Draft 填充与首洗；注入牌是否被填充清除由 `awprobe.inject` 标记实验补验，失败时改为 `preBattlePrep` 前缀。
- **不可移除入口枚举不全**：以探针清单为准逐入口补齐；必要时以 `CardGroup.removeCard` 守卫兜底（先确认不干扰「变化」的 remove+add 复合动作）。
- **随机诅咒池排除时机不对**：R-007 实测后改用 `StartGame` 钩子（新档与读档都触发且晚于池重建）；若时点再变则改 `initializeCardPools` 补丁。
- **读档后选项判定失效**：所有判定以存档恢复后的自定义模式项列表为准。
- **Maven 本地依赖配置繁琐**：先用最小可编译配置打通构建，再补依赖声明。

## 参考证据

- `memory/facts/f005`：本体不可移除与随机诅咒池都是硬编码 cardID 名单（V2.3.4 反汇编）。
- `memory/facts/f009`：BaseMod jar 实物的订阅接口名与签名（选项注册、注册牌、文本、开局注入、诅咒池钩子），以及 `CustomMod` 构造器（V2.3.4）。
- `memory/facts/f010`：洗牌原语（`shuffle(Random)` / `shuffle()` 互不转调）、`initializeDeck` 与 `EmptyDeckShuffleAction` 的搬运顺序、抽牌方向（drawPile 索引 0 = 底部）。
- `memory/facts/f011`：`initializeCardPools()` 内部调用 `addCurseCards()`；读档重建池但 `PostDungeonInitialize` 不触发。
- `memory/log/018`：第一轮游戏内探针采集（4 局 / 6 战）与 addcard 控制台事故的根因、修复。
- `memory/log/007`：参照牌类参数、文本出处与枚举过程。
- `memory/log/009`：第四轮 analyze 的口径与探针问题清单。
- `specs/_archive/001-anthonys-wrath/`：迁移前的旧三件套。
