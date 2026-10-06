# PLAN：融合 shmetro-accessibility 地铁可达性数据

> 来源项目：https://github.com/Hecate2/shmetro-accessibility
> 目标：把其本地算好的地铁 N×N 通勤时间矩阵接入本工具，纯前端展示，无后端。

## 0. 改动归属边界（重要）

融合工作分属两个仓库，**不要混淆**：

### 本项目（CommuteTime）内的改动 —— 已落地

| 文件 | 说明 |
|---|---|
| `metro.html` | 新页面：全城可达性热力图 + 单站出发等值圈（已实现，已浏览器验证） |
| `tools/build_metro_data.py` | 转换脚本：爬虫 CSV → 前端静态 JSON（纯标准库） |
| `tools/fixtures/shanghai/` | 12 站演示 fixture（供开发联调，非真实数据） |
| `data/shanghai/`、`data/index.json` | 由脚本生成的演示数据；接真实数据后重跑脚本覆盖 |
| `assets/metro-view*.png`、`README.md` | 截图与文档 |

### 朋友项目（git clone / pull 到本地后）—— 原则上**不改代码**，只做配置与操作

- `.env` 配置高德 **Web 服务** key + 数字签名私钥（✅已做）。
- 上海需先跑 `shmetro_accessibility_legacy.py` 生成 `stations_all.csv` 站点目录。
- 先 `--resolve-only` 验证站点匹配质量（看 `amap_station_matches.md`），再 `--max-routes 80` 冒烟，最后长跑全量。
- 用 `--date/--time` 指定非节假日工作日的早高峰出发时间（默认 7:15）。
- 断点续爬直接重跑同一命令；`--compute-only` 可离线重建 CSV。
- **若确需改朋友项目代码**（如某站匹配不到时调整 `candidate_score` 规则、新增城市）：在朋友仓库的 clone 里改，向上游提 PR，不要把改动混进本仓库。
- 朋友项目的 `output/` 原始 CSV/SQLite **不进本仓库**；本仓库只提交转换后的 `data/`。
- 朋友仓库暂无 LICENSE，公开发布本仓库前与作者确认数据与代码的使用授权。

## 1. 背景与可行性结论

**结论：可行，且互补性好。** 本工具的地铁等时圈来自高德 ArrivalRange，不含等车与换乘步行时间；对方项目的矩阵恰好是含候车 + 换乘步行的真实早高峰时间（默认工作日 7:15 出发），是对现有功能的升级校正。

关键事实：

- 对方输出为本地 CSV/SQLite，静态数据可直接随页面部署，无需后端。
- 站点坐标为 GCJ-02（`amap_station_matches.csv` 的 `location` 列），与高德 JS API 天然匹配，零纠偏。
- 对方仓库无任何前端输出（无 GeoJSON），需要自己做数据转换。
- 站点是「线路 × 站名」节点：同站不同线算不同节点，坐标刻意匹配不同出入口。前端按站名聚合成组显示，组内保留各节点 id。
- 站点 ID 跨城市格式不一致（上海为数字码，深圳/武汉为 `02-pinyin` 式），前端一律当不透明 key；节点唯一键为 `{line_order}:{station_id}`。

## 2. 数据规模与成本

| 事项 | 量级 |
|---|---|
| 爬数据 | 上海约 500 节点，N×N 有向对约 25 万条；单 key 限流 3.01 QPS ≈ 23 小时，3 个 key ≈ 8 小时。支持 SQLite 断点续爬、`--compute-only` 离线重建 |
| 前端数据体积 | 只保留 `status=done` 的行，时间量化为整数分钟；按站分组、按 origin 分片，首屏零矩阵加载 |

## 3. 数据管线（已实现）

```
shmetro-accessibility/output/<城市>/
  ├─ amap_station_matches.csv   (站点元数据 + GCJ-02 坐标)
  ├─ travel_time_matrix.csv     (长表: from_id,to_id,status,duration_seconds,...,utf-8-sig)
  └─ average_time_ranking.csv   (rank,station_id,...,average_minutes,sample_size)
        │
        ▼  tools/build_metro_data.py --city shanghai --city-name 上海 --input <output目录>
data/<城市>/
  ├─ meta.json       城市名、节点/组数、pair 数、生成日期、口径、unresolved_nodes
  ├─ stations.json   groups[]: {id(g001...), name, lng, lat, lines[], avg_minutes, rank, nodes[]}
  └─ rows/<组id>.json  {"t": {目的组id: 分钟整数}}，点击时按需加载
data/index.json      城市清单（重跑幂等，按 city 更新条目）
```

转换规则（脚本已实现）：

- 只保留 `status == "done"` 的记录；`duration_seconds / 60` 四舍五入为整数分钟。
- 同名不同线节点按站名聚合为组：坐标取均值，avg_minutes 取组内均值（排除 NaN），rank 取组内最小。
- 组间时间 = 组内 origin 节点 × 目的组节点的最小值；rows 不含自身。
- 坐标缺失/未解析的节点记入 `meta.json` 的 `unresolved_nodes`。

## 4. 前端展示方案（视图一、二已实现于 metro.html）

### 视图一：全城可达性热力图（默认视图）

- 选城市 → 加载 `stations.json`，`AMap.MassMarks` 绘制全部站点组，按 avg_minutes 分档：≤40 绿、≤50 黄、≤60 橙、>60 红、无数据灰。
- 侧栏排行榜（可排序、搜索过滤、点行定位高亮），悬浮/点击站点弹 InfoWindow。

### 视图二：单站出发的真实等时圈

- 点击站点 → 加载 `rows/<组id>.json`，其余站点按 ≤15 深绿/≤30 绿/≤45 黄/≤60 橙/>60 红上色。
- 按 15/30/45/60 分钟四档画等值圈（Andrew 凸包 + 质心外扩 12% 缓冲，不足 3 点跳过）。
- 面板显示出发站信息、四档可达站数统计、按时间升序的可达站列表。

### 视图三（后续阶段）：打通现有「批量算通勤」

- 候选点 → 最近地铁站（直线距离估步行）+ 矩阵查站间时间 + 出站步行 → 秒级出结果、零 API 消耗。
- 作为批量通勤的「本地估算」模式，与调高德 API 的精确模式并存。

### 口径声明（页面上已注明）

数据为「工作日早高峰出发、含进出站步行与候车」的静态快照，与实时查询存在差异；节假日时刻表未纳入。

## 5. 实施步骤与进度

1. [ ] 克隆朋友仓库，`.env` 配置 Web 服务 key + 数字签名私钥。
2. [ ] 用 legacy 脚本生成 `stations_all.csv`，跑 `--resolve-only` 验证站点匹配质量。
3. [ ] 全量爬取矩阵（先 `--max-routes 80` 冒烟，再长跑）。
4. [x] 编写 `tools/build_metro_data.py` 转换脚本（含 fixture 与断言校验）。
5. [x] 新建 `metro.html`，实现视图一（热力图 + 排行榜）。
6. [x] 实现视图二（单站网络热力 + 分档等值圈）。
7. [ ] 用真实爬虫输出替换 `data/shanghai/` 演示数据（重跑步骤 4 的命令）。
8. [x] README 增加地铁可达性章节，注明数据来源与作者出处。
9. [ ] （后续）视图三：候选点本地通勤估算，接入 `index.html` 的批量算通勤。

## 6. 待确认决策点

- 首个城市按上海推进（fixture 已就绪）；其他城市等城市数据爬完后按同一管线扩展。
- 视图三待真实数据到位后再启动。
- 矩阵分片格式已定为每组一个 JSON 文件；若单城文件数过多再评估合并为二进制。
