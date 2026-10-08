# SatGPT 洪水年度数据更新与 PR 交接说明

日期：2026-10-08。供代码交接、PR 描述和维护者审核使用。

本文件详细说明本次历史图层更新，并列出同一工作区中其他修改的审核入口。当前代码包含未提交修改；生成本文件时，没有执行提交、推送、创建 PR 或远程合并。

## 1. 修改目的

`Single Inundation Event` 和 `Inundation Hotspot` 原先只能选择到 2021 年。本次把这两个历史图层接入的年度数据更新到 **1984–2024**，并同步年份选择、来源说明和代码导出。

没有直接把上限标成 2026：截至 2026-10-08 核查，[JRC 官方数据发布页](https://global-surface-water.appspot.com/download)提供的最新完整年度记录到 2024 年。2025–2026 年的具体事件应使用基于事件日期和卫星影像的 Agent / Flood Detection 流程，实际结果取决于影像和服务可用性。

**历史年度水体图层的覆盖年份，与 Agent 可以查询、尝试分析的事件年份，是两回事。** 季节性水体分类也不能直接证明发生了某一场洪水。

## 2. 年份更新的前后变化

| 项目 | 修改前 | 当前行为 |
| --- | --- | --- |
| 两个历史图层的年份上限 | 2021 | 2024 |
| 默认完整 Hotspot 周期 | 1984–2021，38 年 | 1984–2024，41 年 |
| 年度数据来源 | 单一 JRC v1.4 年度集合 | 按年份拼接旧数据、修正数据和新增数据 |
| 两个图层的年份范围配置 | 前后端分别限制到 2021 | 从共享数据配置读取 1984–2024 范围 |
| 图层说明和来源 | 覆盖到 2021 | 覆盖到 2024，并说明较新事件使用 Flood Detection |
| 导出的 GEE 分析代码 | 使用原 v1.4 集合 | 使用与后端对应的三个数据来源及年份筛选 |

仍使用原 v1.4 数据的其他目录产品，保持各自原有覆盖范围；本次不是所有 JRC 图层的整体升级。

### 数据怎样拼接

| 采用年份 | Earth Engine 资产 |
| --- | --- |
| 1984–2015 | `JRC/GSW1_4/YearlyHistory`，仅筛选这一段 |
| 2016–2021 | `projects/global-surface-water/assets/GSW1_5/YearlyHistory_2016_2021` |
| 2022–2024 | `projects/global-surface-water/assets/GSW1_5/YearlyHistory_2022_2024` |

使用修正后的 2016–2021 年数据，并排除旧集合中的重复年份，合并后为 **41 个唯一年份**。涉及这一段旧年份的分析结果也可能因数据修正而变化。

JRC 提示，新旧 Landsat 集合之间可能存在配准偏移；来源说明已保留这一限制。完整来源与更新说明见 [JRC Data Access](https://global-surface-water.appspot.com/download)。

### 90% 封顶的最终处理

**保留原系统的 90% 频率封顶和原图例。** 超过 90% 的热点频率仍按 90% 输出，后端与导出的 GEE 代码一致。

提供的评审文档将热点图层称为“原系统的一个简化的洪水逻辑”，但没有直接解释为什么选择 90%。本次保持旧方法，不将封顶定性为 bug，也不将取消封顶混入年度数据更新。封顶仍影响原始频率值；若以后调整，应作为独立的方法变更审核。

## 3. 年份更新对应的文件

部分文件还包含其他功能的未提交修改。审核或拆分 PR 时，需要检查具体代码段，不能认为整份文件的差异都属于年份更新。

| 文件 | 本次年度更新涉及的内容 |
| --- | --- |
| `frontend/src/config/layerCatalog.json` | 共享年度范围、三个来源及其采用年份；移除两个图层重复的单一年度来源配置 |
| `agent/flood_api_services.py` | 读取共享范围、拼接年度集合；两个图层及下载采用新集合 |
| `frontend/src/hooks/useAgentLayerManagerGroups.js` | 两个图层的年份限制、刻度、默认周期、年数和帮助文字 |
| `frontend/src/config/agentRasterLayerConfig.js` | 覆盖范围和说明文字；保留原图例 |
| `frontend/src/config/agentLayerSourceReferences.js` | 数据资产、覆盖年份、官方来源和使用限制 |
| `frontend/src/export/geeCodeGenerator.js` | 两类导出采用相同的数据拼接方案；保留 90% 封顶 |
| `frontend/src/hooks/useAgentLayerManagerGroups.test.jsx` | 两个滑块可提交 2024 年；其他 v1.4 产品仍按原覆盖范围限制 |
| `frontend/src/export/geeCodeGenerator.test.js` | 两类导出包含新增、修正来源及年份筛选，生成代码语法有效 |

## 4. 同一工作区的其他修改

如果提交整版代码，PR 描述需要同时说明以下内容；如果只提交年份更新，需要先整理并排除其他功能差异。

| 修改组 | 目的和审核入口 |
| --- | --- |
| 行政边界搜索、选择和完整几何裁剪 | 修正 Bangkok 范围、候选歧义、近似范围和自动抽稀问题。详细行为、来源、文件和限制见 [行政边界修改说明](BOUNDARY_CHANGES.md) |
| 洪水影像请求与加载状态 | 让切换 FLOOD 面板不因组件卸载而取消分析请求；显示加载、可用或失败状态，并防止旧响应覆盖新范围。入口：`FloodAgentViewSync.jsx`、`FloodImageryStatus.jsx`、`useFloodAnalysisRequests.js`、`AppContext.jsx` |
| Agent 事件确认与提示 | 事件和空间范围确认、分析启动条件及英文提示等流程调整。入口：`agent/flood_agent.py`、`agent/prompts.py`、`agent/state.py`、`frontend/src/utils/floodWorkflow.js` 及对应测试 |

这张表提供审核分组，不是所有旧差异的逐行审计结论。新增的 Python 模块、边界数据、前端组件、工具和测试也需纳入整版交付；只复制修改过的旧文件可能遗漏依赖。

## 5. 验证记录与范围

最终恢复 90% 封顶后，已完成：

- 相关前端测试：**2 个文件、13 项通过**，覆盖年份滑块和 GEE 导出。
- 当前工作区后端测试：**74 项通过**。这是整个后端测试集的数量，并非全部针对年份更新。
- 真实 Earth Engine 检查：合并集合包含 **41 个唯一年份，1984–2024**。
- 本机运行服务的 Bangkok 示例、2022–2024 周期：原始热点频率最大值 **1.0**，封顶后 **0.9**；与保留封顶规则的独立计算相比，示例范围内最大差值为 **0**。
- 本机 Compose 项目 `satgpt` 的 agent 与 frontend 已重建、更新，健康检查通过。

此前年度更新完成时，完整前端测试曾通过 **25 个文件、96 项测试**。恢复封顶后重新执行的是上述 13 项相关前端测试和 74 项后端测试。

这些记录说明本地代码与所测示例的行为，不代表已部署到学长的服务器，也不证明所有地区的洪水分类都准确。拆分提交、改变代码或处理远程冲突后，应在最终待审核版本重新运行相关检查。

### 快速复核

1. 更新 agent 和 frontend，刷新浏览器，在 Pro 模式打开 FLOOD 图层管理。
2. 检查 Single Inundation Event 可选 2024；Inundation Hotspot 完整周期为 1984–2024、41 年。
3. 在小范围 AOI 请求 2024 年历史图层和 2022–2024 热点图层，检查返回和地图显示。
4. 导出 GEE 代码，核对三个来源、互不重叠的采用年份及 90% 封顶语句。
5. 如审核整版，再按边界说明复核范围选择，并检查切换面板时的影像加载状态。

## 6. GitHub 交接：不是直接执行 git pull

| 操作 | 含义 |
| --- | --- |
| `git pull` | 获取远程更新，并整合进当前本地分支；用于更新本地，不能上传本地修改。见 [Git 官方说明](https://git-scm.com/docs/git-pull) |
| `git commit` | 在本地保存一组可追踪的修改 |
| `git push` | 将本地提交上传到有写权限的远程仓库或分支 |
| PR（Pull Request） | 请求维护者审核并将你的分支合入目标分支；可以查看逐行差异、提交记录、说明和检查结果 |
| Merge | 审核后实际合入目标分支；是否允许合并取决于权限、检查和仓库规则 |

“看情况 PR 合并”通常指：**你提供 PR，学长审核改动、提出意见，再决定是否合并。** 这句话不能单独证明你已经获得仓库写入或合并权限。参考：[GitHub flow](https://docs.github.com/en/get-started/using-github/github-flow)。

### 没有原仓库写权限时

若可以 Fork，通常这样提交：

1. 在 GitHub 把原仓库 Fork 到自己的账号。
2. 在自己的副本创建功能分支，放入整理好的代码和说明文件。
3. 检查差异、commit，再 push 到自己的 Fork。
4. 从自己的分支向原仓库创建 PR。`base` 选择学长指定的接收分支，`compare` 选择自己的修改分支。
5. 将 PR 链接交给学长审核；根据意见继续提交，由有权限的人决定合并。

向自己的 Fork 推送不需要原仓库写权限。如果仓库访问或组织策略不允许 Fork，需要维护者提供可用的协作方式，例如开放合适权限或接收补丁。参考：[从 Fork 创建 PR](https://docs.github.com/en/pull-requests/how-tos/create-pull-requests/creating-a-pull-request-from-a-fork)。

### 当前代码提交前的整理要求

- 本地 `origin`：`https://github.com/BarberHu/SatGPT-app.git`；当前分支：`review/deployment-refactor`；本地基础提交：`cef27424`。这些是本地配置，不代表已核实远程最新状态、目标合并分支或账户权限。
- 当前有多组未提交修改。先保存现有成果、按功能检查差异，再同步目标分支；不要把 `git pull` 当作上传操作。
- 建议按年度数据、行政边界、事件与影像流程分别组织提交或 PR。多个功能修改了同一文件，拆分需要检查具体代码段、依赖和测试，不能只按文件列表全部加入。
- 若整版放在一个 PR，标题和正文应列出各组内容，并附本文件及边界说明。
- 交付代码时保留新增模块、数据来源和测试；不包含 `.env`、服务账号私钥、访问令牌、依赖缓存及本地运行输出。

只发送完整文件夹，学长可以阅读最终代码，但需要自己比较版本才能定位变化。**PR 的逐行差异，加上说明文件和清楚的提交信息，才能更方便地看出改了什么、为什么改。**

## 7. 可复制到“仅年度数据更新”PR 的内容

下面的标题和正文只适用于已整理为年份更新范围的 PR。若提交整版，还需补充第 4 节的其他功能，不能用这个标题概括全部改动。

### 标题

Extend flood historical water layers to JRC annual history through 2024

### 正文

The Single Inundation Event and Inundation Hotspot layers previously stopped at 2021. This change extends their selectable annual history to 2024 using the v1.4 record for 1984–2015, corrected v1.5 classes for 2016–2021, and new v1.5 classes for 2022–2024. The merged record contains 41 unique years.

The backend, slider limits, source descriptions and exported GEE code use the updated annual sources. Other catalog products retain their own coverage limits. The existing permanent-water exclusion, 90% hotspot-frequency cap and legend are preserved. Corrected source data can change results for 2016–2021.

Annual history ends in 2024. Flood Detection remains the separate path for newer event dates, subject to imagery availability. Seasonal-water classification is not proof of an individual flood event.

Validation: 13 relevant frontend tests and 74 backend tests passed in the current local working tree. Live Earth Engine checks confirmed 41 unique years and the retained 90% cap in a Bangkok sample. The local agent and frontend were rebuilt and passed health checks; shared-server deployment has not been established. After isolating this PR, rerun the relevant checks on its final commit.

Details and review scope: `FLOOD_HISTORY_AND_PR_HANDOFF.md`.
