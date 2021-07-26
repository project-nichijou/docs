# 番剧数据库间的匹配

## `Matcher` 实现细节

新增数据源中的`name`字段将会与**内部数据库**进行模糊匹配，计算出编辑距离 (Levenshtein距离)。如果某一条目的任意一个`name`可以精确匹配 (即编辑距离为0)，则确认匹配，反之，将记录下与每条`name`前五小的编辑距离。

此外，由于某些相似的条目可能会有相同的别名，所以在进行`Matcher`匹配之前，我们会筛选出重复项。匹配的过程当中，如果某一项为重复项，那么他的编辑距离将被设置为`-1`，同样需要进行人工校验（其他别名成功匹配除外）。关于这一点，可以参考<a href="#/server/anime-database/nichijou-db?id=nichijou-table-match_fail">`Nichijou: [Table] match_fail`</a>
