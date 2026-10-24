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
- **Implemented（2026-10-24，T008 回写）**: 入口类同时实现 `EditStringsSubscriber` / `EditCardsSubscriber` / `AddCustomModeModsSubscriber` / `PostCreateStartingDeckSubscriber`，全部经 `BaseMod.subscribe(this)` 注册；选项在 `receiveCustomModeMods` 中 `mods.add(AnthonysWrathOption.create())`。选项实例 = `new CustomMod(optionId, "r", true)`：第 2 参数是**颜色**而非名称、第 3 参数 `isDailyMod` 才是持久化通道，口径见 §R-014。

## R-004 洗牌后置底介入点（2026-10-24 两轮探针实测，已定稿）

- **Decision**: 在洗牌原语上做后置介入：`patches/vanilla/` 对 `CardGroup.shuffle(Random)` 与 `CardGroup.shuffle()` 各挂一个 postfix；当被洗的组是抽牌堆 / 弃牌堆（战斗开始时是 `initializeDeck` 内部的临时 DRAW_PILE 组）且组内存在本牌时，把本牌移到**该组索引 0 端**。实测两条洗牌路径都从洗后组索引 0 开始消费，因此这一处理最终落在抽牌堆底部。`StartBattle` / `OnPlayerTurnStart` 只覆盖首次洗牌，不能作为唯一介入点。
- **Rationale**: 规格要求「每次洗牌后」都在底部（FR-006/007、EC-001）；洗牌后牌序已定，此时移动不依赖后续搬运实现，且两个洗牌原语是全部路径的公共汇合点。
- **Verified（两轮 awprobe.log：654 行 + 250 行；证据 `memory/log/018`、`memory/log/019`，抽牌方向与常量见 `memory/facts/f010`）**:
  - 首次洗牌 6 次：`CardGroup.shuffle <- initializeDeck:1031 <- preBattlePrep:1982`，作用对象是 `initializeDeck` 新建的临时 DRAW_PILE 组（`groupTag` 显示 `other(type=DRAW_PILE)`），此时 `player.drawPile` 仍为空；`initializeDeck` 之后按洗后顺序把牌填入抽牌堆。
  - 耗尽重洗 5 次：`CardGroup.shuffle <- EmptyDeckShuffleAction.update:49`，作用对象是 `player.discardPile`；该 action 每次 `update()` 只搬 1 张，搬入顺序 = 洗后弃牌堆索引 0→尾，抽牌堆最终顺序与洗后弃牌堆逐张一致（两轮、14 张与 10/11 张共三次独立核对）。
  - 钩子时序：`StartBattle` 在首洗之后、起手摸牌之前（早 0.8–0.9 s），`PlayerTurnStart` 紧随其后；中途重洗发生在 `PlayerTurnStart` 之后数秒，故两个钩子都不覆盖。
  - 抽牌方向：`getTopCard()` 返回 `group.get(size-1)`，两轮 15/15 次实测「dump 末位先被抽走」⇒ **drawPile 索引 0 = 底部（最后抽到），末位 = 下一次抽到**；`ProbeLog.ids()` 原注释「index 0 = top」已修正。
  - **无参原语**（第二轮）：`awprobe shuffledeck` 直接触发 `patch.shuffle().before/after`（`player.drawPile`，`CardGroup.shuffle()`），证明第二原语可被补丁观测且作用于玩家抽牌堆。
- **Remaining（留待实现后验证）**: 多副本成组置底与相对顺序（FR-007、QS-05）只能在真实卡牌注册后用 QS-05 验证；端到端置底断言属 QS-03 / QS-04 / QS-17。探针侧已无缺口：两个洗牌原语、首次与耗尽重洗、抽牌方向均已实测；原版是否存在「卡牌 / 遗物触发的非耗尽洗牌来源」未找到独立样本，但介入点落在洗牌原语上，任何来源都会被覆盖。
- **Implemented（2026-10-24，T016~T019 回写）**: 排序规则在 `run/DrawPileOrdering.sinkCurseToBottom(CardGroup)`，`patches/vanilla/CardGroupShufflePatch` 对两个原语各挂一个 postfix 调用它。
  - **作用范围**: 只处理 `CardGroupType.DRAW_PILE` / `DISCARD_PILE`（战斗开始的临时组由 `initializeDeck` 构造为 `DRAW_PILE`），外加 `player.drawPile` / `player.discardPile` 身份兜底；暂存 `UNSPECIFIED` 组与卡池组不重排。
  - **置底方向复核（字节码）**: `initializeDeck` 先 `new CardGroup(masterDeck, DRAW_PILE)` 再 `shuffle(shuffleRng)`，随后按索引 0→尾对 `drawPile.addToTop(card)` 搬运；`EmptyDeckShuffleAction` 洗 `discardPile` 后由 `Soul.shuffle` 把 `group` 指向 `drawPile` 再 `addToTop(card)`——两条路径都把被洗组的索引 0 映射为 `drawPile` 索引 0（底部），与 §R-004 的实测方向一致。
  - **多副本与缺席**: 按洗后相对顺序收成连续块放在 `0..k-1`；组内无本牌时直接返回 false、不做额外判定（EC-003）。
  - **洗牌路径静态清点（V2.3.4，全 jar 常量池扫描 + javap）**: 全库仅 8 处 `CardGroup.shuffle` 调用点——`initializeDeck`（战斗开始首次洗牌，临时 `DRAW_PILE` 组，`shuffle(Random)`）、`EmptyDeckShuffleAction`（`discardPile`，`shuffle(Random)`）、`ShuffleAction`（`Reboot` / `DeepBreath` 对 `drawPile` 调 `shuffle()`）、`ShuffleAllAction`（`Reboot`，`discardPile`，`shuffle(Random)`）这四处作用于战斗牌堆；其余 `DrawPileToHandAction` / `UpgradeRandomCardAction` / `TwistingMind` / `CardLibrary.getAnyColorCard` 洗的都是 `UNSPECIFIED` 暂存组，已被类型过滤跳过。另扫描全库 `Collections.shuffle` 调用：没有一处直接洗 `drawPile` / `discardPile` 的 `group` 列表（只有克隆的手牌列表、遗物/药水池、事件暂存组）。因此「卡牌 / 遗物触发的非耗尽重洗」真实样本是 `Reboot` / `DeepBreath`，且都收敛到两个原语。
  - **离线自检**: 复刻 MTS `paramtypez` 解析逻辑检查 18/18 补丁全部命中目标（含两条新补丁：`paramtypez={}` 命中 `shuffle()`、`{Random}` 命中 `shuffle(Random)`）；JDK8 `javac` 全量编译 exit 0。证据 `memory/log/023`。
  - **Remaining（转 T020）**: 游戏内端到端 QS-03 / QS-04 / QS-05 / QS-17；“多副本相对顺序”另有确定性自检 `awprobe sinktest`，多副本端到端用 `awprobe addcard anthonyswrath:AnthonysWrath` 后进战斗，非耗尽重洗可用 `awprobe shuffleaction`（真实 `ShuffleAction` 路径）。
- **Alternatives considered**: 只在 `StartBattle` 置底（否决：不覆盖中途重洗，实测重洗发生在该钩子之后）；只挂 `shuffle(Random)`（否决：无参重载是独立实现而非转调，必须两处都挂）；在每次抽牌前动态校正作为唯一手段（否决：不满足「洗牌后即在底部」的可观察性）。

## R-005 入手判定点（2026-10-24 两轮探针实测，已定稿）

- **Decision**: 两个判定点，不做「共用单一判定点」：① `CardGroup.addToHand` postfix（普通抽牌、临时牌入手、弃牌堆→手中的取牌）；② `CardGroup.addToTop` postfix 且目标组是 `player.hand`（抽牌堆→手中的「搜寻」取牌、Retain 回手）。两处都按 cardID 命中本牌即判负，并置本局一次性守卫。`PostDrawSubscriber` 只覆盖普通抽牌，不能作为唯一判定点。
- **Rationale**: FR-008、EC-008 要求「效果直接取牌」同样立即判负；EC-004 与 FR-010 要求只结算一次、后续不再处理抽牌。
- **Verified（两轮 awprobe.log；证据 `memory/log/018`、`memory/log/019`、`memory/facts/f013`）**:
  - 普通抽牌 80 次全部走 `addToHand <- AbstractPlayer.draw:2071 <- draw:2098 <- DrawCardAction.update:146`，并触发 `PostDraw`；
  - 临时牌 6 次走 `addToHand <- ShowCardAndAddToHandEffect.<init>:67 <- MakeTempCardInHandAction.addToHand:102`，不触发 `PostDraw`；
  - **抽牌堆**「搜寻」取牌 2 次只走 `removeCard(drawPile) <- SkillFromDeckToHandAction.update:54/:78` + `addToTop(hand) <- ...:55/:79`，**完全绕过 addToHand、PostDraw 与 draw(int)**；
  - **弃牌堆→手中**（第二轮，Meditate）走 `addToHand <- MeditateAction.update:84` 紧接着 `removeCard(discardPile) <- MeditateAction.update:86` ⇒ 弃牌堆取牌走 **addToHand**，与抽牌堆检索不同；
  - Retain 回手 10 次走 `addToTop(hand) <- RestoreRetainedCardsAction.update:26`；判定点必须按 cardID 过滤，避免把普通回手当成本牌入手。
- **Remaining（留待实现后验证）**: 判负落点（`AbstractDungeon.fail` / 结算界面）与「只结算一次」守卫属 QS-06 / QS-09；`addToBottom(hand)` 通道两轮均未触发（探针已布观测补丁，保留以防第三方效果）。
- **Alternatives considered**: 只挂 `PostDrawSubscriber`（否决：取牌不触发）；只挂 `addToHand`（否决：抽牌堆检索绕过）；回合结束扫描手牌（否决：不满足「同一瞬间结束」）。

## R-006 开局注入时机（2026-10-24 两轮探针实测，已定稿）

- **Decision**: 定在 `PostCreateStartingDeckSubscriber.receivePostCreateStartingDeck`：判定本局是否选中该选项，选中则加入恰好一张。2026-10-24 哨兵实验确认注入牌在随后的 Draft 填充中存活，且注入早于第一次洗牌（FR-003、EC-002），无需改用其它落点。
- **Rationale**: FR-003、EC-002；不实测无法判断注入牌在「空牌组 → 轮抽填充」过程中的存活，不能按猜测实现。
- **Verified（两轮 awprobe.log；证据 `memory/log/018`、`memory/log/019`）**:
  - Draft 局 `initializeStarterDeck` 前后、`PostCreateStartingDeck`、`StartGame`、`StartAct`、`PostDungeonInitialize` 看到的 `masterDeck` 全部为 0（`PostCreateStartingDeck` 收到的是空牌组）；
  - 填充发生在 `PostDungeonInitialize` 之后、首战 `preBattlePrep` 之前；`PostCreateStartingDeck` 比首次洗牌早约 11 s，「注入早于第一次洗牌」成立（FR-003）；
  - 普通对照局 `PostCreateStartingDeck` 即收到 10 张标准牌组；读档路径不触发 `initializeStarterDeck` / `PostCreateStartingDeck`（牌组来自存档 ⇒ 注入器不会重复加牌，对 FR-002 有利）；
  - **哨兵实验（第二轮）**: `PostCreateStartingDeck` 空牌组（size=0）→ `probe.inject marker=AscendersBane deck=size=1` → `StartGame` / `StartAct` / `PostDungeonInitialize` 均为 masterDeck=1 → Draft 填充 15 张后首战 `preBattlePrep` 为 **masterDeck=16 且含 `AscendersBane`** ⇒ 注入牌未被填充清除，落点成立。
- **Remaining**: 无探针侧缺口；实现后由 QS-08 做端到端复验（**2026-10-24 已通过**：真实卡牌 + Draft 局注入后填充 15 张仍存活，见 `memory/log/022`）。
- **Implemented（2026-10-24）**: `run/StartDeckInjector.injectIfSelected(...)` 挂在 `receivePostCreateStartingDeck`，选中时 `startingDeck.addToTop(new AnthonysWrathCurse())`（`CardGroup.addToTop` = 追加到组末），并用 cardID 防重复；选中判定见 §R-014。空牌组 Draft 局注入即 size=1，与哨兵实验一致。
- **Alternatives considered**: 在第一次战斗开始时补齐注入（否决：会被存档与开局校验遗漏）；`preBattlePrep` 前缀（否决：哨兵实验已证明 `PostCreateStartingDeck` 足够，无需更晚介入）。

## R-007 随机诅咒池排除（2026-10-24 探针实测；落点改为 StartGame 钩子）

- **Decision**: 用 `StartGameSubscriber.receiveStartGame` 把本牌从 `AbstractDungeon.curseCardPool` 移出：实测新档与读档都触发该钩子，且都在 `initializeCardPools()` 重建池之后。`PostDungeonInitializeSubscriber` **不能**作为唯一落点（读档不触发）。若后续版本观测到 StartGame 时点变化，退回 `patches/vanilla/` 对 `AbstractDungeon.initializeCardPools()` 打 postfix（其内部调用 `addCurseCards()` 填充池）。
- **Rationale**: 原方案只用 PostDungeonInitialize，读档路径会让本牌重新进入随机诅咒池，违反 FR-011、EC-010。
- **Verified（两轮 awprobe.log；证据 `memory/log/018`、`memory/log/019`，反汇编见 `memory/facts/f011`）**:
  - 4 次启动（新档 2、读档 2）全部执行 `initializeCardPools`，`curseCardPool` 稳定为 10 张原版诅咒、顺序固定；
  - 新档：`PostDungeonInitialize` 晚于池初始化 10–43 ms（局1 +43 ms、局2 +10 ms）；
  - 读档（floor=2 / floor=3）：`PostDungeonInitialize` **0 次**，而 `initializeCardPools` 照常执行；
  - `StartGame` 在新档与读档都触发，且都在池重建之后（新档 5–16 ms 后、读档 3 ms 后）；
  - **第二轮读档再次复现**：`PreStartGame floor=0` → `initializeCardPools floor=4` → `StartGame floor=4 masterDeck=11`，全程无 `initializeStarterDeck` / `PostDungeonInitialize`；`srcCurseCardPool` 同样只有 10 张原版诅咒。
- **Remaining**: 本牌注册后按 T027 复测「存档→退出→继续」后池内不含本牌（探针阶段无法用真实本牌验证）。
- **Alternatives considered**: 只靠 `PostDungeonInitialize`（否决：读档不触发）；依赖卡牌类别或稀有度自然排除（否决：本体按卡牌库全量收录，无法自然规避）。

## R-008 不可移除的实现方式

- **Decision**: 杀戮尖塔 1 没有可启用的「不可移除属性」，本体自身也是靠硬编码 cardID 名单保护参照牌「铃铛的诅咒」。因此把本牌加入同款拦截点：`CardGroup.getPurgeableCards()` 的返回结果过滤掉本牌（覆盖商店移除、休息点、遗物与各类事件候选列表），净化之泉 `FountainOfCurseRemoval` 的批量删除循环跳过本牌；探针枚举出的其它移除路径一并补齐。拦截只判定本牌 cardID，不改变原版对其它卡牌的处理。
- **Rationale**: 依据本机 `desktop-1.0.jar` V2.3.4 反汇编结论（`memory/facts/f005`、过程 `memory/log/007`）；只有与参照牌落在同一批判定点，才能保证「不加限制、不放宽」。
- **Alternatives considered**: 给卡牌加自定义属性并在每个入口判断（否决：仍要拦截同一批入口，且多一套机制）。
- **Status（2026-10-24 第二轮采集，证据 `memory/log/019`、`memory/facts/f014`）**:
  - **候选列表排除（方法级）**：`awprobe purge` 时 `masterDeck` 12 张含 `CurseOfTheBell`，`purgeable` 11 张**不含**它；同一方法在商店内被调用时补丁记录到 `caller=CardGroup.getPurgeableCards <- ShopScreen.purchasePurge:1204 <- ShopScreen.updatePurgeCard:1173 <- ShopScreen.update:620`。
  - **商店移除路径**：实删一张普通牌时记录到 `patch.removeCard(obj) | player.masterDeck card=Strike_P <- ShopScreen.updatePurge:329 <- ShopScreen.update:621` ⇒ 商店移除 = `getPurgeableCards()` 出候选 + `player.masterDeck.removeCard(AbstractCard)`。
  - **变化行为**：`transformCard(CurseOfTheBell)` 两次，结果分别为 `Parasite` / `Doubt`（都是 `CardType.CURSE`）；参照牌自身**永不**作为结果（`CardLibrary.getCurse()` 与 cursePool 均硬编码排除它）⇒ 本牌实现需同款：作为输入可被变化，且结果只会是其它随机诅咒。
  - **仍未实测**：净化之泉 `FountainOfCurseRemoval`（本轮未遇到事件，quickstart §5 允许跳过，拦截清单以 `memory/facts/f005` 反汇编兜底）、`removeCard(String)` 重载未触发、休息点移除未单独触发（未持有 Peace Pipe）——这三项留到实现后的 QS-11 实测。

## R-009 诅咒牌参数与文本对齐

- **Decision**: 诅咒牌继承 `CustomCard`：`CardType.CURSE`、`CardColor.CURSE`、费用 -2（本体「不可打出」判定依据）、稀有度取参照牌实际使用的 `CardRarity.SPECIAL`、不可升级、无使用效果、`makeCopy()` 返回新实例；名称与描述取自双语 `strings/`，描述按参照牌措辞直接写明「不能被打出」「不能从牌组中移除」，不使用关键词样式或颜色标记。
- **Rationale**: FR-004/005/012/013 要求与本体诅咒及参照牌行为一致；硬编码文本会违反宪法第二条。
- **Implemented（2026-10-24，T010/T013）**: `cards/AnthonysWrathCurse` 的 `super(...)` 参数为 `(CARD_ID, NAME, "placeholder/AnthonysWrath.png", -2, DESCRIPTION, CURSE, CURSE, SPECIAL, NONE)`；`CardType.CURSE` 使本体 `canUpgrade()` 恒 false（`upgrade()` 仅因抽象而留空），`makeCopy()` 显式返回新实例。名称/描述静态取自 `CardCrawlGame.languagePack.getCardStrings(CARD_ID)`（`EditStrings` 先于 `EditCards` 触发，类在 `receiveEditCards` 首次加载时字符串已就绪）。贴图为开发期占位图 `src/main/resources/placeholder/AnthonysWrath.png`（由 `src/.gitignore` 忽略，T036 替换）；`ImageMaster.loadImage` 对缺失图片只告警并返回 null，故不能省略占位文件。
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

## R-014 选项注册、选中状态与文本口径（2026-10-24 实现期实物核对，T007~T014）

- **Decision**:
  1. 选项 ID 固定 `anthonyswrath:AnthonysWrathOption`，按「运行期型负面修正」注册：`new CustomMod(optionId, "r", true)`（第 2 参数是颜色而非名称；第 3 参数 `isDailyMod=true` 与本体 `Binary` / `Terminal` 等运行期型负面修正一致）。
  2. 「本局是否选中」唯一来源 = `AbstractPlayer.customMods` 是否包含 optionId（只按标识判定，EC-007）。该静态列表在新开标准局时不会清空，故再与 `Settings.isTrial || CardCrawlGame.trial instanceof CustomTrial` 相与；新档注入额外要求 `CardCrawlGame.trial instanceof CustomTrial`（`run/RunState`）。
  3. 文本文件 `src/main/resources/strings/anthonyswrath/<lang>/{cards,run_mods}.json`，`<lang>` = `Settings.GameLanguage` 小写值（`eng` / `zhs`），其它语言回退 `eng`；`RunModStrings` 键 = optionId，`CardStrings` 键 = 完整 cardID。
  4. 中文选项名写入「负面：安东尼之怒」，英文名按规格保持 `Anthony's Wrath`。
- **Verified（`javap -p -c`：`libs/BaseMod.jar` 5.56.0 + `libs/desktop-1.0.jar` 12-18-2022；方法见 `memory/log/021`，结论已升为 `memory/facts/f015`、`f016`）**:
  - `CustomMod.<init>(String ID, String color, boolean isDailyMod)`：`this.color = color`、`this.ID = ID`、`name/description = CardCrawlGame.languagePack.getRunModString(ID)`、`this.isDailyMod = isDailyMod`；label = `colorString("[" + name + "]", color) + " " + description`。本体运行期型负面选项（DeadlyEvents / Binary / Cursed Run 等）一律 `color="r"` 且 `isDailyMod=true`；`isDailyMod=false` 的一批（Praise Snecko / One Hit Wonder / My True Form / Starter Deck 等）只在 `CustomModeScreen.addNonDailyMods` 里做开局改写、不进入持久化列表。
  - 新开自定义局：`CustomModeScreen.updateEmbarkButton` 把 `getActiveDailyModIds()` 交给 `CustomTrial.addDailyMods`，随后 `CardCrawlGame.trial = trial`、`AbstractPlayer.customMods = trial.dailyModIDs()`、`Settings.isTrial = true`。读档：`CardCrawlGame.loadPlayerSave` 恢复 `Settings.isTrial` 并 `AbstractPlayer.customMods = SaveFile.custom_mods`。存档：`SaveFile` 构造器把 `AbstractPlayer.customMods` 写入 `custom_mods` 字段。⇒ `isDailyMod=true` 是「选中状态跨读档保留」的必要条件。
  - 标准局 / 每日挑战：`CardCrawlGame.updateFade` 的 startOver 分支会 `trial = null`、`Settings.isTrial = false`（标准局安全）；每日挑战走自己的 trial 类型（注入判定用 `instanceof CustomTrial`，安全）。
  - 文本加载：`BaseMod.loadCustomStringsFile(Type.class, path)` = `Gdx.files.internal(path).readString(UTF-8)` → `loadJsonStrings`；`typeMaps` 把 `CardStrings → cards`、`RunModStrings → mod`。**5.56.0 的 `findCallingModName()` 是 `aconst_null; areturn` 桩**，`loadJsonStrings` 因此不会自动加 `<modId>:` 前缀 ⇒ JSON 键必须是完整 id。
  - 卡牌注册：`BaseMod.addCard(card)` 按 `CardColor.CURSE` 进 `curseToAdd`，由 `CurseCardsPatch` 在卡库初始化后 `CardLibrary.add(card)`；随后 `AbstractDungeon.initializeCardPools → addCurseCards` 会把它收进 `curseCardPool` ⇒ EC-010 仍需 T027 在 `StartGame` 中排除。
- **Verified in-game（2026-10-24：`awprobe.log` + 游戏日志 + `target/anthonyswrath.jar`（18:36 构建）；证据 `memory/log/022`）**:
  - 两局（铁甲战士普通自定义局、观者 + Draft 局）`PreStartGame` 均读到 `customMods` 含 `anthonyswrath:AnthonysWrathOption`（选中态在新档可读，与第 3 条一致）；
  - `PostCreateStartingDeck` 先后为 10 → 11（普通局，恰好 1 张）与 0 → 1（Draft 空牌组），`StartGame` / `StartAct` / `PostDungeonInitialize` 保持；Draft 填充 15 张后 `masterDeck=16` 仍含本牌，首洗晚于注入约 12 s；
  - `RunModStrings` / `CardStrings` 均无缺键报错（选项与卡牌文本命中 JSON）；游戏日志同时出现 BaseMod 的 `adding [ … anthonyswrath:AnthonysWrath ]` 记录；
  - 未覆盖：读档路径的选中态恢复与诅咒池复测（留 T027「存档→退出→继续」）；US2 未实现，首洗后本牌不在底部属预期。
- **「负面：」前缀结论（T009）**: 本体 `CustomModeScreen`、`localization/*/run_mods.json` 与 `ui.json` 中都没有「负面：」文本，负面性只由红色（`"r"`）方括号表达，界面槽位不提供该前缀。为满足 US1 验收场景 1 的显式文案，中文选项名写入「负面：安东尼之怒」（不会重复）；英文按规格固定为 `Anthony's Wrath`。详见 `memory/log/021`。
- **Alternatives considered**: 注册为 `isDailyMod=false`（否决：不进 `AbstractPlayer.customMods`，读档后无从判断，只能另造持久化，违背「不新增存档字段」）；读档后按显示名或卡牌是否在牌组反推选中状态（否决：无权威来源，且与 EC-007 冲突）。

## 风险与兜底（Risks and Fallbacks）

- **置底介入点覆盖不全**（最可能出问题）：首洗、耗尽重洗与无参洗牌原语均已实测覆盖；多副本成组置底只能在实现后由 QS-05 验证；必要时在抽牌前加一道兜底校正，两道防线可并存，重复校正无副作用。
- **效果直接取牌的判定点**：R-005 实测确认三条通道（抽牌/临时牌/弃牌堆取牌走 `addToHand`；抽牌堆检索走 `addToTop`），按两个判定点实现即可覆盖。
- **判负调用方式不正确**（误记放弃或不计失败）：以本体失败流程为准，实测同时核对结算界面与游戏记录。
- **注入晚于牌组固定**：R-006 哨兵实验确认 `PostCreateStartingDeck` 注入牌在 Draft 填充后存活且早于首洗，落点定稿；实现后 QS-08 复验。
- **不可移除入口枚举不全**：商店移除路径已实测（`getPurgeableCards` + `masterDeck.removeCard(obj)`）；净化之泉与休息点待实现后 QS-11 实测，清单以 `memory/facts/f005` 反汇编兜底。
- **随机诅咒池排除时机不对**：R-007 两轮实测后改用 `StartGame` 钩子（新档与读档都触发且晚于池重建）；若时点再变则改 `initializeCardPools` 补丁。
- **读档后选项判定失效**：所有判定以存档恢复后的自定义模式项列表为准。
- **Maven 本地依赖配置繁琐**：先用最小可编译配置打通构建，再补依赖声明。

## 参考证据

- `memory/facts/f005`：本体不可移除与随机诅咒池都是硬编码 cardID 名单（V2.3.4 反汇编）。
- `memory/facts/f009`：BaseMod jar 实物的订阅接口名与签名（选项注册、注册牌、文本、开局注入、诅咒池钩子），以及 `CustomMod` 构造器（V2.3.4）。
- `memory/facts/f010`：洗牌原语（`shuffle(Random)` / `shuffle()` 互不转调）、`initializeDeck` 与 `EmptyDeckShuffleAction` 的搬运顺序、抽牌方向（drawPile 索引 0 = 底部）。
- `memory/facts/f011`：`initializeCardPools()` 内部调用 `addCurseCards()`；读档重建池但 `PostDungeonInitialize` 不触发。
- `memory/facts/f013`：入手通道与判定点（抽牌/临时牌/弃牌堆取牌走 `addToHand`；抽牌堆检索走 `addToTop(hand)`；Retain 回手噪声）。
- `memory/facts/f014`：不可移除与变化的实测行为（`getPurgeableCards` 排除参照牌、商店移除链路、`transformCard(CurseOfTheBell)` → 其它随机诅咒）。
- `memory/log/018`：第一轮游戏内探针采集（4 局 / 6 战）与 addcard 控制台事故的根因、修复。
- `memory/log/019`：第二轮采集定稿（读档路径、商店移除、变化行为、Draft 注入存活、无参洗牌原语、弃牌堆取牌）。
- `memory/log/007`：参照牌类参数、文本出处与枚举过程。
- `memory/log/009`：第四轮 analyze 的口径与探针问题清单。
- `specs/_archive/001-anthonys-wrath/`：迁移前的旧三件套。
