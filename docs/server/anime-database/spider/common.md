`repo site`: https://github.com/project-nichijou/spider-common/

- [介绍](#介绍)
- [结构](#结构)
- [使用方法](#使用方法)
- [接口](#接口)
	- [`common.spiders.common_spider.CommonSpider`](#commonspiderscommon_spidercommonspider)
		- [`use_cookies`](#use_cookies)
		- [`initialize`](#initialize)
		- [`init_normal_datasource`](#init_normal_datasource)
		- [`init_fail_datasource`](#init_fail_datasource)
	- [`common.items.common_item.CommonItem`](#commonitemscommon_itemcommonitem)
		- [`table`](#table)
		- [`primary_keys`](#primary_keys)
		- [`_url`](#_url)
		- [`use_fail`](#use_fail)
	- [关于`cache`](#关于cache)

# 介绍

本项目是[Project Nichijou](https://project-nichijou.github.io/docs)的一个子项目。根据[内部规范](https://project-nichijou.github.io/docs/#/./server/anime-database/spider)实现的基于[Scrapy](https://scrapy.org/)二次开发的爬虫框架。

# 结构

```
common
├── cache
│   ├── cache_maker.py              # cache 生产
│   └── cache_response.py           # 封装过的cache Response
├── config
│   ├── settings.py                 # 配置文件
│   └── settings_template.py        # 配置模板
├── cookies
│   ├── cookies.json                # cookies
│   ├── cookies.json.backup         # cookies 备份
│   ├── cookies_io.py               # cookies的IO封装
│   └── cookies_template.json       # cookies 模板
├── database
│   ├── database.py                 # 数据库
│   └── database_command.py         # 根据 [规范] 封装的 建表命令
├── items                           # 根据 [规范] 封装的 Item
│   ├── anime_item.py
│   ├── anime_name_item.py
│   ├── cache_item.py
│   ├── common_item.py              # Item 自定义父类
│   ├── episode_item.py
│   ├── episode_name_item.py
│   ├── fail_request_item.py
│   └── log_item.py
├── middlewares
│   ├── cache_middleware.py         # 请求缓存中间件
│   └── cookie_middleware.py        # cookies持久化中间件
├── pipelines
│   └── storing_pipeline.py         # 储存管道
├── spiders
│   └── common_spider.py            # Spider 自定义父类
└── utils
    ├── ac.py                       # AC 自动机封装
    ├── checker.py                  # 数据有效性封装
    ├── datetime.py                 # 日期时间格式封装
    ├── formatter.py                # 格式化工具封装
    ├── hash.py                     # 哈希工具封装
    └── logger.py                   # 日志封装
```

# 使用方法

1. 根据下面的`template`进行配置 (复制到同目录并重命名)
   - `common/cookies/cookies_template.json`
   - `common/config/settings_template.py`
2. 在主项目中配置`scrapy`的配置文件，重点有如下字段：
	```python
	DOWNLOADER_MIDDLEWARES = {
		'scrapy.downloadermiddlewares.cookies.CookiesMiddleware': None,
		'common.middlewares.cookie_middleware.CommonCookiesMiddleware': 920,
		'common.middlewares.cache_middleware.CommonCacheMiddleware': 930,
	}
	ITEM_PIPELINES = {
		'common.pipelines.storing_pipeline.CommonStoringPipeline': 300,
	}
	```
	注意：上面的配置不是必须的

可以考虑如下几种使用方式:
- 子类继承父类，自定义某些字段，覆写值
- 传参使用。注意：`Item`只有变量名为`_`开始的才能够作为属性直接修改，否则需要通过`dict`的方式。

# 接口

## `common.spiders.common_spider.CommonSpider`

- `parent`: `scrapy.Spider`

### `use_cookies`

- `type`: `boolean`
- `desc`: 为`True`则为此蜘蛛启用`cookies`组件，为`False`则不启用。注意：启用的前提是在`settings`中配置了`middlewares`

### `initialize`

- `type`: `function`
- `desc`: 初始化`spider`

### `init_normal_datasource`

- `type`: `function`
- `desc`: 初始化正常情况下的数据源

### `init_fail_datasource`

- `type`: `function`
- `desc`: 初始化失败重试情况下的数据源

## `common.items.common_item.CommonItem`

- `parent`: `scrapy.Item`

### `table`

- `type`: `str`
- `desc`: 此`Item`将被保存到的数据表

### `primary_keys`

- `type`: `list`
- `desc`: 存入数据表的`primary_keys` (主键)，用于`update`数据，若此项缺失，则会直接覆写

### `_url`

- `type`: `str`
- `desc`: 产生该`Item`请求的`url`，用于删除`fail`记录

### `use_fail`

- `type`: `boolean`
- `desc`: 此`Item`是否回进行重试，或重试时是否需要删除失败记录

## 关于`cache`

`CommonCacheMiddleware`只处理了`cache`的读取，`cache`的写入需要在`Spider`中实现。可以使用`common/cache/cache_maker.py`当中封装过的函数实现。
