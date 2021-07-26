`repo site`: https://github.com/project-nichijou/anime-database-matcher/

- [介绍](#介绍)
- [原理](#原理)
- [配置方法](#配置方法)
- [使用方法](#使用方法)

# 介绍

本项目为[Project Nichijou](https://github.com/project-nichijou)中的子项目，进行不同数据源数据库之间的匹配工作。

# 原理

参考：
- [文档/Matcher实现细节](https://project-nichijou.github.io/docs/#/./server/anime-database/matcher)

# 配置方法

- `database/database_settings_template.py`复制、重命名为`database_settings.py`，并配置相应字段。
- `plan.txt`中记录需要匹配的数据库名称，用换行分隔

# 使用方法

```
$ python3 main.py 
Usage: main.py [OPTIONS] COMMAND [ARGS]...

Options:
  --help  Show this message and exit.

Commands:
  match  match `target`
  run    perform matching task
```

```
$ python3 main.py run --help
Usage: main.py run [OPTIONS]

  perform matching task

Options:
  -f, --config PATH  match configuration file. `plan.txt` will be used if not specified.
  --help             Show this message and exit.
```
