`repo site`: https://github.com/project-nichijou/nichijou.server.spider.anidb-index

- [介绍](#介绍)
- [环境](#环境)
- [配置方法](#配置方法)
- [使用方法](#使用方法)

# 介绍

本项目作为项目[Project Nichijou](https://github.com/project-nichijou)中的子项目，是[AniDB](anidb.net)的数据库标题索引，用于构建番剧数据库。完整内容详见: https://project-nichijou.github.io/docs

本项目根据[内部规范](https://project-nichijou.github.io/docs/#/./server/anime-database/spider)，基于[内部框架](https://github.com/project-nichijou/nichijou.server.spider.common)进行开发。

本repo只包含：
- 从[官方API](https://wiki.anidb.net/API)获取标题`XML`数据
- 写入数据库

# 环境

- MySQL 5.7.4 +
- Python 3.6 +
- mysql-connector-python
- click

# 配置方法

`common`子项目的配置，详见[这里](https://github.com/project-nichijou/nichijou.server.spider.common#%E4%BD%BF%E7%94%A8%E6%96%B9%E6%B3%95)。由于本项目比较简单，所以只需要配置数据库字段即可。

# 使用方法

通过CLI调用，下面是说明：

```
$ python3 main.py --help
Usage: main.py [OPTIONS] COMMAND [ARGS]...

Options:
  --help  Show this message and exit.

Commands:
  dellog    delete loggings in the database.
  download  download the anidb title index database dump
  parse     download then save the parse and save the result into the...
```

```
$ python3 main.py download --help
Usage: main.py download [OPTIONS]

  download the anidb title index database dump

Options:
  --url TEXT      use other url to download database instead of the one in the
                  configuration file
  --ignore_cache  whether to ignore cache files
  --help          Show this message and exit.
```

```
$ python3 main.py dellog --help
Usage: main.py dellog [OPTIONS]

  delete loggings in the database.

Options:
  --before TEXT  delete the loggings which are before the time in the
                 database. default is None, which means delete all. data
                 format: YYYY-MM-DD hh:mm:ss
  --help         Show this message and exit.
```
