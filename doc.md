此文档将详细描述各组件的技术实现以及规范。

- [Server](#server)
	- [番剧数据库的构建](#番剧数据库的构建)
		- [总架构](#总架构)
		- [`Spider` 推荐实现](#spider-推荐实现)
		- [`Matcher` 实现细节](#matcher-实现细节)
		- [自定义数据结构](#自定义数据结构)
			- [`[Object] RelatedItem`](#object-relateditem)
			- [`[Enum] AnimeEpisodeType`](#enum-animeepisodetype)
			- [`[Enum] AiringStatus`](#enum-airingstatus)
		- [爬取数据库规范](#爬取数据库规范)
			- [关于`id`](#关于id)
			- [`Spider: Tables`](#spider-tables)
			- [`Spider: [Table] anime`](#spider-table-anime)
			- [`Spider: [Table] anime-episode`](#spider-table-anime-episode)
			- [`Spider: [Table] anime-name`](#spider-table-anime-name)
			- [`Spider: [Table] log`](#spider-table-log)
			- [`Spider: [Table] request-failed`](#spider-table-request-failed)
			- [`Spider: [Table] cache`](#spider-table-cache)
		- [内部数据库规范](#内部数据库规范)
			- [内部`ID` (`Nichijou ID`)](#内部id-nichijou-id)
			- [`Nichijou: Tables`](#nichijou-tables)
			- [`Nichijou: [Table]`](#nichijou-table)
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



### 自定义数据结构

#### `[Object] RelatedItem`

| Field  |   Type   | Nullable |
| :----: | :------: | :------: |
| `name` | `String` |    ❌     |
| `url`  | `String` |    ❌     |

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

存在两类`id`:
- `aid`: `anime id`
- `eid`: `episode id`

如果数据源的`id`不是数型的，那么应该进行`MD5`的哈希，取低32位 (共128位) 进行储存。由于数据量不是特别大，所以碰撞的概率较低。

#### `Spider: Tables`

关于数据库的`tables`:

- `anime`
- `anime-episode`
- `anime-name`

|       名称       | Required |              Description              |
| :--------------: | :------: | :-----------------------------------: |
|     `anime`      |    ✅     |             动画详细信息              |
| `anime-episode`  |    ✅     |           动画分集详细信息            |
|   `anime-name`   |    ✅     |             动画名称汇总              |
|      `log`       |    ✅     |                 日志                  |
| `request-failed` |    ✅     |        失败请求 (可以进行重试)        |
|     `cache`      |    ✅     |               蜘蛛缓存                |
|       `id`       |    ❌     | 精简的`id`表, 可以附上`URL`和基本信息 |

上述Required的数据表需要存在，用于统一管理检索。如有必要，可以额外建立数据表 (如`id`表)。

#### `Spider: [Table] anime`

**`Primary Key`: `aid`**

|   名称    |  数据类型  | 长度/集合 | 无符号 | Nullable |          描述           |
| :-------: | :--------: | :-------: | :----: | :------: | :---------------------: |
|   `aid`   |   `INT`    |     /     |   ✅    |    ❌     |  数据源番剧的唯一`id`   |
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

#### `Spider: [Table] anime-episode`

**`Primary Key`: `eid`**

|    名称    |  数据类型  | 长度/集合 | 无符号 | Nullable |           描述            |
| :--------: | :--------: | :-------: | :----: | :------: | :-----------------------: |
|   `eid`    |   `INT`    |     /     |   ✅    |    ❌     |   数据源剧集的唯一`id`    |
|   `aid`    |   `INT`    |     /     |   ✅    |    ❌     |   数据源番剧的唯一`id`    |
|   `url`    | `LONGTEXT` |     /     |   /    |    ❌     |           链接            |
|   `name`   | `VARCHAR`  |    200    |   /    |    ✅     |           原名            |
| `name_cn`  | `VARCHAR`  |    200    |   /    |    ✅     |          中文名           |
|   `type`   |   `INT`    |     /     |   ✅    |    ❌     | `[Enum] AnimeEpisodeType` |
|   `sort`   |   `INT`    |     /     |   ✅    |    ❌     |    当前`type`中多少话     |
|  `status`  |   `INT`    |     /     |   ✅    |    ❌     |         放送状态          |
| `duration` |   `INT`    |     /     |   ✅    |    ✅     |         时长 (秒)         |
|   `date`   |   `DATE`   |     /     |   /    |    ✅     |         放送日期          |
|   `desc`   | `LONGTEXT` |     /     |   /    |    ✅     |           简介            |

#### `Spider: [Table] anime-name`

**`Primary Key`: `aid`**

|  名称  | 数据类型  | 长度/集合 | 无符号 | Nullable |         描述         |
| :----: | :-------: | :-------: | :----: | :------: | :------------------: |
| `aid`  |   `INT`   |     /     |   ✅    |    ❌     | 数据源番剧的唯一`id` |
| `name` | `VARCHAR` |    200    |   /    |    ✅     |         原名         |

#### `Spider: [Table] log`

|   名称    |  数据类型  | 长度/集合 | 无符号 | Nullable | 描述  |
| :-------: | :--------: | :-------: | :----: | :------: | :---: |
|  `time`   | `DATETIME` |     /     |   /    |    ❌     | 时间  |
| `content` | `LONGTEXT` |     /     |   /    |    ❌     | 内容  |

#### `Spider: [Table] request-failed`

|   名称   |  数据类型  | 长度/集合 | 无符号 | Nullable |        描述        |
| :------: | :--------: | :-------: | :----: | :------: | :----------------: |
|   `id`   |   `INT`    |     /     |   ✅    |    ❌     | 失败任务的相应`id` |
| `spider` | `VARCHAR`  |    20     |   /    |    ❌     |     蜘蛛的名称     |
|  `desc`  | `LONGTEXT` |     /     |   /    |    ✅     |      错误内容      |

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

放长远眼光看，为了稳定、可持续的发展，建立一套内部自己的`ID`是十分有必要的。

#### `Nichijou: Tables`

#### `Nichijou: [Table]`



### 发布数据规范


### 发布数据方式

