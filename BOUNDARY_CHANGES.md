# 行政边界搜索与分析范围修正

日期：2026-10-08。本文说明 Bangkok 边界参考数据及所有地点共用的边界搜索、选择和裁剪流程修改，供代码审查与交接使用。其他功能的改动应单独说明。

## 修改原因

搜索 Bangkok 时，原先使用的 OpenStreetMap 行政范围包含一段较长的离岸延伸。用于陆地洪水分析时，这个范围不符合预期。原实现还将 OSM 结果标为 `official_boundary`，容易让使用者误以为已经核实为官方边界。

检查共用流程后，发现另外几种可能引入错误范围的行为：把地点的外接矩形作为边界、查不到时让模型估画多边形、直接使用多个候选中的第一个，以及在影像裁剪前自动抽稀边界顶点。

本次修改的目的，是让范围选择有来源、歧义有提示、无效数据不能自动用于分析，并保持搜索预览与实际裁剪使用同一完整多边形。**几何检查不等于核实全球最新官方边界；本次没有更换地图底图。**

## 使用者能看到的变化

| 情况 | 修改前 | 修改后 |
| --- | --- | --- |
| 搜索 Bangkok 城市范围 | 使用含离岸延伸的 OSM 轮廓 | 使用随仓库提供的 Bangkok 2017 年陆地参考多边形，同时显示来源与年份 |
| 搜索其他地区 | OSM 多边形可能显示为“官方边界” | 显示为 OpenStreetMap 边界，注明年份未提供及可能包含海域 |
| 只有地点坐标或外接矩形 | 可生成近似矩形范围 | 不作为行政边界候选；提示补充国家/地区，或绘制、上传范围 |
| 查不到边界或查询失败 | 可能调用模型估画多边形 | 返回未解析状态，不生成替代边界 |
| 同名城市、省份等多个结果 | 搜索默认预览第一个，聊天解析默认取第一个 | 搜索需要明确选择，聊天返回需要选择的状态；保留完整地名和行政层级 |
| 无效多边形 | 缺少统一检查 | 拒绝未闭合、自相交、空或零面积图形，以及非有限值、越界坐标 |
| 复杂边界用于影像与下载 | 可能通过抽稀改变边界形状 | 保留原始顶点、岛屿与内部孔洞；过大的请求可能失败，不自动改变范围 |
| 旧的近似搜索范围 | 可能继续进入分析 | 带有近似来源或状态的范围不能通过确认、启动分析 |

## 对应代码

| 文件 | 主要作用 |
| --- | --- |
| [agent/flood_aoi.py](agent/flood_aoi.py) | 搜索和聊天共用行政多边形候选；筛除非行政范围；从实际几何计算边界框；对等价结果去重；多候选时要求选择；取消矩形和模型兜底；查询缓存一小时后过期 |
| [agent/boundary_geometry.py](agent/boundary_geometry.py)（新增） | 用 Shapely 检查多边形合法性；识别等价几何；拆解单个 AOI 的 GeoJSON 包装并保留原始坐标；识别旧的近似搜索范围 |
| [agent/boundary_reference.py](agent/boundary_reference.py)（新增） | 根据精确别名或稳定的 OSM 对象身份匹配参考数据；当前收录 Bangkok，不按地名子字符串替换 |
| [agent/data/boundaries/bangkok.geojson](agent/data/boundaries/bangkok.geojson)（新增）及[数据说明](agent/data/boundaries/README.md) | 保存 Bangkok 多边形、来源、年份、许可与处理记录 |
| [LocationScopePicker.jsx](frontend/src/components/LocationScopePicker.jsx)及[样式](frontend/src/components/LocationScopePicker.css) | 显示来源、年份和层级；多候选不自动预览；选择时预览对应范围；导入时保留完整地名、原始几何与来源信息；忽略过期搜索响应 |
| [agent/flood_agent.py](agent/flood_agent.py) | 确认和处理节点拦截近似范围；解析存在歧义时说明候选；旧的坐标解析入口也转到共用解析器 |
| [EventConfirmation.jsx](frontend/src/components/EventConfirmation.jsx)、[aoi.js](frontend/src/utils/aoi.js)、[floodWorkflow.js](frontend/src/utils/floodWorkflow.js) | 前端确认与分析启动同样拦截近似或不可分析的范围 |
| [agent/flood_api_services.py](agent/flood_api_services.py)、[agent/server.py](agent/server.py) | 影像和栅格下载使用完整多边形；无效影像 AOI 返回输入错误；兼容保留的 `thin_geojson_geometry` 名称现在执行检查与拆解，不再抽稀 |
| [依赖声明](agent/requirements.txt)、[依赖锁定文件](agent/requirements.lock.txt) | 增加 Shapely，生产锁定版本为 2.1.2 |

## Bangkok 数据来源与适用范围

随仓库记录的来源为 **Royal Thai Survey Department / HDX Thailand administrative boundaries**，经 geoBoundaries **gbHumanitarian** 分发，使用其中的 Bangkok ADM1 要素：

- 代表年份：**2017**；取得日期：2026-10-08。
- 数据许可：**CC BY 3.0 IGO**。
- 多边形坐标保持不变；只提取对应要素并补充说明属性，没有手工沿海岸线切割。
- 精确的 Bangkok 城市别名，以及 OSM relation `92277`，使用这份参考数据。
- 该多边形不是更广的 Bangkok Metropolitan Region，也不会按名称子字符串替换 Bangkok Noi 等区级范围。
- 原始发布页、固定版本下载链接及完整归属说明见[边界数据 README](agent/data/boundaries/README.md)。

这是一份有来源的历史参考数据，不能据此声称已经反映 2017 年以后所有行政或海岸变化。

## 验证记录

该边界修改完成时（2026-10-08）的验证记录：前端 **25 个测试文件、92 项测试**通过；后端 **74 项测试**通过；前后端 Docker 构建通过；本地 `satgpt` 的 agent 与 frontend 已更新。后续代码变化需要重新运行相关检查，这些数量不是对未来版本的承诺。

主要回归用例：

- [test_flood_aoi_bounds.py](agent/tests/test_flood_aoi_bounds.py)：Bangkok 搜索与分析使用相同参考几何；拒绝坐标点、非行政图形和无效多边形；保留岛屿、孔洞；区分同名区域；网络失败时不估画边界。
- [test_deployment_merge.py](agent/tests/test_deployment_merge.py)：无效影像 AOI 在请求服务前返回错误；Bangkok 的完整复杂几何传入影像服务，没有抽样删除顶点。
- [test_flood_workflow.py](agent/tests/test_flood_workflow.py)：旧的近似范围不能确认或处理；歧义提示包含实际候选名称。
- [LocationScopePicker.test.jsx](frontend/src/components/LocationScopePicker.test.jsx)：来源说明、预览与导入一致、城市/省份选择、近似候选禁用，以及过期搜索响应拦截。
- [floodWorkflow.test.js](frontend/src/utils/floodWorkflow.test.js)：导入后仍带有近似来源或状态的范围不能启动分析。

运行中的搜索服务还实测了 Bangkok、Chiang Mai、Dushanbe、Zhengzhou、Jakarta 和 Singapore：Bangkok 返回陆地参考范围，清迈保留市与省两个候选，杜尚别去除了非行政候选，其他结果通过几何检查。运行中的聊天解析也确认了 Bangkok 可解析、清迈需要选择。

上述实测覆盖候选结果、地图预览和解析行为，没有逐一核对六个地区的最新官方边界，也没有增加对真实 GEE 洪水影像结果的完整验证。

## 如何复核和更新

更新现有、本地 Compose 项目 `satgpt` 时，在项目根目录运行：

```powershell
docker compose -p satgpt --env-file .env -f scripts/docker/compose.yml build agent frontend
docker compose -p satgpt --env-file .env -f scripts/docker/compose.yml up -d --no-deps agent frontend
```

本次包含后端依赖和前端修改，需要同时更新 agent 与 frontend。其他运行环境按[项目部署说明](README.md#deployment-and-startup)安装或重建依赖。浏览器刷新后，再重新搜索、添加范围；已保存的范围不会自动替换。

建议复核以下行为：

1. 搜索 `Bangkok`：来源显示 Royal Thai Survey Department / HDX 和 2017；点击候选后检查轮廓，确认没有原先伸入海中的长条。
2. 搜索 `Chiang Mai, Thailand`：能看到城市与省两个结果；必须明确选择，并确认加入的完整地名与所选结果一致。
3. 将一个范围加入图层后再用于分析：预览和导入应保留相同几何；完整多边形传入影像处理的情况由上述接口回归用例验证。
4. 边界缺失、无效及旧近似范围的情况，运行对应自动测试，确认分析不会自动启动。

前端完整测试可在 `frontend` 目录运行 `npm test`。构建后端镜像后，可在项目根目录用隔离容器复核后端测试；以下测试设置不使用真实模型密钥，且禁止网络访问：

```powershell
docker run --rm --network none -e PYTHONPATH=/app:/app/agent -e LLM_MODEL=test-model -e OPENAI_API_KEY=test-key -e OPENAI_API_BASE=http://localhost:1 --entrypoint python satgpt-agent:latest -m unittest discover -s /app/agent/tests -q
```

## 仍需注意的范围限制

- 几何合法、候选唯一和有来源信息，都不能单独证明边界符合最新官方行政数据。
- 其他地区仍主要使用 OSM；沿海行政范围可能合法包含领海。它与纯陆地洪水分析范围不同，需要合适的陆地参考数据或用户提供的范围。
- OSM 行政层级编号按原始结果显示，不能直接认为不同国家的相同编号代表同一级别。
- 已保存、上传、绘制或编辑的范围不会自动迁移；旧近似范围可能需要重新搜索或替换后才能分析。
- 多要素 FeatureCollection 不能在影像处理中被悄悄丢弃到只剩第一个要素；需先合并为一个合适的 Polygon/MultiPolygon。
- 保留完整几何可能增加处理时间，超大 AOI 也可能达到 Earth Engine 的请求限制。

发布代码时，需一并包含新增的 Python 模块、Bangkok GeoJSON、数据来源说明、依赖文件和测试。只更新旧文件而遗漏新增资源，会导致运行失败或无法使用参考边界。
