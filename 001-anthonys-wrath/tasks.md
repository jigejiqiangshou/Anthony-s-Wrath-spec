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

- [ ] T004 **前置（一次性人工准备）**：ModTheSpire 加载器与 BaseMod、StSLib 已可用（Steam 创意工坊订阅或手动放入游戏目录 `mods/`），且 ModTheSpire 启动日志的 Mod list 中 `basemod` 与 `stslib` 同时出现、BaseMod 完成初始化；安装证据（订阅清单或启动器日志摘录）记入 `memory/log/`，作为本任务可被执行的判据。编写并运行游戏内时序探针 `src/main/java/<modId>/probe/TimingProbe.java`（仅日志输出）：回答 research.md R-004～R-008 与 quickstart §3 的六个问题——首次洗牌与候选钩子的先后、能覆盖首次洗牌与重洗的置底介入点、效果直接取牌与抽牌的判定点是否同一、清空初始牌组与开局牌组创建的先后、参照牌「铃铛的诅咒」在变化 / 移除下的实际表现、`PostDungeonInitialize` 是否晚于 `AbstractCardPools` 初始化；并枚举本体让参照牌「删不掉」的全部判定点，回写 research.md §R-008；探针输出与结论追加到 `memory/log/`（2026-10-24：探针代码已就位 `src/main/java/anthonyswrath/probe/`，含 TimingProbe / AwProbeCommand / ProbePatches / ProbeLog，JDK8 `javac` 编译通过；待用户运行采集数据，见 `memory/log/015`）
- [ ] T005 [P] 建立双语文本资源骨架 `src/main/resources/strings/<modId>/`：中英 key 一一对应，格式按 BaseMod 实际加载规则在依赖 jar 就绪后核对；本任务只建空骨架与命名约定
- [ ] T006 [P] 建立 `src/main/java/<modId>/patches/vanilla/` 包骨架与命名约定，供 US5 逐条实现；不可移除判定点清单由 T004 的枚举结论直接回写 `research.md` §R-008，本任务不重复承接该回写

**Checkpoint**: 实现落点全部有实测依据，文本与补丁包骨架就绪。

---

## Phase 3: User Story 1 - 选择选项并在开局牌组中获得诅咒牌 (Priority: P1) 🎯 MVP

**Goal**: 玩家在自定义模式中可见并勾选本选项；开局牌组获得恰好一张同名诅咒牌（含空开局牌组）。

**Independent Test**: 在自定义模式勾选本选项开一局，打开牌组查看界面核对（quickstart.md QS-01 / QS-02 / QS-08）。

### Implementation for User Story 1

- [ ] T007 [P] [US1] 实现自定义模式选项类 `src/main/java/<modId>/options/AnthonysWrathOption.java`：稳定 `optionId` 形如 `<modId>:AnthonysWrathOption`，显示名与说明文本取自 `strings/`，不写死字符串
- [ ] T008 [US1] 在入口类注册该选项（注册接口名以 BaseMod jar 实物为准，并回写 research.md §R-003），确认标识不与本体选项冲突，同名第三方选项通过标识区分、互不覆盖（`EC-007`）
- [ ] T009 [P] [US1] 在双语资源写入选项显示名「安东尼之怒」与勾选前可见的效果说明文本（`FR-014`），路径 `src/main/resources/strings/<modId>/`；并比对本体负面选项的前缀规则，确认「负面：」前缀由界面槽位提供还是需写入文本、不得重复，结论记入 `memory/log/`
- [ ] T010 [P] [US1] 实现诅咒牌类 `src/main/java/<modId>/cards/AnthonysWrathCurse.java`：`CardType.CURSE`、`CardColor.CURSE`、费用 -2、`CardRarity.SPECIAL`、不可升级、无使用效果、`makeCopy()` 返回新实例（`FR-004`、`FR-005`）
- [ ] T011 [US1] 在 `EditCardsSubscriber.receiveEditCards` 中注册诅咒牌并确认游戏将其识别为诅咒（`FR-004`），路径 `src/main/java/<modId>/<ModEntry>.java`
- [ ] T012 [US1] 实现开局注入 `src/main/java/<modId>/run/StartDeckInjector.java`：实现 `PostCreateStartingDeckSubscriber`，基于 `optionId` 判定本局是否选中，选中则加入恰好一张；注入点在牌组创建之后、首次洗牌之前，具体先后按 T004 结论执行（`FR-002`、`FR-003`；覆盖 `EC-002` 空开局牌组）
- [ ] T013 [P] [US1] 在双语资源写入诅咒牌名称与描述初版（名称：中文「安东尼之怒」/ 英文 `Anthony's Wrath`），路径 `src/main/resources/strings/<modId>/`；描述定稿由 T033 按参照牌措辞覆盖完成
- [ ] T014 [US1] 选项标识容错（路线 a）：从存档恢复后的自定义模式项列表读取时忽略未知标识，保证本 mod 加载时对其它未知项健壮；EC-006 的「本 mod 未加载」场景由本体 / BaseMod 负责、QS-14 只作验证（实测崩溃则回 `/speckit-clarify`），路径 `src/main/java/<modId>/options/`
- [ ] T015 [US1] 实测并记录 US1：选项可见 / 显示名 / 效果说明（QS-01，`FR-014`）、开局牌组恰好一张（QS-02）、空开局牌组注入（QS-08，仅验加入与时机）、未加载本 mod 后读档不崩溃（QS-14）；结果追加到 `memory/log/`

**Checkpoint**: 用户故事 1 可独立验证——选项可见、选中后牌组含牌、卸载容错。

---

## Phase 4: User Story 2 - 每次洗牌后诅咒牌位于抽牌堆底部 (Priority: P2)

**Goal**: 战斗开始洗牌、抽牌堆耗尽重洗、以及卡牌 / 遗物触发的非耗尽洗牌后，该牌都在抽牌堆最底部；多副本成组且相对顺序不变。

**Independent Test**: 一场战斗中按顺序查看抽牌堆（「冰冻之眼」优先），覆盖三次洗牌路径（quickstart.md QS-03 / QS-04 / QS-05）。

### Implementation for User Story 2

- [ ] T016 [US2] 实现置底逻辑 `src/main/java/<modId>/run/DrawPileOrdering.java`：在 T004 确定的「洗牌结束、顺序已确定」介入点把该牌移到抽牌堆底部（`FR-006`）
- [ ] T017 [US2] 在入口类接入该逻辑，覆盖战斗开始首次洗牌、抽牌堆耗尽重洗与卡牌 / 遗物触发的非耗尽重洗（`FR-006`、`EC-001`）
- [ ] T018 [US2] 处理多副本：全部副本成组置于底部且相对顺序不变（`FR-007`），并在找不到该牌时直接跳过、不做额外判定（`EC-003`）
- [ ] T019 [US2] 若介入点落在原版方法上，则把对应补丁放入 `src/main/java/<modId>/patches/vanilla/`，只判定本牌 cardID
- [ ] T020 [US2] 实测并记录 US2：首次洗牌置底（QS-03，含空开局牌组）、两次以上重洗（含非耗尽，QS-04）、多副本成组（QS-05）、第二场战斗重新置底（QS-17），覆盖 `SC-001`、`SC-002`；结果追加到 `memory/log/`

**Checkpoint**: 用户故事 2 可独立验证——三条洗牌路径后该牌均在底部。

---

## Phase 5: User Story 3 - 抽到诅咒牌立即判负 (Priority: P3)

**Goal**: 该牌被抽入手中（或经效果直接取得）时本局立即判负并计入失败记录；只结算一次，之后不再处理抽牌与操作。

**Independent Test**: 等待自然抽到该牌、再用「搜寻」类效果取牌各一次，核对结算界面与游戏记录（quickstart.md QS-06 / QS-07 / QS-09 / QS-15）。

### Implementation for User Story 3

- [ ] T021 [US3] 实现入手判定 `src/main/java/<modId>/run/DefeatOnDraw.java`：在取得动作完成时立即触发本局失败，走游戏本体失败流程（`FR-008`、`FR-009`）
- [ ] T022 [US3] 覆盖「效果直接取牌」路径（`EC-008`）：按 T004 结论与抽牌共用判定点，或另建独立监听；两条路都不可行时停止并回到 `/speckit-clarify`，不得缩减覆盖范围
- [ ] T023 [US3] 加入本局一次性守卫与终局短路：失败只结算一次（`FR-010`），判负后后续抽牌在入口整段跳过、不接受可操作结果（`EC-004`）
- [ ] T024 [US3] 在入口类接入判定组件；若判定点需拦截原版抽牌动作，补丁放入 `patches/vanilla/`，否则优先使用 BaseMod 既有钩子
- [ ] T025 [US3] 实测并记录 US3：自然抽到即负（QS-06，`SC-003`）、搜寻取牌判负（QS-07）、多副本只结算一次与终局短路（QS-09）、失败记录（QS-15）、主动放弃不触发特判（QS-16，`EC-005`）；结果追加到 `memory/log/`

**Checkpoint**: 用户故事 3 可独立验证——抽到即负、只结算一次、失败入记录。

---

## Phase 6: User Story 4 - 未选择选项时无任何影响 (Priority: P4)

**Goal**: 未勾选本选项时，牌组、抽牌堆与随机诅咒来源都不出现该牌；标准模式与每日挑战同样不受影响。

**Independent Test**: 不勾选开三局（标准 / 每日 / 自定义），逐张比对牌组，并用事件 / 遗物反复产出诅咒观察（quickstart.md QS-10）。

### Implementation for User Story 4

- [ ] T026 [US4] 让所有注入 / 置底 / 判负组件在未选中时短路：统一以存档恢复后的自定义模式项列表 + `optionId` 为准（`FR-011`），路径 `src/main/java/<modId>/run/`
- [ ] T027 [US4] 把本牌排除出随机诅咒池：优先在 `PostDungeonInitializeSubscriber.receivePostDungeonInitialize` 中移出（时序以 T004 结论为准），钩子不可用时改为 `patches/vanilla/` 补丁；记录所选落点（`EC-010`）
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
- [ ] T041 按 [quickstart.md](./quickstart.md) 完成一次完整回归（QS-01 至 QS-17，覆盖 `SC-001` 至 `SC-005`），逐条记录结果到 `memory/log/`，作为宪法第五条「完成标准」的证据

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
