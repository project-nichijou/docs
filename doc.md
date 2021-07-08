此文档将详细描述各组件的技术实现以及规范。

- [Server](#server)
	- [番剧数据库的构建](#番剧数据库的构建)
		- [总架构](#总架构)
		- [`Spider` 推荐实现](#spider-推荐实现)
		- [`Matcher` 实现细节](#matcher-实现细节)
		- [`Publisher` 实现细节](#publisher-实现细节)
		- [自定义数据结构](#自定义数据结构)
			- [`[Object] NameDistance`](#object-namedistance)
			- [`[Object] RelatedItem`](#object-relateditem)
			- [`[Enum] AnimeEpisodeType`](#enum-animeepisodetype)
			- [`[Enum] AiringStatus`](#enum-airingstatus)
		- [爬取数据库规范](#爬取数据库规范)
			- [关于`id`](#关于id)
			- [`Spider: Tables`](#spider-tables)
			- [`Spider: [Table] anime`](#spider-table-anime)
			- [`Spider: [Table] episode`](#spider-table-episode)
			- [`Spider: [Table] anime-name`](#spider-table-anime-name)
			- [`Spider: [Table] episode-name`](#spider-table-episode-name)
			- [`Spider: [Table] log`](#spider-table-log)
			- [`Spider: [Table] request-failed`](#spider-table-request-failed)
			- [`Spider: [Table] cache`](#spider-table-cache)
		- [内部数据库规范](#内部数据库规范)
			- [内部`ID` (`Nichijou ID`)](#内部id-nichijou-id)
			- [`Nichijou: Tables`](#nichijou-tables)
			- [`Nichijou: [Table] anime`](#nichijou-table-anime)
			- [`Nichijou: [Table] anime-name`](#nichijou-table-anime-name)
			- [`Nichijou: [Table] episode-name`](#nichijou-table-episode-name)
			- [`Nichijou: [Table] match-fail`](#nichijou-table-match-fail)
			- [`Nichijou: [Table] conflict`](#nichijou-table-conflict)
			- [`Nichijou: [Table] revise`](#nichijou-table-revise)
		- [发布数据规范](#发布数据规范)
		- [发布数据方式](#发布数据方式)

# Server

## 番剧数据库的构建

### 总架构

番剧数据库的大部分数据源于爬取。爬取的数据将根据**爬取数据库规范**进行数据转换存入相应的数据库。在爬取之后，爬取得到的数据将会与**内部数据库**中已有的数据进行匹配，根据**内部数据库规范**进行合理储存。在完成上述工作之后，将会根据**发布数据规范**将数据以多种形式分发。

除此之外，**内部数据库**还包含了社区修订、维护的数据。

总结一下，整个数据库构建过程中的数据流动可以拆分为两个部分：
- **爬虫自动爬取发布部分**
  - **爬取数据** `Spider`
  - **转换数据** `Converter` (一般包含在`Spider`当中)
  - **匹配数据** `Matcher`
  - **发布数据** `Publisher`
- **人工处理部分**
  - 解决 `Matcher` 未能精确匹配的问题
  - 解决 `Publisher` 发布时多个数据源数据冲突 (默认情况下会有合并优先级)
  - 人工修订数据

![nichijou-database](imgs/nichijou-database.svg)

### `Spider` 推荐实现

使用`Python`的`Scrapy`框架进行开发。我们将提供部分`Converter`以及数据库相关的，可以共用的代码。

### `Matcher` 实现细节

新增数据源中的`name`字段将会与**内部数据库**进行模糊匹配，计算出编辑距离 (Levenshtein距离)。如果某一条目的任意一个`name`可以精确匹配 (即编辑距离为0)，则确认匹配，反之，将记录下与每条`name`前五小的编辑距离。

### `Publisher` 实现细节

根据**内部数据库**的匹配数据，综合来自多个数据源的信息，并进行确认、核实。如果遇到多个信息源数据不同的情况，首先会根据默认优先级选择数据，但同时会记录一条`conflict`信息，等待人工审核批准。所有批准后的信息将作为`revise`写入数据库。注意：在所有信息合并的过程中，`revised`的数据有着最高优先级，并且不会产生`conflict`.

### 自定义数据结构

#### `[Object] NameDistance`

| Field  |   Type   | Nullable | Description |
| :----: | :------: | :------: | :---------: |
| `name` | `String` |    ❌     |    名称     |
| `dis`  |  `int`   |    ❌     |  编辑距离   |

#### `[Object] RelatedItem`

| Field  |   Type   | Nullable | Description |
| :----: | :------: | :------: | :---------: |
| `name` | `String` |    ❌     |    名称     |
| `url`  | `String` |    ❌     |    链接     |

#### `[Enum] AnimeEpisodeType`

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

#### `[Enum] AiringStatus`

**放送状态**

| Name  | Value |
| :---: | :---: |
|  Air  |   0   |
|  NA   |   1   |
| Today |   2   |
| 其他  |   3   |

### 爬取数据库规范

由于我们需要爬取多个站点来丰富数据，同时需要确保极强的可扩展性，我们势必需要确定一套持久化、可扩展的存储结构。下面将简述爬取得到后的数据的转换要求 (关系到`Converter`的实现)，以及数据库各张`table`的具体建立方式。

#### 关于`id`

如果数据源的`id`不是数型的，那么应该进行`MD5`的哈希，取低32位 (共128位) 进行储存。由于数据量不是特别大，所以碰撞的概率较低。

#### `Spider: Tables`

关于数据库的`tables`:

|       名称       | Required |              Description              |
| :--------------: | :------: | :-----------------------------------: |
|     `anime`      |    ✅     |             动画详细信息              |
|    `episode`     |    ✅     |           动画分集详细信息            |
|   `anime-name`   |    ✅     |             动画名称汇总              |
|  `episode-name`  |    ✅     |             剧集名称汇总              |
|      `log`       |    ✅     |                 日志                  |
| `request-failed` |    ✅     |        失败请求 (可以进行重试)        |
|     `cache`      |    ✅     |               蜘蛛缓存                |
|       `id`       |    ❌     | 精简的`id`表, 可以附上`URL`和基本信息 |

上述Required的数据表需要存在，用于统一管理检索。如有必要，可以额外建立数据表 (如`id`表)。

#### `Spider: [Table] anime`

**`Primary Key`: `id`**

|   名称    |  数据类型  | 长度/集合 | 无符号 | Nullable |          描述           |
| :-------: | :--------: | :-------: | :----: | :------: | :---------------------: |
|   `id`    |   `INT`    |     /     |   ✅    |    ❌     |  数据源番剧的唯一`id`   |
|   `url`   | `LONGTEXT` |     /     |   /    |    ❌     |          链接           |
|  `name`   | `VARCHAR`  |    200    |   /    |    ❌     |          原名           |
| `name_cn` | `VARCHAR`  |    200    |   /    |    ✅     |         中文名          |
|  `desc`   | `LONGTEXT` |     /     |   /    |    ✅     |          简介           |
| `eps_cnt` |   `INT`    |     /     |   ✅    |    ✅     |          话数           |
|  `date`   |   `DATE`   |     /     |   /    |    ✅     |        放送日期         |
| `weekday` |   `INT`    |     /     |   ✅    |    ✅     |        放送星期         |
|  `meta`   | `LONGTEXT` |     /     |   /    |    ✅     |         元数据          |
|  `tags`   | `LONGTEXT` |     /     |   /    |    ✅     |          标签           |
|  `type`   | `VARCHAR`  |    10     |   /    |    ✅     |          种类           |
|  `image`  | `LONGTEXT` |     /     |   /    |    ✅     |        图像地址         |
| `rating`  | `DECIMAL`  |   32,28   |   ✅    |    ✅     |        站内评分         |
|  `rank`   |   `INT`    |     /     |   ✅    |    ✅     |        站内排名         |
| `related` | `LONGTEXT` |     /     |   /    |    ✅     | `json`: `[RelatedItem]` |

注: 
- `date`之所以是`DATE`而不是`VARCHAR`，是因为诸如`bangumi`的网站里面虽然`date`有额外信息，比方说特典日期，但是已经包含在`meta`字段了，规范格式有助于后期处理。
- `meta`中不同分条一般用换行隔开
- `tags`中不同条目用换行隔开
- `type`可能值: `TV`, `OVA`, ...

#### `Spider: [Table] episode`

**`Primary Key`: `id` & `type` & `sort`**

|    名称    |  数据类型  | 长度/集合 | 无符号 | Nullable |           描述            |
| :--------: | :--------: | :-------: | :----: | :------: | :-----------------------: |
|    `id`    |   `INT`    |     /     |   ✅    |    ❌     |   数据源番剧的唯一`id`    |
|   `type`   |   `INT`    |     /     |   ✅    |    ❌     | `[Enum] AnimeEpisodeType` |
|   `sort`   |   `INT`    |     /     |   ✅    |    ❌     |    当前`type`中多少话     |
|   `url`    | `LONGTEXT` |     /     |   /    |    ❌     |           链接            |
|   `name`   | `VARCHAR`  |    200    |   /    |    ✅     |           原名            |
| `name_cn`  | `VARCHAR`  |    200    |   /    |    ✅     |          中文名           |
|  `status`  |   `INT`    |     /     |   ✅    |    ❌     |   `[Enum] AiringStatus`   |
| `duration` |   `INT`    |     /     |   ✅    |    ✅     |         时长 (秒)         |
|   `date`   |   `DATE`   |     /     |   /    |    ✅     |         放送日期          |
|   `desc`   | `LONGTEXT` |     /     |   /    |    ✅     |           简介            |

#### `Spider: [Table] anime-name`

**`Primary Key`: `id` & `name`**

|  名称  | 数据类型  | 长度/集合 | 无符号 | Nullable |         描述         |
| :----: | :-------: | :-------: | :----: | :------: | :------------------: |
|  `id`  |   `INT`   |     /     |   ✅    |    ❌     | 数据源番剧的唯一`id` |
| `name` | `VARCHAR` |    200    |   /    |    ❌     |       番剧名称       |

#### `Spider: [Table] episode-name`

**`Primary Key`: `id` & `type` & `sort` & `name`**

|  名称  | 数据类型  | 长度/集合 | 无符号 | Nullable |           描述            |
| :----: | :-------: | :-------: | :----: | :------: | :-----------------------: |
|  `id`  |   `INT`   |     /     |   ✅    |    ❌     |   数据源番剧的唯一`id`    |
| `type` |   `INT`   |     /     |   ✅    |    ❌     | `[Enum] AnimeEpisodeType` |
| `sort` |   `INT`   |     /     |   ✅    |    ❌     |    当前`type`中多少话     |
| `name` | `VARCHAR` |    200    |   /    |    ✅     |         剧集名称          |


#### `Spider: [Table] log`

|   名称    |  数据类型  | 长度/集合 | 无符号 | Nullable | 描述  |
| :-------: | :--------: | :-------: | :----: | :------: | :---: |
|  `time`   | `DATETIME` |     /     |   /    |    ❌     | 时间  |
| `content` | `LONGTEXT` |     /     |   /    |    ❌     | 内容  |

#### `Spider: [Table] request-failed`

|   名称   |  数据类型  | 长度/集合 | 无符号 | Nullable |      描述       |
| :------: | :--------: | :-------: | :----: | :------: | :-------------: |
|  `url`   | `LONGTEXT` |     /     |   ✅    |    ❌     | 失败任务的`url` |
| `spider` | `VARCHAR`  |    20     |   /    |    ❌     |   蜘蛛的名称    |
|  `desc`  | `LONGTEXT` |     /     |   /    |    ✅     |    错误内容     |

注:
- `spider`字段，以`Scrapy`为例，应该为`<spider>.name`
- `desc`字段，如果可以，尽可能附带堆栈信息

#### `Spider: [Table] cache`

|   名称    |  数据类型  | 长度/集合 | 无符号 | Nullable |      描述      |
| :-------: | :--------: | :-------: | :----: | :------: | :------------: |
|   `url`   | `LONGTEXT` |     /     |   /    |    ❌     | 缓存内容的链接 |
| `expire`  | `DATETIME` |     /     |   /    |    ❌     |  缓存过期时间  |
| `content` | `LONGTEXT` |     /     |   /    |    ❌     |    缓存内容    |

### 内部数据库规范

内部数据库的主要功能是进行条目的**社区修订与维护**，设计原则是**最小冗余**，即大部分数据仍旧保存在数据源的数据库当中，内部数据库仅保留**匹配**与**修订**相关的内容。下面将简述各张`tables`及详细内容。

#### 内部`ID` (`Nichijou ID`)

`Nichijou ID`在下面将简记为`nid`。放长远眼光看，为了稳定、可持续的发展，建立一套内部自己的`ID`是十分有必要的。

现阶段我们私有`ID`的建立方法如下:
1. `ID`格式为`INT`
2. 以`AniDB`为核心，由于其也有跳跃的`ID`，我们按照从小到大的顺序排列入库，并重新编号
3. 在此之后，新加入的条目一般需要人工审核（即`Matcher`运行后无法和现有条目精确匹配）

#### `Nichijou: Tables`

|      名称      | Required | Description  |
| :------------: | :------: | :----------: |
|    `anime`     |    ✅     | 番剧匹配索引 |
|  `anime-name`  |    ✅     |   番剧名称   |
| `episode-name` |    ✅     |   剧集名称   |
|  `match-fail`  |    ✅     |   失败匹配   |
|   `conflict`   |    ✅     |   冲突内容   |
|    `revise`    |    ✅     |   修订数据   |

#### `Nichijou: [Table] anime`

**`Primary Key`: `nid`**

|    名称    | 数据类型  | 长度/集合 | 无符号 | Nullable |       描述       |
| :--------: | :-------: | :-------: | :----: | :------: | :--------------: |
|   `nid`    |   `INT`   |     /     |   ✅    |    ❌     |   内部番剧`ID`   |
| `<source>` | `VARCHAR` |    40     |   ✅    |    ❌     | 数据源的番剧`ID` |

说明：`<source>`为模板，即数据源的名称。多个数据源会增加列数。

#### `Nichijou: [Table] anime-name`

**`Primary Key`: `nid` & `name`**

|  名称  | 数据类型  | 长度/集合 | 无符号 | Nullable |     描述     |
| :----: | :-------: | :-------: | :----: | :------: | :----------: |
| `nid`  |   `INT`   |     /     |   ✅    |    ❌     | 内部番剧`id` |
| `name` | `VARCHAR` |    200    |   /    |    ❌     |   番剧名称   |

#### `Nichijou: [Table] episode-name`

**`Primary Key`: `nid` & `type` & `sort` & `name`**

|  名称  | 数据类型  | 长度/集合 | 无符号 | Nullable |           描述            |
| :----: | :-------: | :-------: | :----: | :------: | :-----------------------: |
| `nid`  |   `INT`   |     /     |   ✅    |    ❌     |       内部番剧`id`        |
| `type` |   `INT`   |     /     |   ✅    |    ❌     | `[Enum] AnimeEpisodeType` |
| `sort` |   `INT`   |     /     |   ✅    |    ❌     |    当前`type`中多少话     |
| `name` | `VARCHAR` |    200    |   /    |    ✅     |         剧集名称          |

#### `Nichijou: [Table] match-fail`

**`Primary Key`: `id` & `source`**

|   名称   |  数据类型  | 长度/集合 | 无符号 | Nullable |             描述             |
| :------: | :--------: | :-------: | :----: | :------: | :--------------------------: |
|   `id`   |   `INT`    |     /     |   ✅    |    ❌     |     数据源番剧的唯一`id`     |
| `source` | `VARCHAR`  |    40     |   /    |    ❌     |          数据源名称          |
|  `dis`   | `LONGTEXT` |     /     |   /    |    ❌     | 前五小`json: [NameDistance]` |

#### `Nichijou: [Table] conflict`

**`Primary Key`: `nid` & `field`**

|  名称   | 数据类型  | 长度/集合 | 无符号 | Nullable |      描述      |
| :-----: | :-------: | :-------: | :----: | :------: | :------------: |
|  `nid`  |   `INT`   |     /     |   ✅    |    ❌     |  内部番剧`id`  |
| `field` | `VARCHAR` |    40     |   /    |    ❌     | 冲突的字段名称 |

#### `Nichijou: [Table] revise`

|  名称   |  数据类型  | 长度/集合 | 无符号 | Nullable |      描述      |
| :-----: | :--------: | :-------: | :----: | :------: | :------------: |
|  `nid`  |   `INT`    |     /     |   ✅    |    ❌     |  内部番剧`id`  |
| `field` | `VARCHAR`  |    40     |   /    |    ❌     | 修订的字段名称 |
| `value` | `LONGTEXT` |     /     |   /    |    ❌     |    修订内容    |

### 发布数据规范


### 发布数据方式

