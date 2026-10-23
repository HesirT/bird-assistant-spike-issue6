# 观鸟助手 API 与数据源调研

> 调研日期：2026-10-23  
> 适用场景：非商业课程项目、原型演示；若项目公开上线、商业化或扩大数据规模，必须重新核对最新条款。  
> 本文是技术与数据合规调研，不构成法律意见。

## 1. 结论先行

推荐采用“一个基础名录、两个实时观测源、一个历史聚合源”的组合：

- **鸟种基础信息**：以 [AviList v2026](https://www.avilist.org/checklist/v2026/) 作为全球鸟类分类学主表；用[中国生物物种名录 2026](http://www.sp2000.org.cn/)补充中文名和中国分布；用 [eBird Taxonomy](https://science.ebird.org/en/use-ebird-data/the-ebird-taxonomy/) 保存 `speciesCode`；需要简介、跨库 ID 和带授权图片时，可接 [BirdNET+ Taxonomy](https://birdnet.cornell.edu/taxonomy/docs)。
- **附近近期记录**：以 [eBird API v2](https://documenter.getpostman.com/view/664302/S1ENwy59) 为主，以 [iNaturalist API v1](https://api.inaturalist.org/v1/docs/) 补充。两者都能按位置、日期和物种查询，但数据质量、许可和敏感坐标规则不同。
- **历史记录和大范围分析**：使用 [GBIF Occurrence API](https://techdocs.gbif.org/en/openapi/v1/occurrence)；查询量大时改用带 DOI 的 GBIF Download，不要翻页抓取完整数据集。
- **地图**：国内课程演示优先考虑高德地图，但其返回数据原则上不得自行缓存、抽取或脱离服务展示；若使用 OpenStreetMap，必须遵守瓦片和 Nominatim 公共实例的严格限流政策。
- **天气**：非商业原型优先 [Open-Meteo](https://open-meteo.com/en/docs)，国内可用性要求较高时使用[和风天气](https://dev.qweather.com/docs/)。
- **国内观鸟社区**：目前未找到“中国观鸟记录中心/鸟类记录中心”等平台公开、稳定的开发者 API。未经书面授权，不抓网页、不逆向 App、不把“公开可见”理解成“允许批量复制”。
- **敏感鸟种**：不能只依赖上游已经模糊过的坐标。应在后端统一执行二次保护，并同时处理坐标、时间、地点文字、照片 EXIF、观察者轨迹和导航能力。

### 1.1 推荐的数据流

```text
AviList / 中国生物物种名录 / BirdNET+ Taxonomy
                         │
                         ▼
                 本地物种与跨库 ID 表
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
      eBird          iNaturalist         GBIF
   近期鸟类记录       补充公民科学记录     历史/馆藏聚合
        └────────────────┼────────────────┘
                         ▼
           去重、许可检查、敏感位置脱敏
                         ▼
            地图展示 + 天气 + 来源标注
```

不要直接把第三方原始响应透传给前端。API 密钥、许可筛选、缓存、限流和敏感位置处理都应放在项目后端。

## 2. 总览对比

| 数据源 | 主要用途 | 官方 API | 位置 / 时间 / 物种查询 | 密钥 | 已公开限制 | 课程展示与缓存结论 |
|---|---|---|---|---|---|---|
| eBird | 近期鸟类观测、热点、名录 | 有，v2 | 均支持；附近近期接口最多回看 30 天、半径最多 50 km | 大多数接口需要 | 未公布固定 QPS；可限制有害或过量访问 | 非商业教育用途通常可；必须署名。API 缓存没有明确通用授权，宜短期内部缓存，不做镜像 |
| iNaturalist | 公民科学观测、分类、照片/声音 | 有，v1 | 均支持 | 公共 GET 不需要；写入/私有数据需要 OAuth/JWT | 硬上限约 100 次/分钟；建议不超过 60 次/分钟和约 10,000 次/日 | 可展示，但每条观测和每个媒体文件分别检查许可；缓存必须保留许可和出处 |
| GBIF | 全球历史观测、标本、聚合检索 | 有 | 均支持 | 搜索不需要；批量下载需账号 | 单页最多 300，搜索窗口最多 100,000；动态限流，429 时退避 | 依每条记录所属数据集的 CC 许可；下载应引用 DOI |
| AviList v2026 | 全球鸟类分类主表 | 无 REST API；有 XLSX/CSV | 不适用 | 不需要 | 版本化整包下载 | CC BY 4.0，可本地缓存和再利用，需署名 |
| BirdNET+ Taxonomy | 简介、多语言名、跨库 ID、图片入口 | 有 | 按名称、类群搜索；无观测查询 | 当前公开 GET 不需要 | 未公布数值上限 | 可缓存官方版本化数据；上游描述和图片许可不同，须逐字段保留来源 |
| 中国生物物种名录 | 中文名、中国分布和分类核对 | 有 API，也有年度下载 | 按学名、中文名、科、代码；无观测查询 | API 需要个人 Key | 未公布统一数值上限 | 年度名录可按官方说明下载使用并引用；若再分发数据库或大量调用应确认授权 |
| IUCN Red List | 受威胁等级和评估 | 有，v4 | 主要按分类单元和评估查询 | 需要申请 Token | 未公布统一数值；禁止抓取式使用 | 条款严格，仅在确有保护教育目的并获准时使用；不是附近观测源 |
| OSM 瓦片 / Nominatim | 底图、逆地理编码 | 有公共实例 | 按位置；不查询鸟类 | 不需要 | Nominatim 绝对上限 1 req/s；公共瓦片禁止批量预取 | ODbL/署名；瓦片须按响应头缓存，Nominatim 要缓存；不适合无节制生产流量 |
| 高德开放平台 | 国内底图、坐标转换、逆地理编码、天气 | 有 | 按位置；不查询鸟类 | 需要 Key | 按产品、认证和套餐变化，以控制台为准 | 学习用途可在授权服务内展示；不得擅自缓存、抓取、建立衍生地图库 |
| Open-Meteo | 天气预报 | 有 | 按坐标和时间；不查询鸟类 | 非商业免费接口不需要 | 免费层：600/min、5,000/h、10,000/day、300,000/month | 非商业课程可用；CC BY 4.0，需署名，可合理缓存 |
| 和风天气 QWeather | 国内天气和预警 | 有 | 按坐标/城市和时间；不查询鸟类 | JWT 或 API Key | 套餐计量；天气类每月前 50,000 次免费额度以当前价目为准 | 可在条款内展示；必须署名；天气允许按官方 TTL 缓存，GeoAPI 不允许缓存 |
| 国内观鸟社区 | 国内观测记录 | 未找到公开开发者 API | 不确定 | 不适用 | 无公开说明 | 没有书面授权就不集成、不抓取、不缓存、不再展示位置数据 |

## 3. 鸟种基本信息与开放名录

### 3.1 AviList：建议作为分类学主表

[AviList](https://www.avilist.org/) 是当前国际统一鸟类名录。调研时最新版本为 **v2026，发布于 2026-10-20**。其下载页提供扩展版、简版和 Explorer CSV，不提供公开 REST API。

可用于：

- 有效学名、属、科、目等分类层级；
- 英文名、同物异名、分类变化和分布摘要；
- 与 eBird/Clements、BirdLife 等外部体系的映射；
- 给本项目生成稳定、可版本化的本地鸟种表。

许可为 **CC BY 4.0**，因此可以在课程项目中复制、转换和缓存，但必须标注来源及版本。推荐引用：

> AviList Core Team. 2026. AviList: The Global Avian Checklist, v2026. https://doi.org/10.2173/avilist.v2026

实现建议：

1. 下载 v2026 文件，作为只读源数据保存；
2. 导入本地数据库时保留 `source_version = "AviList v2026"`；
3. 不用中文名或学名直接做永久主键，使用项目自己的 `taxon_id`；
4. 把 eBird `speciesCode`、iNaturalist `taxon_id`、GBIF `taxonKey` 保存为映射字段；
5. 每次更新名录都做迁移表，因为拆分、合并和学名变更会影响历史记录。

IOC World Bird List 的最终独立版本为 v15.2，项目方已转向 AviList；新项目不建议再以 IOC 建主表。参见 [IOC World Bird List 官方说明](https://www.worldbirdnames.org/)。

### 3.2 BirdNET+ Taxonomy：适合物种详情补全

[BirdNET+ Taxonomy API 文档](https://birdnet.cornell.edu/taxonomy/docs) 提供多语言名称、简介、外部数据库 ID、图片及其来源信息。鸟类分类以 AviList 为基础，但调研时公开数据版本为 **v0.3-Jul2026**，早于 AviList v2026，不能假设二者已经同步。

常用请求：

```bash
# 物种详情；可请求多个语言
curl 'https://birdnet.cornell.edu/taxonomy/api/species/Passer%20montanus?locale=zh,en'

# 搜索鸟类
curl 'https://birdnet.cornell.edu/taxonomy/api/search?q=麻雀&group=Aves&locale=zh&per_page=20'

# 分页列出鸟类
curl 'https://birdnet.cornell.edu/taxonomy/api/species?group=Aves&per_page=500&format=json'

# 下载完整 JSON
curl -O 'https://birdnet.cornell.edu/taxonomy/api/download/json'
```

返回中可包含 `scientific_name`、本地化名称、简介、eBird/iNaturalist/GBIF 等 ID，以及图片的 `source`、`author`、`license`。

许可注意：

- BirdNET+ Taxonomy 项目代码采用 MIT 许可，但聚合内容并不自动全部变成 MIT；
- AviList 字段为 CC BY，Wikidata 常为 CC0，Wikipedia 描述通常为 CC BY-SA；
- 图片可能来自 iNaturalist、Macaulay Library 等，许可可能是 CC、限制非商业使用，甚至保留全部权利；
- 展示或缓存媒体时必须使用响应中的作者、来源和许可。课程项目应优先筛选明确允许的 CC 图片，不把“能返回图片 URL”当作转载授权。

API 没有公开数值限流。由于提供官方整包下载，分类静态数据宜按版本下载后本地使用，不要在每次页面打开时逐物种请求。

### 3.3 中国生物物种名录：中文和国内分布补充

[中国生物物种名录](http://www.sp2000.org.cn/)由中国科学院生物多样性委员会发布。2026 版可免费下载使用，适合补充中文名、异名、中国分布和文献，不是观测记录源。

[API 文档](http://www.sp2000.org.cn/api/document)要求注册后使用个人 `apiKey`。常见 v2 接口形态：

```text
GET http://www.sp2000.org.cn/api/v2/getSpeciesByScientificName
    ?apiKey=YOUR_KEY&scientificName=Passer%20montanus&page=1

GET http://www.sp2000.org.cn/api/v2/getSpeciesByCommonName
    ?apiKey=YOUR_KEY&commonName=树麻雀&page=1

GET http://www.sp2000.org.cn/api/v2/getNameByKeyword
    ?apiKey=YOUR_KEY&keyword=柳莺&page=1
```

还可按科 ID、科名和物种 NameCode 查询，具体参数以官方文档为准。该服务目前未公开统一的 QPS、日限额或通用缓存条款。

建议优先使用官方年度下载包并固定版本；若要把其数据再发布为自己的 API、长期镜像或大量自动请求，应先向平台确认。动物名录的官方引用格式可在[引用说明](http://www.sp2000.org.cn/info/info_how_to_cite)查看。

注意：其官网目前主要使用 HTTP，浏览器或小程序可能因混合内容/安全域名策略无法直接访问，因此应由后端下载或代理，且不能把 Key 放入前端。

### 3.4 eBird Taxonomy：用于连接 eBird 观测

接口：

```bash
curl -H 'X-eBirdApiToken: YOUR_TOKEN' \
  'https://api.ebird.org/v2/ref/taxonomy/ebird?fmt=json&locale=zh_SIM'
```

它能返回 eBird 使用的 `speciesCode`、学名、常用名、分类顺序和类别。项目查询 eBird 时应使用 `speciesCode`，不要靠中文名拼接 URL。

调研时 eBird 科学页面仍把 eBird Taxonomy v2025/Clements v2025 标为当前版本，而 AviList v2026 刚发布。因此要记录每个映射所对应的版本，不要假设同名物种在各库中一一对应。详见 [eBird Taxonomy](https://science.ebird.org/en/use-ebird-data/the-ebird-taxonomy/)和[eBird 常用名与语言](https://support.ebird.org/en/support/solutions/articles/48000804865)。

### 3.5 IUCN Red List API：可选，不建议作为 MVP 必需依赖

[IUCN Red List API v4](https://api.iucnredlist.org/api-docs/index.html)提供全球受威胁等级和评估内容，需要注册并申请 Token。它不是观测 API，不能回答“附近有什么鸟”。

IUCN API 的非商业限制和内容使用规则比普通开放数据严格；其开发者说明还特别提醒移动应用、课程计算项目或与保护无关的可视化可能需要额外许可。若课程确实要展示保护等级，应在申请中写清教育目的、展示字段、用户规模和缓存方式，并以批准结果为准。仅为了显示等级时，也可先使用 AviList 中已有的 IUCN 类别映射，减少额外依赖。

官方入口：[API](https://api.iucnredlist.org/)；[使用条款](https://www.iucnredlist.org/terms/terms-of-use)。

## 4. eBird API v2

### 4.1 能提供什么

eBird 是鸟类专用数据源，适合：

- 用户当前位置附近最近出现过的鸟；
- 指定鸟种在附近的近期记录；
- 稀有/值得注意的观测；
- 地区某一天的历史观测；
- 热点、清单详情和地区物种列表；
- eBird 分类表和地区代码。

它的 API 是面向近期、摘要型和有限结果集设计的，不等同于完整 eBird 数据库。大规模研究应使用 [eBird Basic Dataset 下载](https://support.ebird.org/en/support/solutions/articles/48000838205-download-ebird-data)，并遵守单独的数据访问条款。

### 4.2 认证

在 [eBird API Key 页面](https://ebird.org/api/keygen)登录并申请个人 Token。请求头：

```http
X-eBirdApiToken: YOUR_TOKEN
```

Token 与账号绑定，不应提交到 Git、放入小程序包或返回给浏览器。通过项目后端代理请求。

### 4.3 常用调用

基础地址：`https://api.ebird.org/v2`

```bash
# 上海人民广场附近 10 km、最近 14 天，每种鸟返回最新一条
curl -H 'X-eBirdApiToken: YOUR_TOKEN' \
  'https://api.ebird.org/v2/data/obs/geo/recent?lat=31.2304&lng=121.4737&dist=10&back=14&maxResults=100&sppLocale=zh_SIM'

# 附近近期值得注意的记录
curl -H 'X-eBirdApiToken: YOUR_TOKEN' \
  'https://api.ebird.org/v2/data/obs/geo/recent/notable?lat=31.2304&lng=121.4737&dist=10&back=14'

# 指定物种附近近期记录；路径参数是 eBird speciesCode
curl -H 'X-eBirdApiToken: YOUR_TOKEN' \
  'https://api.ebird.org/v2/data/obs/geo/recent/eutspa?lat=31.2304&lng=121.4737&dist=20&back=30'

# 指定地区、指定日期的历史记录
curl -H 'X-eBirdApiToken: YOUR_TOKEN' \
  'https://api.ebird.org/v2/data/obs/CN-31/historic/2026/10/20?maxResults=100'

# 附近热点
curl -H 'X-eBirdApiToken: YOUR_TOKEN' \
  'https://api.ebird.org/v2/ref/hotspot/geo?lat=31.2304&lng=121.4737&dist=25&back=30&fmt=json'
```

附近近期接口的重要限制：

- `dist`：0–50 km，默认 25 km；
- `back`：1–30 天，默认 14 天；
- `maxResults`：1–10,000；
- 一个结果通常代表该物种最近的一次观测，不是所有观测事件；
- 默认不包含未确认的 provisional 记录；是否展示可由 `includeProvisional` 控制；
- 任意日期范围不是一个通用参数。近 30 天用 recent 接口；某一天用 historic；更长区间或完整序列考虑 GBIF/EBD。

典型响应字段：

```json
{
  "speciesCode": "eutspa",
  "comName": "Eurasian Tree Sparrow",
  "sciName": "Passer montanus",
  "locId": "L123456",
  "locName": "Example Hotspot",
  "obsDt": "2026-10-20 08:15",
  "howMany": 12,
  "lat": 31.23,
  "lng": 121.47,
  "obsValid": true,
  "obsReviewed": false,
  "locationPrivate": false,
  "subId": "S123456789"
}
```

完整端点和字段应查 [eBird API v2 官方文档](https://documenter.getpostman.com/view/664302/S1ENwy59)。

### 4.4 限流、许可、缓存和署名

- **频率**：官方没有公布固定 QPS 或日额度，但可暂停影响服务稳定性、滥用或违反条款的 Token。项目自己的保守策略可设为每个上游约 1 req/s、遇到 429 指数退避；这是工程建议，不是官方配额。
- **课程展示**：[API 使用条款](https://www.birds.cornell.edu/home/ebird-api-terms-of-use/)允许非商业的个人、教育、研究和保护用途。商业使用须书面许可。
- **署名**：界面中至少显示“数据来源：eBird，Cornell Lab of Ornithology”，链接到 [eBird](https://ebird.org/)。引用研究数据时按[eBird 引用指南](https://science.ebird.org/en/use-ebird-data/citation)执行。
- **缓存**：API 条款没有给出统一 TTL 或明确授权建立持久副本。课程项目可为减少重复调用做短期、内部、按查询键缓存，例如 10–30 分钟，但不要公开原始缓存、做数据镜像或长期批量积累；需要长期保存时向 eBird 确认。
- **再分发**：[eBird 数据访问条款](https://www.birds.cornell.edu/home/ebird-data-access-terms-of-use/)对原始数据集再分发有约束。EBD 下载数据不能因为是课程项目就任意打包给第三方。
- **媒体**：API 中出现一个观测不代表该观测关联的照片/录音可以复制。Macaulay Library 媒体需按各自授权使用，eBird 名称和 Logo 也不能擅自当作项目品牌。

### 4.5 敏感物种

[eBird 敏感物种规则](https://support.ebird.org/en/support/solutions/articles/48000803210)会隐藏站点级精确位置，并以约 **400 km²** 的网格或区域化形式展示部分记录；某些输出还会完全省略细节。

应用侧必须：

- 尊重 `locationPrivate` 和上游已经概化的坐标；
- 不通过清单、观察者其他记录、地点文字或时间推断真实点位；
- 不把模糊坐标作为可导航目的地；
- 对本地敏感名单再做一层展示控制。

## 5. iNaturalist API v1

### 5.1 能提供什么

iNaturalist 覆盖所有生物类群，适合补充 eBird 没覆盖的用户群和带照片的公民科学记录。它不是鸟类专用名录，质量等级、物种鉴定和许可都要明确展示。

使用新版 `https://api.inaturalist.org/v1`；旧 API 除 OAuth 等少数用途外已经不应作为新开发基础。官方资料：[API Reference](https://www.inaturalist.org/pages/api+reference)、[Developer 页面](https://www.inaturalist.org/pages/developers)。

### 5.2 认证

- 公共观测和分类 GET 请求不需要 Key；
- 创建/修改记录、访问用户私有信息等需要 OAuth 应用和 JWT；
- JWT 有效期短，当前文档说明通常为 24 小时；
- 能用匿名 GET 时不要带认证，因为认证响应可能包含私有数据，而且服务端缓存策略不同。

### 5.3 常用调用

iNaturalist 中 Aves 的 `taxon_id` 为 `3`。

```bash
# 10 km 内、给定日期范围内、研究级鸟类记录
curl --get 'https://api.inaturalist.org/v1/observations' \
  --data-urlencode 'taxon_id=3' \
  --data-urlencode 'lat=31.2304' \
  --data-urlencode 'lng=121.4737' \
  --data-urlencode 'radius=10' \
  --data-urlencode 'd1=2026-10-01' \
  --data-urlencode 'd2=2026-10-23' \
  --data-urlencode 'quality_grade=research' \
  --data-urlencode 'per_page=100' \
  --data-urlencode 'order_by=observed_on' \
  --data-urlencode 'order=desc'

# 用包围盒查询
curl 'https://api.inaturalist.org/v1/observations?taxon_id=3&swlat=31.1&swlng=121.3&nelat=31.4&nelng=121.7&per_page=100'

# 搜索中文名或学名
curl 'https://api.inaturalist.org/v1/taxa/autocomplete?q=麻雀&locale=zh-CN'

# 分类单元详情
curl 'https://api.inaturalist.org/v1/taxa/13851'
```

主要筛选能力：

- 位置：`lat`、`lng`、`radius`，或 `swlat`、`swlng`、`nelat`、`nelng`；
- 时间：`d1`、`d2`、`observed_on`，也可按记录创建时间筛选；
- 物种：`taxon_id`，也可用 `taxon_name`，生产中优先 ID；父类群 ID 会包含子类群；
- 质量：`quality_grade=research,needs_id,casual`；不要把 research grade 翻译成“官方确认”；
- 许可：可按观测许可和照片许可筛选，具体参数查 API 文档。

### 5.4 限流和翻页

[推荐实践](https://www.inaturalist.org/pages/api+recommended+practices)给出的边界：

- 硬上限约 100 请求/分钟；
- 建议控制在不超过 60 请求/分钟，约 1 请求/秒；
- 建议每日约 10,000 请求以内；
- `per_page` 最大 200，应尽量提高每页数量而不是发很多小请求；
- 普通分页只能访问前 10,000 条左右；大数据使用 iNaturalist 导出或 GBIF；
- 媒体带宽限制更严格，文档列出约 5 GB/小时或 24 GB/日，严重滥用可能被永久阻断；
- 使用能识别项目的 `User-Agent`，429 后退避。

### 5.5 许可、缓存和署名

iNaturalist 平台不是所有内容统一一个许可。观察记录、照片和声音分别由用户选择许可，可能是 CC0、CC BY、CC BY-NC、其他 CC，也可能是 **All rights reserved**。见[许可说明](https://help.inaturalist.org/en/support/solutions/articles/151000175695)和[媒体许可说明](https://help.inaturalist.org/en/support/solutions/articles/151000169918)。

课程项目应：

1. 保存每条记录/媒体的 `license_code`、作者、原始 URL 和来源 URL；
2. 照片只展示许可满足用途的内容；All rights reserved 不复制、不本地缓存，除非获得作者许可；
3. CC BY-NC 内容可用于非商业课程，但必须署名且不得把成果转为商业服务；
4. CC BY-SA 内容如有改编，衍生成果需遵守相同方式共享；
5. 界面建议显示“观察者名 · iNaturalist · 许可”，并链接原记录；
6. 定期刷新，处理用户删除记录、改变许可或分类鉴定更新的情况。

API 没有禁止合理缓存，但缓存不能消除原内容的许可与隐私义务。建议附近结果缓存 30–60 分钟、分类详情缓存约 24 小时；不缓存登录后才能看到的私有坐标。

### 5.6 坐标隐私

[iNaturalist 地理隐私说明](https://help.inaturalist.org/en/support/solutions/articles/151000169938)包含：

- `open`：公开精确坐标；
- `obscured`：公共点随机落在约 0.2° × 0.2° 的单元内，赤道附近约 500 km²，真实坐标仅授权人员可见；
- `private`：公共数据不提供位置。

即使坐标已模糊，精确日期、地点备注、同一观察者相邻记录、照片背景和 EXIF 仍可能泄露位置。项目不得逆向推断，也不应请求受信任项目成员权限来绕过保护。

## 6. GBIF API

### 6.1 定位

GBIF 是跨机构、跨平台的数据聚合器，包含标本、观察记录、文献记录和来自 eBird/iNaturalist 的发布数据。优点是历史跨度大、查询统一；缺点是并非实时、字段完整度不一，而且同一次观察可能同时出现在源平台和 GBIF 中。

官方文档：[Occurrence API](https://techdocs.gbif.org/en/openapi/v1/occurrence)、[Species API](https://techdocs.gbif.org/en/openapi/v1/species)。

### 6.2 常用调用

```bash
# 学名匹配，返回 GBIF taxon key
curl --get 'https://api.gbif.org/v2/species/match' \
  --data-urlencode 'scientificName=Passer montanus'

# Aves 的 taxonKey 为 212：上海 10 km 内、指定时间段的人类观测
curl --get 'https://api.gbif.org/v1/occurrence/search' \
  --data-urlencode 'taxonKey=212' \
  --data-urlencode 'geoDistance=31.2304,121.4737,10km' \
  --data-urlencode 'eventDate=2026-10-01,2026-10-23' \
  --data-urlencode 'basisOfRecord=HUMAN_OBSERVATION' \
  --data-urlencode 'occurrenceStatus=PRESENT' \
  --data-urlencode 'hasCoordinate=true' \
  --data-urlencode 'limit=100'
```

位置还可用经纬度范围或 WKT `geometry`；物种最好使用 `taxonKey`/`speciesKey`；时间可用 `eventDate` 的单日或范围。

### 6.3 限流和大数据下载

- 搜索单页 `limit` 最大 300；
- `offset + limit` 的普通搜索窗口最多 100,000；
- GBIF 使用动态限流，没有一个可依赖的固定 QPS；收到 429 必须退避；
- 预计运行超过约 15 分钟或需要完整结果时，使用 [Occurrence Download](https://www.gbif.org/developer/occurrence#download)，不要并发扫页；
- Search API 不需要账号，异步 Download 需要免费 GBIF 账号和 HTTP Basic 认证；
- Download 会生成可引用 DOI，适合课程报告和可复现研究。

### 6.4 许可、缓存和署名

GBIF 只是索引，记录许可由发布数据集决定，通常为 CC0、CC BY 或 CC BY-NC。见 [GBIF 条款](https://www.gbif.org/terms)和[数据使用协议](https://www.gbif.org/terms/data-user)。

- 保存 `datasetKey`、发布机构、`occurrenceID`、`license` 和原记录链接；
- 混合许可结果应逐条判断，或在查询/导入时筛选到项目接受的许可；
- 缓存和派生数据也继续受原数据集许可约束；
- 报告中引用 GBIF 和数据发布者；批量下载优先引用 DOI，参见[引用指南](https://www.gbif.org/citation-guidelines)；
- 去重时利用源标识符和 `occurrenceID`，避免把 GBIF 中的 eBird/iNaturalist 副本算作独立记录。

### 6.5 敏感数据

GBIF 的敏感信息概化主要由发布者完成，并不保证所有数据都达到同一标准。应检查：

- `coordinateUncertaintyInMeters`；
- `dataGeneralizations`；
- `informationWithheld`；
- 坐标相关 `issues`；
- 数据集说明和本地保护名录。

参考 [GBIF 敏感物种数据概化指南](https://docs.gbif.org/sensitive-species-data-generalization/en/)和[最佳实践](https://docs.gbif.org/sensitive-species-best-practices/master/en/)。

## 7. 地图与地理编码

### 7.1 OpenStreetMap 地图数据与瓦片

OSM 数据采用 ODbL，必须在地图上清晰显示 `© OpenStreetMap contributors` 并链接到[版权与许可页](https://www.openstreetmap.org/copyright)。详细样式见 [OSMF Attribution Guidelines](https://osmfoundation.org/wiki/Attribution_Guidelines)。

公共瓦片地址：

```text
https://tile.openstreetmap.org/{z}/{x}/{y}.png
```

[公共瓦片政策](https://operations.osmfoundation.org/policies/tiles/)要求：

- 清晰署名；
- 提供能识别应用的 User-Agent/Referer；
- 遵守 HTTP 缓存头；无法读取时至少缓存 7 天；
- 禁止批量下载、离线包、预取整片区域和故意绕过缓存；
- 免费公共服务没有 SLA，可能封禁大流量应用。

小型课堂演示可以使用；若公开上线或需要离线地图，应选择明确允许该用途的商业瓦片商或自建瓦片服务，而不是扩大使用公共实例。

### 7.2 Nominatim 逆地理编码

```bash
curl -H 'User-Agent: bird-assistant-course/0.1 (contact@example.edu)' \
  'https://nominatim.openstreetmap.org/reverse?format=jsonv2&lat=31.2304&lon=121.4737&zoom=14&addressdetails=1'
```

[Nominatim 公共实例政策](https://operations.osmfoundation.org/policies/nominatim/)规定：

- **绝对上限 1 请求/秒**，是整个应用合计，不是每个用户；
- 必须有有效 User-Agent 或 Referer，并显示 OSM 署名；
- 应缓存查询与结果；
- 只适合用户触发、较低量使用；
- 禁止客户端自动补全、系统化扫区和大规模抓取。

因此附近记录列表不要对每一条坐标单独做逆地理编码。先按网格/地点 ID 合并，服务端排队并长缓存；敏感记录甚至不应反查精细地址。

### 7.3 高德开放平台

高德适合中国境内的底图、小程序地图、逆地理编码、POI、坐标转换和天气。Web 服务需要注册、认证并申请对应类型的 Key。

```bash
# 逆地理编码；高德参数是“经度,纬度”
curl 'https://restapi.amap.com/v3/geocode/regeo?key=YOUR_KEY&location=121.4737,31.2304&radius=1000&extensions=base'

# 把 GPS/WGS84 坐标转换为高德坐标，最多可一次传 40 对
curl 'https://restapi.amap.com/v3/assistant/coordinate/convert?key=YOUR_KEY&locations=121.4737,31.2304&coordsys=gps&output=json'
```

官方资料：[逆地理编码](https://lbs.amap.com/api/webservice/guide/api/georegeo)、[坐标转换](https://lbs.amap.com/api/webservice/guide/api/convert)、[配额与价格](https://lbs.amap.com/pages/base_service_price)。

重要约束：

- 配额和 QPS 按产品、认证类型和购买方案变化。调研时非商业个人认证的基础 LBS 有共享月免费额度，但不应把网页上的当前数值写死进程序；上线前以开发者控制台显示为准；
- [高德开放平台服务协议](https://lbs.amap.com/pages/terms/)明确限制抓取、预读取、存储、缓存、截图、索引、衍生数据库，以及脱离服务单独展示其结果；
- 课程学习可在认证和授权范围内使用服务及免费额度，但不等于可下载高德数据建立本地地名库；
- 不得把自己的 API 再封装成向第三方提供的“高德代理 API”；
- Key 应根据平台安全方案限制域名/IP，服务端 Key 不下发前端。

#### 坐标系必须分开

eBird、iNaturalist、GBIF 和 GPS 通常使用 **WGS84**；高德在中国境内地图展示使用高德坐标（GCJ-02）。推荐：

1. 数据库始终保存来源的 WGS84 坐标和精度；
2. 调 eBird/iNaturalist/GBIF 距离查询时用 WGS84；
3. 只在高德地图展示环节用官方接口转换为高德坐标；
4. 不用转换结果覆盖原始科学记录；
5. 明确记录 `coordinate_system`，否则点位会出现明显偏移。

## 8. 天气服务

### 8.1 Open-Meteo

Open-Meteo 的非商业免费 API 不需要 Key，适合课程项目。示例：

```bash
curl --get 'https://api.open-meteo.com/v1/forecast' \
  --data-urlencode 'latitude=31.2304' \
  --data-urlencode 'longitude=121.4737' \
  --data-urlencode 'hourly=temperature_2m,precipitation_probability,cloud_cover,visibility,wind_speed_10m' \
  --data-urlencode 'daily=sunrise,sunset,precipitation_sum' \
  --data-urlencode 'timezone=Asia/Shanghai' \
  --data-urlencode 'forecast_days=3'
```

观鸟场景建议字段：温度、降水概率、云量、能见度、10 米风速、日出日落。不要把天气和鸟类出现概率直接等同；若给出推荐，要说明是启发式规则。

[Open-Meteo 条款](https://open-meteo.com/en/terms)中非商业免费层当前限制为：

- 600 请求/分钟；
- 5,000 请求/小时；
- 10,000 请求/日；
- 300,000 请求/月。

限制按应用/IP 等综合执行，免费服务无 SLA。数据采用 CC BY 4.0，可合理缓存；界面显示 `Weather data by Open-Meteo.com` 并链接官网。建议当前/小时预报缓存 15–60 分钟，日预报缓存数小时。

### 8.2 和风天气 QWeather

[和风天气开发文档](https://dev.qweather.com/docs/)覆盖实时、逐小时、逐日天气和预警。新项目推荐使用 Ed25519 JWT，也支持 API Key；JWT 有效期最长通常为 24 小时。认证资料见[身份认证](https://dev.qweather.com/en/docs/configuration/authentication/)。

新 API 使用账号分配的专属 Host，例如：

```bash
curl -H 'Authorization: Bearer YOUR_JWT' \
  'https://YOUR_HOST/weather/v1/current/31.2304/121.4737'

# 或使用 X-QW-Api-Key，具体路径以当前文档为准
curl -H 'X-QW-Api-Key: YOUR_API_KEY' \
  'https://YOUR_HOST/weather/v1/current/31.2304/121.4737'
```

限制与合规：

- 当前采用按量计费，天气与基础服务每月前 50,000 次有免费额度；价格可能变化，以[计费页](https://dev.qweather.com/en/docs/finance/pricing/)和控制台为准；
- 页面需清晰显示 `Weather service by QWeather` 并链接官网；预警响应中的 `refer.sources` 等来源不得擅自修改，见[署名规则](https://dev.qweather.com/en/docs/terms/attribution/)；
- 官方[缓存最佳实践](https://dev.qweather.com/docs/best-practices/cache/)建议：实时天气约 10–30 分钟、逐小时 30–60 分钟、逐日 1–6 小时、预警 5–20 分钟；
- GeoAPI 地理数据不得缓存、批量下载或建立索引，天气缓存许可不能外推到所有接口；
- JWT 私钥/API Key 只能在后端保存。

## 9. 国内观鸟社区数据

### 9.1 中国观鸟记录中心 / BirdReport

官方网站：[birdreport.cn](https://www.birdreport.cn/)。截至调研日期，未找到面向第三方开发者的公开 API 文档、Key 申请、QPS、缓存政策或数据库再利用许可。

[用户协议](https://www.birdreport.cn/home/terms/page.html)提到用户上传内容在未另行说明时采用 CC BY-NC-SA 4.0，但这不等于：

- 平台数据库整体采用同一许可；
- 网页、接口、地图和整理成果允许批量抓取；
- 第三方有权批量再展示观测位置和用户信息；
- 可以逆向移动端私有接口。

[隐私政策](https://www.birdreport.cn/home/privacy/page.html)还涉及位置和用户信息。结论是：**没有书面授权就不接入**。可联系 `birdreport@163.com`，申请时至少确认：

1. 是否有官方 API 或可交付的数据集；
2. 允许访问的字段、地区、时间范围和用途；
3. QPS、日额度、分页与批量下载方式；
4. 能否缓存、缓存多久、能否生成派生统计；
5. 课程演示、公开答辩视频和 GitHub 截图是否允许；
6. 每条记录和平台的署名格式；
7. 敏感鸟种、巢址、夜栖地和个人位置如何概化；
8. 用户删除或撤回授权后的同步删除机制；
9. 许可终止后历史缓存如何处理。

在获得答复前，产品可以只提供“前往原平台查看”的普通链接，或使用团队自己采集并获得参与者授权的数据。

### 9.2 中国自然观察 / 中国自然观察数据

[山水自然保护中心的中国自然观察项目](https://www.shanshui.org/project/1056/)发布过报告和数据成果，但未找到适用于通用第三方应用的公开观测 API 与统一再利用条款。部分资料是特定合作项目、特定数据包授权，不能把一次下载许可外推到全部数据。

如需使用，应逐个数据包确认许可证并联系项目方；不抓取网页或地图。其公开说明也体现了合作授权和敏感坐标降精度的要求，可参考[相关数据说明](https://www.shanshui.org/information/1229/)。

### 9.3 国内数据源的判断原则

`网页能打开`、`搜索引擎能收录`、`App 能看到`、`第三方数据商能出售`，都不能代替原始权利人的 API/数据授权。任何国内社区数据接入都要形成可留档的书面授权或公开许可证。

## 10. 敏感鸟种和位置隐私设计

### 10.1 为什么上游模糊还不够

- 不同平台对敏感物种的定义、季节和地区规则不同；
- 濒危等级不等于敏感等级，普通物种的巢址也可能需要保护；
- 坐标模糊后，精确时间、地点名称、照片 EXIF、观察者连续轨迹仍可重新定位；
- 将模糊点用于“导航到此处”会把保护措施变成定位提示。

### 10.2 建议的统一字段

```text
geoprivacy: open | obscured | private | unknown
location_visibility: exact | source_obscured | app_obscured | withheld
coordinate_uncertainty_m: number | null
data_generalizations: text | null
information_withheld: text | null
sensitive_reason: source_rule | local_rule | nesting | roost | user_privacy | unknown
```

同时保留来源平台已经公开的坐标与不确定度，不要把“随机模糊点”标记成精确点。

### 10.3 服务端处理规则

1. **先判断再缓存**：原始响应进入受限处理层，脱敏后才写普通缓存、日志和分析库。
2. **从不获取无必要的真坐标**：不使用用户/项目的信任权限读取 iNaturalist 私有位置，也不尝试恢复 eBird 隐藏点。
3. **分级展示**：一般记录可显示合理精度；敏感记录显示 10–25 km 网格、县级区域或完全不显示地图点；极敏感记录只显示省/大区和月份。
4. **保护时间**：繁殖、巢址和夜栖地记录可把时间从分钟降为日期、月份，必要时延迟发布。
5. **保护文本和媒体**：删除精细地点名、路线描述、照片 EXIF/GPS；检查画面中的路牌和明显地标。
6. **关闭定位能力**：敏感记录不提供距离排序、路线规划、坐标复制、罕见鸟实时推送或按观察者轨迹浏览。
7. **最少权限**：客户端 API 永远只返回已经脱敏的字段；管理员查看也要留审计日志。
8. **定期更新名单**：结合来源平台规则、国家/地方保护名录、季节和专家意见，不只看 IUCN 等级。

概括原则：**只提供完成观鸟教育目标所需的最低位置精度。**

## 11. 缓存与署名矩阵

下面的 TTL 是课程项目的推荐起点；标为“官方建议”的才是平台给出的明确缓存指导。

| 数据 | 推荐缓存 | 依据/注意事项 | 推荐界面署名 |
|---|---:|---|---|
| AviList v2026 | 整版长期保存 | CC BY 4.0；按版本更新 | `Taxonomy: AviList v2026` + DOI |
| 中国生物物种名录 | 整版保存 | 使用官方年度包；再分发前确认 | `中国生物物种名录 2026` + 引用 |
| BirdNET+ Taxonomy | 按版本保存 | 字段和媒体保留各自来源/许可 | `BirdNET+ Taxonomy`；媒体另列作者和许可 |
| eBird 附近记录 | 10–30 分钟 | 工程建议；不做持久公开镜像 | `Data from eBird, Cornell Lab of Ornithology` |
| iNaturalist 附近记录 | 30–60 分钟 | 记录和媒体逐项许可，支持删除/变更 | `观察者 · iNaturalist · 许可` |
| GBIF 搜索结果 | 约 24 小时；历史数据可更久 | 遵守数据集逐项许可；大量数据用 DOI 下载 | `GBIF.org` + 发布者/数据集/DOI |
| OSM 公共瓦片 | 按 HTTP 头；否则至少 7 天 | 官方政策；禁止离线预取 | `© OpenStreetMap contributors` |
| Nominatim | 长缓存、复用 | 官方要求缓存、全应用 ≤1 req/s | `© OpenStreetMap contributors` |
| 高德结果 | 不自行缓存 | 协议原则上禁止；需例外则提交工单 | 按高德 SDK/服务要求展示标识 |
| Open-Meteo | 15–60 分钟；日预报数小时 | CC BY 4.0，合理缓存 | `Weather data by Open-Meteo.com` |
| QWeather | 当前 10–30 分钟等 | 官方按产品给 TTL；GeoAPI 不缓存 | `Weather service by QWeather` + 预警来源 |
| 国内社区 | 不缓存 | 等书面授权明确后再设计 | 按授权协议 |

所有缓存键至少包含：来源、接口版本、物种 ID、空间网格/经纬度、半径、时间范围、语言和筛选条件。缓存值应保存 `retrieved_at`、来源 URL、许可和过期时间。

## 12. 建议的内部数据模型

```json
{
  "source": "ebird",
  "source_record_id": "S123456789:eutspa",
  "source_url": "https://ebird.org/checklist/S123456789",
  "retrieved_at": "2026-10-23T10:00:00Z",
  "source_updated_at": null,
  "license": "eBird API Terms",
  "attribution": "eBird, Cornell Lab of Ornithology",

  "taxon_id": "internal-uuid",
  "scientific_name": "Passer montanus",
  "taxonomy_source": "AviList",
  "taxonomy_version": "v2026",
  "source_taxon_ids": {
    "ebird": "eutspa",
    "inat": 13851,
    "gbif": 5231198
  },

  "observed_at": "2026-10-20T08:15:00+08:00",
  "lat_public": 31.23,
  "lng_public": 121.47,
  "coordinate_system": "WGS84",
  "coordinate_uncertainty_m": 10000,
  "geoprivacy": "obscured",
  "location_visibility": "source_obscured",
  "sensitive_reason": "source_rule",

  "quality_grade": "reviewed",
  "media": [],
  "raw_payload_expires_at": "2026-10-23T10:30:00Z"
}
```

注意：示例中的跨库 ID 已按调研日期核对，但仍可能随平台分类版本变化；导入时必须从当前版本映射表再次核验，不能把示例值直接写入正式数据库。

## 13. 后端接口建议

项目可以对前端提供自己的最小接口：

```text
GET /api/v1/species/search?q=树麻雀&locale=zh-CN
GET /api/v1/species/{internalTaxonId}
GET /api/v1/observations/nearby?lat=...&lng=...&radiusKm=10&days=14&taxonId=...
GET /api/v1/weather?lat=...&lng=...
```

后端工作顺序：

```text
参数校验 → 坐标系确认 → 网格化缓存键 → 上游限流
→ 来源 ID 映射 → 许可筛选 → 跨源去重
→ 敏感位置/时间/文本/媒体脱敏 → 返回前端
```

关键要求：

- 附近请求限制最大半径和最大天数，避免被当作批量导出接口；
- Token 和私钥只读环境变量，不写入仓库；
- 给不同上游设置独立并发池、超时、重试和熔断；
- 429 读取 `Retry-After`，没有时指数退避并加入随机抖动；
- 不自动重试 400/401/403；
- 记录上游、请求时间、缓存命中、许可版本和脱敏结果，但日志不记录敏感原坐标；
- 明确时区。上游本地观察时间可能没有 UTC 偏移，不能盲目附加 `Z`；
- 观测存在只能说明“有人在这里记录到”，不能据此推断未记录物种不存在。

## 14. MVP 推荐方案

### 阶段一：可以直接开始

1. 下载 AviList v2026，建立本地物种表；
2. 导入 eBird Taxonomy，建立 `speciesCode` 映射；
3. 申请 eBird Token，通过后端实现 `/observations/nearby`；
4. 匿名调用 iNaturalist 公共 GET，补充记录；
5. 只展示许可合适的图片，先不接 All rights reserved 媒体；
6. 用 Open-Meteo 获取天气；国内网络不稳定时切换 QWeather；
7. 地图按部署环境选择高德或合规的 OSM 服务；
8. 在后端先实现敏感位置脱敏，再做前端地图。

### 阶段二：需要更多历史数据时

1. 接 GBIF Search 做低频历史查询；
2. 大样本分析用 GBIF Download DOI；
3. 增加 BirdNET+ Taxonomy 的简介和跨库 ID；
4. 用中国生物物种名录补充中文名、异名和国内分布；
5. 实现跨源去重、来源许可筛选和数据删除同步。

### 暂缓

- 国内社区观测数据：等书面授权；
- IUCN API：等明确需要且申请获批；
- eBird/iNaturalist 私有位置：不接入；
- 公共 OSM 瓦片离线包、Nominatim 自动补全：政策不允许；
- 把高德返回结果做成本地地址/POI 数据库：条款不允许。

## 15. 上线前检查清单

- [ ] 每个上游都记录了官方文档、条款 URL、核对日期和负责人；
- [ ] eBird、高德、QWeather 等密钥只在后端；
- [ ] 请求能按位置、日期、物种 ID 正确过滤，坐标系没有混用；
- [ ] 限流、缓存、超时、429 退避和故障降级已经测试；
- [ ] 页面级和记录级署名都完整、可点击；
- [ ] iNaturalist/GBIF 媒体与记录逐项保存许可；
- [ ] 敏感位置在进入普通缓存、日志和前端之前已经脱敏；
- [ ] 精确时间、地点文本、EXIF、轨迹和导航没有造成二次泄露；
- [ ] 跨源重复记录不会被当作多个独立证据；
- [ ] 课程答辩录屏、公开部署和 GitHub 文档没有暴露密钥或受限数据；
- [ ] 国内社区数据已经取得可留档的明确授权，否则保持未接入；
- [ ] 上线前再次核对价格、配额、版本和使用条款。

## 16. 官方链接汇总

- eBird：[API v2](https://documenter.getpostman.com/view/664302/S1ENwy59) · [申请 Token](https://ebird.org/api/keygen) · [API 条款](https://www.birds.cornell.edu/home/ebird-api-terms-of-use/) · [数据访问条款](https://www.birds.cornell.edu/home/ebird-data-access-terms-of-use/) · [敏感物种](https://support.ebird.org/en/support/solutions/articles/48000803210)
- iNaturalist：[API v1](https://api.inaturalist.org/v1/docs/) · [开发者说明](https://www.inaturalist.org/pages/developers) · [推荐实践](https://www.inaturalist.org/pages/api+recommended+practices) · [地理隐私](https://help.inaturalist.org/en/support/solutions/articles/151000169938)
- GBIF：[Occurrence API](https://techdocs.gbif.org/en/openapi/v1/occurrence) · [Species API](https://techdocs.gbif.org/en/openapi/v1/species) · [数据使用协议](https://www.gbif.org/terms/data-user) · [引用指南](https://www.gbif.org/citation-guidelines)
- 名录：[AviList v2026](https://www.avilist.org/checklist/v2026/) · [BirdNET+ Taxonomy](https://birdnet.cornell.edu/taxonomy/docs) · [中国生物物种名录](http://www.sp2000.org.cn/)
- 地图：[OSM 许可](https://www.openstreetmap.org/copyright) · [OSM 瓦片政策](https://operations.osmfoundation.org/policies/tiles/) · [Nominatim 政策](https://operations.osmfoundation.org/policies/nominatim/) · [高德 Web API](https://lbs.amap.com/api/webservice/summary/) · [高德条款](https://lbs.amap.com/pages/terms/)
- 天气：[Open-Meteo 文档](https://open-meteo.com/en/docs) · [Open-Meteo 条款](https://open-meteo.com/en/terms) · [QWeather 文档](https://dev.qweather.com/docs/) · [QWeather 署名](https://dev.qweather.com/en/docs/terms/attribution/)
- 国内社区：[中国观鸟记录中心](https://www.birdreport.cn/) · [用户协议](https://www.birdreport.cn/home/terms/page.html) · [隐私政策](https://www.birdreport.cn/home/privacy/page.html) · [中国自然观察](https://www.shanshui.org/project/1056/)

