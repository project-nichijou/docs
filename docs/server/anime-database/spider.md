# 构建番剧数据库所用的`Spider`

## `Spider` 推荐实现

使用`Python`的`Scrapy`框架进行开发。我们将提供部分`Converter`以及数据库相关的，可以共用的代码。

`repo`: https://github.com/project-nichijou/spider-common/

## 爬取数据库规范

由于我们需要爬取多个站点来丰富数据，同时需要确保极强的可扩展性，我们势必需要确定一套持久化、可扩展的存储结构。下面将简述爬取得到后的数据的转换要求 (关系到`Converter`的实现)，以及数据库各张`table`的具体建立方式。

### 关于`id`

如果数据源的`id`不是数型的，那么应该进行`MD5`的哈希，取低32位 (共128位) 进行储存。由于数据量不是特别大，所以碰撞的概率较低。

### `Spider: Tables`

关于数据库的`tables`:

|       名称       | Required |              Description              |
| :--------------: | :------: | :-----------------------------------: |
|     `anime`      |    ✅     |             动画详细信息              |
|    `episode`     |    ✅     |           动画分集详细信息            |
|   `anime_name`   |    ✅     |             动画名称汇总              |
|  `episode_name`  |    ✅     |             剧集名称汇总              |
|      `log`       |    ✅     |                 日志                  |
| `request_failed` |    ✅     |        失败请求 (可以进行重试)        |
|     `cache`      |    ✅     |               蜘蛛缓存                |
|       `id`       |    ❌     | 精简的`id`表, 可以附上`URL`和基本信息 |

上述Required的数据表需要存在，用于统一管理检索。如有必要，可以额外建立数据表 (如`id`表)。

### `Spider: [Table] anime`

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
|  `sites`  | `LONGTEXT` |     /     |   /    |    ✅     |  `json`: `[AnimeSite]`  |

注: 
- `date`之所以是`DATE`而不是`VARCHAR`，是因为诸如`bangumi`的网站里面虽然`date`有额外信息，比方说特典日期，但是已经包含在`meta`字段了，规范格式有助于后期处理。
- `meta`中不同分条一般用换行隔开
- `tags`中不同条目用换行隔开
- `type`可能值: `TV`, `OVA`, ...

### `Spider: [Table] episode`

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
|  `sites`   | `LONGTEXT` |     /     |   /    |    ✅     |   `json`: `[AnimeSite]`   |

### `Spider: [Table] anime_name`

**`Primary Key`: `id` & `name`**

|  名称  | 数据类型  | 长度/集合 | 无符号 | Nullable |         描述         |
| :----: | :-------: | :-------: | :----: | :------: | :------------------: |
|  `id`  |   `INT`   |     /     |   ✅    |    ❌     | 数据源番剧的唯一`id` |
| `name` | `VARCHAR` |    200    |   /    |    ❌     |       番剧名称       |

### `Spider: [Table] episode_name`

**`Primary Key`: `id` & `type` & `sort` & `name`**

|  名称  | 数据类型  | 长度/集合 | 无符号 | Nullable |           描述            |
| :----: | :-------: | :-------: | :----: | :------: | :-----------------------: |
|  `id`  |   `INT`   |     /     |   ✅    |    ❌     |   数据源番剧的唯一`id`    |
| `type` |   `INT`   |     /     |   ✅    |    ❌     | `[Enum] AnimeEpisodeType` |
| `sort` |   `INT`   |     /     |   ✅    |    ❌     |    当前`type`中多少话     |
| `name` | `VARCHAR` |    200    |   /    |    ❌     |         剧集名称          |


### `Spider: [Table] log`

|   名称    |  数据类型  | 长度/集合 | 无符号 | Nullable | 描述  |
| :-------: | :--------: | :-------: | :----: | :------: | :---: |
|  `time`   | `DATETIME` |     /     |   /    |    ❌     | 时间  |
| `content` | `LONGTEXT` |     /     |   /    |    ❌     | 内容  |

### `Spider: [Table] request_failed`

**`Primary Key`: `spider` & `url_md5`**

|   名称    |  数据类型  | 长度/集合 | 无符号 | Nullable |       描述       |
| :-------: | :--------: | :-------: | :----: | :------: | :--------------: |
| `url_md5` | `VARCHAR`  |    32     |   /    |    ❌     | `url`的大写`MD5` |
|   `url`   | `LONGTEXT` |     /     |   /    |    ❌     | 失败任务的`url`  |
| `spider`  | `VARCHAR`  |    20     |   /    |    ❌     |    蜘蛛的名称    |
|  `desc`   | `LONGTEXT` |     /     |   /    |    ✅     |     错误内容     |
| `params`  | `LONGTEXT` |     /     |   /    |    ✅     |     其他参数     |

注:
- `spider`字段，以`Scrapy`为例，应该为`<spider>.name`
- `desc`字段，如果可以，尽可能附带堆栈信息
- `params`字段主要为重试时需要附带的参数，可以用`json`存

### `Spider: [Table] cache`

**`Primary Key`: `url_md5`**

|   名称    |  数据类型  | 长度/集合 | 无符号 | Nullable |       描述       |
| :-------: | :--------: | :-------: | :----: | :------: | :--------------: |
| `url_md5` | `VARCHAR`  |    32     |   /    |    ❌     | `url`的大写`MD5` |
|   `url`   | `LONGTEXT` |     /     |   /    |    ❌     |  缓存内容的链接  |
| `expire`  | `DATETIME` |     /     |   /    |    ❌     |   缓存过期时间   |
| `content` | `LONGBLOB` |     /     |   /    |    ✅     |     缓存内容     |
