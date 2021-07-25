# 番剧数据库中的数据结构与格式

## 关于日期与时间

本项目中对于日期**不采用**ISO 8601标准。我们的时间格式统一为：
- 日期：`YYYY-MM-DD `
- 日期与时间：`YYYY-MM-DD HH:MM:SS`

由于数据量很大，且都为自动处理的，所以大部分时间都只能精确到天，且无法确定时区，所以作如上规定。

## 自定义数据结构

### `[Object] NameDistance`

**名称与距离**

| Field  |   Type   | Nullable | Description |
| :----: | :------: | :------: | :---------: |
| `name` | `String` |    ❌     |    名称     |
| `dis`  |  `int`   |    ❌     |  编辑距离   |

### `[Object] RelatedItem`

**相关条目**

| Field  |   Type   | Nullable | Description |
| :----: | :------: | :------: | :---------: |
| `name` | `String` |    ❌     |    名称     |
| `url`  | `String` |    ❌     |    链接     |

### `[Object] AnimeSite`

**动画网站链接**

| Field  |   Type   | Nullable |  Description   |
| :----: | :------: | :------: | :------------: |
| `name` | `String` |    ❌     |    网站名称    |
|  `id`  | `String` |    ❌     | 内容在网站的ID |

### `[Enum] AnimeEpisodeType`

**动画剧集种类**

| Name  | Value |
| :---: | :---: |
| 本篇  |   0   |
|  SP   |   1   |
|  OP   |   2   |
|  ED   |   3   |
| PV/CM |   4   |
|  MAD  |   5   |
| 其他  |   6   |

### `[Enum] AnimeRegion`

**动画播放地区**

| Name  | Value |
| :---: | :---: |
| 大陆  | `CN`  |
| 香港  | `HK`  |
| 澳门  | `MO`  |
| 台湾  | `TW`  |
| 日本  | `JP`  |

### `[Enum] AiringStatus`

**放送状态**

| Name  | Value |
| :---: | :---: |
|  Air  |   0   |
|  NA   |   1   |
| Today |   2   |
| 其他  |   3   |
