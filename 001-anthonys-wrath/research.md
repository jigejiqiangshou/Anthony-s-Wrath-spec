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

- **Decision**: 入口类实现并订阅：`EditCardsSubscriber.receiveEditCards`（注册诅咒牌）、`EditStringsSubscriber.receiveEditStrings`（加载双语文本）、`AddCustomModeModsSubscriber.receiveCustomModeMods(List<CustomMod>)`（注册自定义模式选项）、`PostCreateStartingDeckSubscriber.receivePostCreateStartingDeck(PlayerClass, CardGroup)`（开局注入）、`PostDungeonInitializeSubscriber.receivePostDungeonInitialize`（随机诅咒池排除）；初始化阶段完成 `BaseMod.subscribe(this)`。
- **Rationale**: 全部走 BaseMod 公开订阅点，避免不必要的原版补丁；只有订阅点覆盖不到的判定才进 `patches/vanilla/`。
- **Alternatives considered**: 全部用补丁实现（否决：补丁面积大、升级游戏版本时易碎）。
- **Verified（2026-10-24，实物核对）**: `libs/BaseMod.jar` 内实有 `basemod/interfaces/AddCustomModeModsSubscriber.class`，方法为 `receiveCustomModeMods(java.util.List<com.megacrit.cardcrawl.screens.custom.CustomMod>)`；对应发布端 `BaseMod.publishAddCustomModeMods(List<CustomMod>)`。旧文档中的 `CustomModeMods` / `receiveAddCustomModeMods` 不存在，作废。方法：`jar tf` + `javap`，证据见 `memory/facts/f009` 与 `memory/log/014`。

## R-004 洗牌后置底介入点（待探针）

- **Decision**: 由时序探针确定；默认优先在「洗牌结束、牌堆顺序已确定」的时机把该牌移到抽牌堆底部，候选为战斗开始钩子与洗牌底层介入点（若落在原版方法上则补丁进 `patches/vanilla/`）。必须同时覆盖首次洗牌、牌堆耗尽重洗、以及由卡牌 / 遗物触发的非耗尽重洗。
- **Rationale**: 规格要求「每次洗牌后」都在底部（FR-006/007、EC-001）；不实测无法确认钩子顺序，猜测会漏路径。
- **Alternatives considered**: 只在战斗开始时置底（否决：无法覆盖中途重洗）；在抽牌前动态校正作为唯一手段（否决：不满足「洗牌后即在底部」的可观察性）。

## R-005 入手判定点（待探针）

- **Decision**: 由时序探针确认「效果直接取牌」是否与「抽牌动作」走同一判定点；能共用则只实现一个判定点，不能共用则另建覆盖直接取牌的监听。任一实现都必须在取得动作完成时立即判负，并置位本局一次性守卫。若两条路都不可行，**不得**缩减规格覆盖范围，须回到 `/speckit-clarify` 重新界定 EC-008。
- **Rationale**: FR-008、EC-008 要求两条路径都判负；EC-004 与 FR-010 要求只结算一次、后续不再处理抽牌。
- **Alternatives considered**: 仅在回合结束扫描手牌（否决：不满足「同一瞬间结束」）。

## R-006 开局注入时机（待探针）

- **Decision**: 在 `PostCreateStartingDeck` 中判定本局是否选中该选项，选中则加入恰好一张；探针需确认「清空初始牌组」与「初始牌组创建」的先后，以决定注入挂在清空之前还是之后，结果必须满足「注入早于第一次洗牌」（FR-003、EC-002）。
- **Rationale**: 自定义模式允许清空初始牌组；注入晚于牌组固定会导致诅咒牌不进入本局牌库。
- **Alternatives considered**: 在第一次战斗开始时补齐注入（否决：会被存档与开局校验遗漏）。

## R-007 随机诅咒池排除

- **Decision**: 本体每次开局 / 读档重建 `AbstractDungeon.curseCardPool` 时会把卡牌库中所有「诅咒」收进池内（参照牌等四张由硬编码 cardID 排除）。优先用 `PostDungeonInitializeSubscriber` 钩子把本牌移出该池；探针须确认该钩子晚于 `AbstractDungeon.initializeCardPools()`。钩子不合用时改为 `patches/vanilla/` 补丁。
- **Rationale**: 不排除会让未选择该选项的玩家从事件 / 遗物处收到本牌（违反 FR-011、EC-010）。
- **Alternatives considered**: 依赖卡牌类别或稀有度自然排除（否决：本体按卡牌库全量收录，无法自然规避）。

## R-008 不可移除的实现方式

- **Decision**: 杀戮尖塔 1 没有可启用的「不可移除属性」，本体自身也是靠硬编码 cardID 名单保护参照牌「铃铛的诅咒」。因此把本牌加入同款拦截点：`CardGroup.getPurgeableCards()` 的返回结果过滤掉本牌（覆盖商店移除、休息点、遗物与各类事件候选列表），净化之泉 `FountainOfCurseRemoval` 的批量删除循环跳过本牌；探针枚举出的其它移除路径一并补齐。拦截只判定本牌 cardID，不改变原版对其它卡牌的处理。
- **Rationale**: 依据本机 `desktop-1.0.jar` V2.3.4 反汇编结论（`memory/facts/f005`、过程 `memory/log/007`）；只有与参照牌落在同一批判定点，才能保证「不加限制、不放宽」。
- **Alternatives considered**: 给卡牌加自定义属性并在每个入口判断（否决：仍要拦截同一批入口，且多一套机制）。

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

- **置底介入点覆盖不全**（最可能出问题）：探针需覆盖首次洗牌、耗尽重洗与非耗尽重洗；必要时在抽牌前加一道兜底校正，两道防线可并存，重复校正无副作用。
- **找不到效果直接取牌的判定点**：先确认是否与抽牌共用；不能共用则另建监听；两条路都不可行时回到 `/speckit-clarify`，不得缩减覆盖范围。
- **判负调用方式不正确**（误记放弃或不计失败）：以本体失败流程为准，实测同时核对结算界面与游戏记录。
- **注入晚于牌组固定**：由探针暴露，改用更早介入点或在该场战斗前补齐。
- **不可移除入口枚举不全**：以探针清单为准逐入口补齐；必要时以 `CardGroup.removeCard` 守卫兜底（先确认不干扰「变化」的 remove+add 复合动作）。
- **随机诅咒池排除时机不对**：优先钩子，不合用改补丁。
- **读档后选项判定失效**：所有判定以存档恢复后的自定义模式项列表为准。
- **Maven 本地依赖配置繁琐**：先用最小可编译配置打通构建，再补依赖声明。

## 参考证据

- `memory/facts/f005`：本体不可移除与随机诅咒池都是硬编码 cardID 名单（V2.3.4 反汇编）。
- `memory/facts/f009`：BaseMod jar 实物的订阅接口名与签名（选项注册、注册牌、文本、开局注入、诅咒池钩子），以及 `CustomMod` 构造器（V2.3.4）。
- `memory/log/007`：参照牌类参数、文本出处与枚举过程。
- `memory/log/009`：第四轮 analyze 的口径与探针问题清单。
- `specs/_archive/001-anthonys-wrath/`：迁移前的旧三件套。
