# Implementation Plan: 安东尼之怒（Anthony's Wrath）

**Branch**: N/A | **Date**: 2026-10-23 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `specs/001-anthonys-wrath/spec.md`

## Summary

为自定义模式增加负面选项「安东尼之怒」：选中时在开局牌组加入一张同名诅咒牌，该牌在每次洗牌后必定位于抽牌堆最底部，玩家抽到它（或经效果直接取得）时本局立即判负，且无法通过任何「移除卡牌」途径除去、不会被随机诅咒来源发出。技术路线：用 BaseMod 订阅者注册选项 / 诅咒牌 / 双语文本 / 开局注入，用 `patches/vanilla/` 拦截本体硬编码的不可移除判定点，随机诅咒池用 `StartGame` 钩子排除（读档也触发；见 research R-007）、钩子不可用时再打补丁，置底与入手判定落点由游戏内时序探针实测后确定；仅依赖 BaseMod、StSLib 与 Java 标准库，Java 8 + Maven，源码与资源位于 `src/` 子模块。

## Technical Context

**Language/Version**: Java 8（编译目标与运行目标均为 Java 8，不使用更高版本语法与 API）

**Primary Dependencies**: ModTheSpire 加载器、BaseMod、StSLib，以及 Java 标准库；编译期依赖 jar（游戏本体、ModTheSpire、BaseMod、StSLib）统一放在顶层 `libs/`，一律不入库。其中游戏本体与 ModTheSpire 以 provided / compileOnly 方式仅作平台产物参与编译，不视为第三方依赖；BaseMod 与 StSLib 是本功能依赖边界内仅有的两个第三方库。

**Storage**: 不写存档；功能在开局或读档时以存档恢复后的玩家自定义模式项列表为准，不新增任何存档字段（宪法第四条自动满足）

**Testing**: 不引入自动化测试框架（宪法第一条禁止第三方依赖）；验证方式为游戏内实测 + `quickstart.md` 中的可复现步骤

**Target Platform**: Slay the Spire 1.x 桌面版 + ModTheSpire 加载器（Windows）

运行期前置（一次性人工准备，T004 起共用）：游戏侧已装好 ModTheSpire 加载器与 BaseMod、StSLib，采用 Steam 创意工坊订阅或手动放入游戏目录 `mods/` 两种方式之一；判据为 ModTheSpire 启动日志的 Mod list 中 `basemod` 与 `stslib` 同时出现、且 BaseMod 完成初始化。安装结果（创意工坊订阅清单、启动器日志摘录或截图）记入 `memory/log/` 作为可查证据。本仓库不提交任何 jar。此项未完成时，T004 时序探针与全部游戏内实测任务均不可执行。

**Project Type**: 单机游戏 mod（桌面应用插件）；Maven 单工程 + 三个子模块挂载点（`specs/`、`memory/`、`src/`）

**Performance Goals**: 无定量指标；置底与入手判定只发生在洗牌 / 抽牌路径，均为牌组规模的常数级操作，不得引入可感知卡顿

**Constraints**: 仅允许 BaseMod / StSLib 两个第三方库（游戏本体与 ModTheSpire 为 provided 平台产物，不计入依赖边界）；玩家可见文本必须外置到 `strings/` 中英双语且 key 一一对应；对原版的拦截集中在 `src/main/java/<modId>/patches/vanilla/`；本 mod 未加载（卸载或禁用）时读旧档不得崩溃；构建产物为单个 mod jar，jar 与占位图不入库

**Scale/Scope**: 单功能：1 个自定义模式选项、1 张诅咒牌、1 个补丁包、1 套中英双语文本与占位贴图；预计新增类少于 20 个，无外部服务、无数据迁移

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| # | 宪法条款 | 门禁 | 结论与落点 |
| :--- | :--- | :--- | :--- |
| 0 | 版本断言（引言） | 游戏补丁版本与 BaseMod / StSLib 版本可只从顶层 `pom.xml` 的 `<properties>` 独立查得 | **PASS**：由首个基础任务把版本键写入顶层构建配置 |
| 一 | 依赖边界 | 除 BaseMod、StSLib 与 Java 标准库外无第三方依赖 | **PASS**：游戏本体 / ModTheSpire 以 provided 平台产物参与编译、不计入依赖；BaseMod / StSLib 之外不引入任何库，测试亦不引入框架 |
| 二 | 文本与本地化 | 玩家可见文本全部外置于 `src/main/resources/strings/`，中英 key 一一对应 | **PASS**：文本全部走 `strings/`；key 比对步骤见 quickstart |
| 三 | 原版改动可追溯 | 所有对原版行为的拦截只出现在 `patches/vanilla/` | **PASS**：不可移除与随机诅咒池的拦截集中在本包内，且只判定本牌 |
| 四 | 存档兼容 | 加载旧存档不得抛异常；新增存档字段必须可缺省 | **PASS**：本功能不写存档；未加载容错由本体 / BaseMod 提供、QS-14 只作验证；mod 侧读取列表时容忍未知标识 |
| 五 | 完成标准 | 构建通过 + 游戏内实测至少一次 + 留下可复现验证步骤 | **PASS**：构建命令与全部实测场景写入 `quickstart.md` |
| 六 | 三层归属 | 内容各归其层；验证记录按「流程 / 日志 / 结论」三级归属 | **PASS**：规格在本层、代码与资源在 `src/`、记录在 `memory/log/`；流程写入 `quickstart.md` |
| 七 | 版本锁定 | 三层以 gitlink 锁定，冻结点指针与各层提交一致 | **PASS**：里程碑任务按 gitlink 断言校验 |
| 八 | 构建配置归属 | `pom.xml`、依赖版本、工具链与 `AGENTS.md` 只在顶层 | **PASS**：构建配置只落在顶层仓库 |
| 九 | 提交顺序 | 先提交被改动的层，再由顶层提交推进指针 | **PASS**：任务按 specs → memory → 顶层的顺序安排提交与指针推进 |

**Phase 1 后复检**：设计未引入外部接口、未新增依赖、未改变三层归属，全部条款仍为 PASS；无需例外说明。

## Project Structure

### Documentation (this feature)

```text
specs/001-anthonys-wrath/
├── plan.md              # 本文件(/speckit-plan 输出)
├── research.md          # Phase 0 输出:技术决策、未决项与风险兜底
├── data-model.md        # Phase 1 输出:实体、规则与状态转换
├── quickstart.md        # Phase 1 输出:可复现、可重跑的验证流程
├── checklists/
│   └── requirements.md  # 规格质量清单(/speckit-specify 输出)
├── spec.md              # 功能规格(/speckit-specify 输出)
└── tasks.md             # Phase 2 输出(/speckit-tasks 生成)
```

本项目不对外提供编程接口，故不生成 `contracts/`。

### Source Code (repository root)

```text
# 顶层仓库:构建配置与三层挂载点
pom.xml                       # Maven 配置:Java 8;显式 sourceDirectory/resources 指向 src/
libs/                         # 编译期依赖 jar(游戏本体、ModTheSpire、BaseMod、StSLib;不入库)
.gitmodules / AGENTS.md / .gitignore
specs/  memory/  src/         # 三个子模块挂载点

# src/ 子模块:代码、数据与资源
src/main/java/<modId>/
├── <ModEntry>.java           # BaseMod 初始化入口与订阅者注册
├── options/                  # 自定义模式选项
├── cards/                    # 诅咒牌「安东尼之怒」
├── run/                      # 开局注入、洗牌置底、入手判定
└── patches/vanilla/          # 原版判定点拦截(洗牌置底、入手判定与终局短路、不可移除、随机诅咒池)
src/main/resources/
├── strings/<modId>/          # 中英双语文本(key 一一对应)
├── placeholder/              # 开发期占位贴图(src/ 层 .gitignore 忽略)
└── ModTheSpire.json          # 加载器清单(jar 根)
```

**Structure Decision**: 沿用既有三层结构——构建配置在顶层、代码与资源在 `src/` 子模块、规格在该功能目录；本功能不新增顶层目录，只在 `src/` 内建立包结构。具体 mod 标识与 Java 包名由首个基础任务确定并写入顶层构建配置，全项目以该配置为唯一命名依据。
