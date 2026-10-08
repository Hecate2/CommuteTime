# AGENTS.md — 面向 LLM 协作指南

## 项目是什么

等时圈通勤工具，纯静态前端（无构建、无后端），基于高德地图 JS API 2.0。两个页面：

- `index.html` — 主工具：设起点、画等时圈、圈内检索候选地点、批量算真实通勤时间（调高德 API）。
- `metro.html` — 地铁可达性可视化：展示由 [shmetro-accessibility](https://github.com/Hecate2/shmetro-accessibility) 项目本地预计算的地铁 N×N 通勤时间矩阵。详见 `PLAN.md`。

## 代码约定

- 单文件页面：HTML + 内联 CSS/JS，除高德 JS API 外**零第三方依赖**，不要引入框架/构建工具。
- JS 用 `var` + `function` 风格，与 `index.html` 一致；UI 文案为中文。
- 密钥加载逻辑（`config.yaml` 解析 → localStorage → 弹窗兜底）在两个页面间保持一致的写法；`metro.html` 无密钥时可跳过弹窗纯数据浏览（地图区显示占位层，排行榜/单站列表可用），有密钥才初始化高德地图。
- 提交信息用中文、带 `feat:`/`fix:` 等前缀，正文列要点。
- 配色约定：通勤时间分档绿/黄/橙/红（index.html：≤30/45/60 分钟，为等时圈设定时长；metro.html 全城视图：≤40/50/60 全市平均通勤，单站视图：≤15/30/45/60 从该站出发）。两页阈值含义不同，靠图例/文案标注口径而非统一数值。

## 数据管线（metro.html 的数据从哪来）

```
朋友仓库 output/<城市>/*.csv
  → python3 tools/build_metro_data.py --city <id> --city-name <中文名> --input <output目录> --output data/<id>
  → data/<id>/{meta.json, stations.json, rows/<组id>.json} + 更新 data/index.json
```

前端数据契约（metro.html 依赖，改动需同步两侧）：

- 站点按**站名分组**（同站不同线的节点合并），组 id 为 `g001…`；`stations.json` 含每组坐标（GCJ-02）、线路、avg_minutes、rank。
- `rows/<组id>.json` = `{"t": {目的组id: 分钟整数}}`，按需 fetch，不含自身。
- 节点唯一键是 `{line_order}:{station_id}`（station_id 跨线路会复用，不能单独用）。

已知坑：

- 爬虫输出的 CSV **可能带 BOM**——读 CSV 一律 `utf-8-sig`。
- 未解析站点（高德 POI 缺失，如厦门 6 号线角美段）会被排除并记入 `meta.json.unresolved_nodes`，属正常现象。
- `tools/fixtures/shanghai/` 是 12 站演示数据，不是真实爬取结果；`data/shanghai/` 当前由它生成。

## 边界与禁区

- **朋友仓库**（通常 clone 在 `../shmetro-accessibility`）的改动不进本仓库；本仓库只提交转换后的 `data/`。若需改爬虫，去朋友仓库改并提 PR。
- `config.yaml` 含真实高德密钥，工作区里的密钥改动一般属于用户本人，**不要代提交**；也不要在公开部署中泄露。
- 朋友项目的 `output/` 原始 CSV/SQLite 不要复制进本仓库。

## 验证方式

改 `metro.html` / 数据后：

```bash
python3 -m http.server 8931 --bind 127.0.0.1
# 浏览器打开 http://127.0.0.1:8931/metro.html
```

验证点：城市下拉框出现新城市、排行榜有数据、单击排行行末「›」（或散点 InfoWindow 里「查看单站视图 ›」）进单站视图（等值圈、可达列表正常）；无密钥时弹窗可跳过、纯数据浏览正常。两页移动端（≤768px）面板为底部抽屉，grabber 按钮循环三档高度。注意底图瓦片要求高德 key 的域名白名单包含当前 origin，否则报 `INVALID_USER_DOMAIN`（站点数据层不受影响，属 key 配置问题而非代码问题）。
