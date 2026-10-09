# 地铁数据管线（metro.html）

`metro.html` 展示的地铁 N×N 通勤时间矩阵是**离线预计算**的静态数据，前端零 API 消耗、秒开。本文件说明数据从哪来、怎么同步、以及前端依赖的数据契约。

## 数据从哪来

```
朋友仓库 output/*.csv（全市三个 CSV + stations_all.csv）
  → python3 tools/build_metro_data.py --city <id> --city-name <中文名> --input <output目录> --output data/<id>
  → data/<id>/{meta.json, stations.json, rows/<组id>.json, lines.json} + 更新 data/index.json
```

数据由 fork 仓库 [Ignareo/shmetro-accessibility](https://github.com/Ignareo/shmetro-accessibility) 的爬虫产出，上游为 [Hecate2/shmetro-accessibility](https://github.com/Hecate2/shmetro-accessibility)。与上游的区别：

- **爬虫与管线能力**：通用管线重构（`metro_accessibility_common.py`）、新增厦门/北京/成都/广州/武汉等城市变体、线路范围过滤、爬取进度与口径守卫、路线质量审计、组代表元爬取等增强；其中通用功能通过 PR 回馈上游，城市专属与本地数据改动留在 fork。
- **数据产出**：fork 的 `output/` 才是本站数据的实际来源，上游仓库本身不维护这些预计算结果；本站只提交转换后的 `data/`，不复制其原始 CSV/SQLite。

> 爬虫需要高德 **Web 服务** key（与页面的 JS API key 不同）。`output/` 的原始 CSV/SQLite **不要**复制进本仓库。

## 同步步骤

1. 在 shmetro-accessibility 项目（fork）中跑爬虫，得到 `output/` 根目录下的 CSV（支持断点续爬）。
2. 运行转换脚本生成本站静态数据：

   ```bash
   python3 tools/build_metro_data.py --city shanghai --city-name 上海 \
       --input <爬虫output目录> --output data/shanghai
   ```

3. 提交 `data/` 目录即可，前端纯静态加载，无需后端。

> 当前 `data/` 已接入真实数据（上海 416 站组、厦门 87 站组，含线路折线 `lines.json`）；`tools/fixtures/shanghai/` 是 12 站演示数据，仅供脚本测试，**不要**拿它覆盖 `data/shanghai/`。

## 转换脚本

`tools/build_metro_data.py` 的输入（`output/` 目录下）：

| 文件 | 用途 |
|---|---|
| `amap_station_matches.csv` | 站点元数据 + GCJ-02 坐标（`status==resolved` 且 `location` 非空才可用） |
| `travel_time_matrix.csv` | N×N 长表（`utf-8-sig`，只保留 `status==done` 的行） |
| `average_time_ranking.csv` | 站点平均通勤时间排行（`average_minutes` 可能为 `NaN`） |
| `stations_all.csv` | 可选：每条线的站点顺序（行序即线路顺序），用于生成地铁线折线 |

输出（`--output` 目录，例如 `data/shanghai/`）：

| 文件 | 内容 |
|---|---|
| `meta.json` | 城市元信息、节点/组/对数、未解析节点列表 |
| `stations.json` | 按站名聚合的分组（坐标均值、线路、排行） |
| `rows/<group_id>.json` | 每组到其他组的分钟数：`{"t": {"g002": 12, ...}}` |
| `lines.json` | 可选：每条线的组 id 有序序列 + 标志色，供前端画 Polyline |

脚本会追加/更新 `data/index.json` 城市清单。线路标志色在脚本的 `CITY_LINE_COLORS` 维护（上海取自官方发布色，厦门为主题色近似值），未收录线路用灰色。

## 前端数据契约（metro.html 依赖，改动需同步两侧）

- 站点按**站名分组**（同站不同线的节点合并），组 id 为 `g001…`；`stations.json` 含每组坐标（GCJ-02）、线路、`avg_minutes`、`rank`。
- `rows/<组id>.json` = `{"t": {目的组id: 分钟整数}}`，按需 fetch，不含自身。
- `lines.json`（可选，旧城市数据可能没有）= `{"lines": [{label, color, groups: [组id 按线路顺序]}]}`，由 `stations_all.csv` 行序生成，前端画 Polyline；前端必须容忍 404（按无线处理）。
- 节点唯一键是 `{line_order}:{station_id}`（`station_id` 跨线路会复用，不能单独用）；`stations_all.csv` 的 `station_id` 是另一套格式，关联只能按 `(line, 站名)`。

## 已知坑

- 爬虫输出的 CSV **可能带 BOM**——读 CSV 一律 `utf-8-sig`。
- 未解析站点（高德 POI 缺失，如厦门 6 号线角美段）会被排除并记入 `meta.json.unresolved_nodes`，属正常现象。
- 口径：矩阵为**工作日早高峰**（默认 07:15 出发，含进出站步行与候车），与 `index.html` 的到达圈（不含等车/换乘步行）含义不同，两页阈值口径靠图例/文案标注区分。

## 边界

- 朋友仓库的改动不进本仓库；若需改爬虫，去朋友仓库改并提 PR，本仓库只提交转换后的 `data/`。
- 上游朋友仓库暂无 LICENSE，公开发布前需与其作者确认数据与代码的使用授权（见 [PLAN.md](../PLAN.md) 第 0 节）。
