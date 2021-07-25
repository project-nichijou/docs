# 内部数据库 (`Nichijou DB`) 规范

## 内部数据库规范

内部数据库的主要功能是进行条目的**社区修订与维护**，设计原则是**最小冗余**，即大部分数据仍旧保存在数据源的数据库当中，内部数据库仅保留**匹配**与**修订**相关的内容。下面将简述各张`tables`及详细内容。

### 内部`ID` (`Nichijou ID`)

`Nichijou ID`在下面将简记为`nid`。放长远眼光看，为了稳定、可持续的发展，建立一套内部自己的`ID`是十分有必要的。

现阶段我们私有`ID`的建立方法如下:
1. `ID`格式为`INT`
2. 以`AniDB`为核心，由于其也有跳跃的`ID`，我们按照从小到大的顺序排列入库，并重新编号
3. 在此之后，新加入的条目一般需要人工审核（即`Matcher`运行后无法和现有条目精确匹配）

### `Nichijou: Tables`

|      名称      | Required | Description  |
| :------------: | :------: | :----------: |
|    `anime`     |    ✅     | 番剧匹配索引 |
|  `anime_name`  |    ✅     |   番剧名称   |
| `episode_name` |    ✅     |   剧集名称   |
|  `match_fail`  |    ✅     |   失败匹配   |
|   `conflict`   |    ✅     |   冲突内容   |
|    `revise`    |    ✅     |   修订数据   |
|    `sites`     |    ✅     |   视频网站   |
|   `sources`    |    ✅     |    数据源    |
|     `log`      |    ✅     |     日志     |

### `Nichijou: [Table] anime`

**`Primary Key`: `nid`**

|    名称    | 数据类型  | 长度/集合 | 无符号 | Nullable |       描述       |
| :--------: | :-------: | :-------: | :----: | :------: | :--------------: |
|   `nid`    |   `INT`   |     /     |   ✅    |    ❌     |   内部番剧`ID`   |
| `<source>` | `VARCHAR` |    40     |   ✅    |    ❌     | 数据源的番剧`ID` |

说明：`<source>`为模板，即数据源的名称。多个数据源会增加列数。

### `Nichijou: [Table] anime_name`

**`Primary Key`: `nid` & `name`**

|  名称  | 数据类型  | 长度/集合 | 无符号 | Nullable |     描述     |
| :----: | :-------: | :-------: | :----: | :------: | :----------: |
| `nid`  |   `INT`   |     /     |   ✅    |    ❌     | 内部番剧`id` |
| `name` | `VARCHAR` |    200    |   /    |    ❌     |   番剧名称   |

### `Nichijou: [Table] episode_name`

**`Primary Key`: `nid` & `type` & `sort` & `name`**

|  名称  | 数据类型  | 长度/集合 | 无符号 | Nullable |           描述            |
| :----: | :-------: | :-------: | :----: | :------: | :-----------------------: |
| `nid`  |   `INT`   |     /     |   ✅    |    ❌     |       内部番剧`id`        |
| `type` |   `INT`   |     /     |   ✅    |    ❌     | `[Enum] AnimeEpisodeType` |
| `sort` |   `INT`   |     /     |   ✅    |    ❌     |    当前`type`中多少话     |
| `name` | `VARCHAR` |    200    |   /    |    ❌     |         剧集名称          |

### `Nichijou: [Table] match_fail`

**`Primary Key`: `id` & `source`**

|   名称   |  数据类型  | 长度/集合 | 无符号 | Nullable |                描述                |
| :------: | :--------: | :-------: | :----: | :------: | :--------------------------------: |
|   `id`   |   `INT`    |     /     |   ✅    |    ❌     |        数据源番剧的唯一`id`        |
| `source` | `VARCHAR`  |    40     |   /    |    ❌     |             数据源名称             |
|  `dis`   | `LONGTEXT` |     /     |   /    |    ❌     | `json: {<name>: [<NameDistance>]}` |

注：
- 关于`dis`字段：主要给出的是最为相似的几个可能结果。一个`id`可能有多个`names`，我们这里给出每个`name`对应的**前五小**编辑距离。此外，如果某个名字在数据库中已经被列为了**重复项**，那么它的编辑距离就是`-1`，这里可以参考[`Matcher` 实现细节](#matcher-实现细节)。下面我们将给出一个示例：

```
{
	"Kanon": [
		{
			"name": "Kanon",
			"dis": -1
		}
	],
	"Kanon（京都版）": [
		{
			"name": "Kanon",
			"dis": 5
		},
		{
			"name": "Kanon 2002",
			"dis": 5
		},
		{
			"name": "Kanon",
			"dis": 5
		},
		{
			"name": "Kanon 06",
			"dis": 5
		},
		{
			"name": "Kanon `06",
			"dis": 5
		}
	],
	"Kanon（第二部 BS-i版）": [
		{
			"name": "Kanon (2002)",
			"dis": 10
		},
		{
			"name": "Kanon 2002",
			"dis": 10
		},
		{
			"name": "Kanon (2006)",
			"dis": 10
		},
		{
			"name": "Kanon 06",
			"dis": 10
		},
		{
			"name": "Kanon Remake",
			"dis": 10
		}
	],
	"华音": [
		{
			"name": "玲音",
			"dis": 1
		},
		{
			"name": "rx",
			"dis": 2
		},
		{
			"name": "OT",
			"dis": 2
		},
		{
			"name": "TU",
			"dis": 2
		},
		{
			"name": "CB",
			"dis": 2
		}
	],
	"雪之少女": [
		{
			"name": "雪之少女",
			"dis": -1
		}
	],
	"雪季之恋": [
		{
			"name": "雪之少女",
			"dis": 3
		},
		{
			"name": "推理之绊",
			"dis": 3
		},
		{
			"name": "風之谷",
			"dis": 3
		},
		{
			"name": "风之谷",
			"dis": 3
		},
		{
			"name": "天空之城",
			"dis": 3
		}
	],
	"雪色奇迹": [
		{
			"name": "夏色奇迹",
			"dis": 1
		},
		{
			"name": "橘色奇迹",
			"dis": 1
		},
		{
			"name": "星空奇迹",
			"dis": 2
		},
		{
			"name": "雪之少女",
			"dis": 3
		},
		{
			"name": "水色時代",
			"dis": 3
		}
	]
}
```

### `Nichijou: [Table] conflict`

**`Primary Key`: `nid` & `field`**

|  名称   | 数据类型  | 长度/集合 | 无符号 | Nullable |      描述      |
| :-----: | :-------: | :-------: | :----: | :------: | :------------: |
|  `nid`  |   `INT`   |     /     |   ✅    |    ❌     |  内部番剧`id`  |
| `field` | `VARCHAR` |    40     |   /    |    ❌     | 冲突的字段名称 |

### `Nichijou: [Table] revise`

**`Primary Key`: `nid` & `field` & `target`**

|   名称   |  数据类型  | 长度/集合 | 无符号 | Nullable |      描述      |
| :------: | :--------: | :-------: | :----: | :------: | :------------: |
|  `nid`   |   `INT`    |     /     |   ✅    |    ❌     |  内部番剧`id`  |
| `field`  | `VARCHAR`  |    40     |   /    |    ❌     | 修订的字段名称 |
| `target` | `VARCHAR`  |    40     |   /    |    ✅     | 修订目标数据源 |
| `value`  | `LONGTEXT` |     /     |   /    |    ✅     |    修订内容    |

### `Nichijou: [Table] sites`

**`Primary Key`: `name`**

|     名称      |  数据类型  | 长度/集合 | 无符号 | Nullable |                 描述                 |
| :-----------: | :--------: | :-------: | :----: | :------: | :----------------------------------: |
|    `name`     | `VARCHAR`  |    40     |   /    |    ❌     |               网站名称               |
| `urlTemplate` | `LONGTEXT` |     /     |   /    |    ❌     |               链接模板               |
|   `regions`   | `LONGTEXT` |     /     |   /    |    ❌     | 支持地区`json, array`, `AnimeRegion` |

注意：
- `urlTemplate`举例: `https://www.bilibili.com/bangumi/media/md{{id}}`
- 此部分设计借鉴 [bangumi-data](https://github.com/bangumi-data/bangumi-data)

### `Nichijou: [Table] sources`

**`Primary Key`: `name`**

|  名称  |  数据类型  | 长度/集合 | 无符号 | Nullable |   描述   |
| :----: | :--------: | :-------: | :----: | :------: | :------: |
| `name` | `VARCHAR`  |    40     |   /    |    ❌     | 网站名称 |
| `url`  | `LONGTEXT` |     /     |   /    |    ❌     | 网站链接 |

### `Nichijou: [Table] log`

|   名称    |  数据类型  | 长度/集合 | 无符号 | Nullable | 描述  |
| :-------: | :--------: | :-------: | :----: | :------: | :---: |
|  `time`   | `DATETIME` |     /     |   /    |    ❌     | 时间  |
| `content` | `LONGTEXT` |     /     |   /    |    ❌     | 内容  |
