# Es的分页

# 背景
在之前的一个项目中，需要从es中导出数据，所以会涉及到es的分页，以下是es的几种分页方式

## from+size

在es中，from+size的方式和mysql中的offest+limit的方式相似

from 和 size 是两个控制分页查询的参数。from 参数用于指定从第几个文档开始返回结果，size 参数用于指定返回的结果集的大小。

From + size 查询优点
支持随机翻页。
From + size 查询缺点
受制于 max_result_window 设置，不能无限制翻页。

存在深度翻页问题，越往后翻页越慢。

# search_after

search_after 查询本质：使用前一页中的一组排序值来检索匹配的下一页。

前置条件：使用 search_after 要求后续的多个请求返回与第一次查询相同的排序结果序列。也就是说，即便在后续翻页的过程中，可能会有新数据写入等操作，但这些操作不会对原有结果集构成影响。

如何实现呢？

可以创建一个时间点 Point In Time（PIT）保障搜索过程中保留特定事件点的索引状态。

Point In Time（PIT）是 Elasticsearch 7.10 版本之后才有的新特性。

PIT的本质：存储索引数据状态的轻量级视图。

search_after 优点
不严格受制于 max_result_window，可以无限制往后翻页。
ps：不严格含义：单次请求值不能超过 max_result_window；但总翻页结果集可以超过。

search_after 缺点
只支持向后翻页，不支持随机翻页。
search_after 适用场景
类似：今日头条分页搜索  https://m.toutiao.com/search
不支持随机翻页，更适合手机端应用的场景。


# scroll

相比于 From + size 和 search_after 返回一页数据，Scroll API 可用于从单个搜索请求中检索大量结果（甚至所有结果），其方式与传统数据库中游标（cursor）类似。

如果把  From + size 和 search_after 两种请求看做近实时的请求处理方式，那么 scroll 滚动遍历查询显然是非实时的。数据量大的时候，响应时间可能会比较长。

  scroll 查询优点
支持全量遍历。
ps：单次遍历的 size 值也不能超过 max_result_window 大小。
    scroll 查询缺点
响应时间非实时。

保留上下文需要足够的堆内存空间。

scroll 查询适用场景

全量或数据量很大时遍历结果数据，而非分页查询。

官方文档强调：不再建议使用scroll API进行深度分页。如果要分页检索超过 Top 10,000+ 结果时，推荐使用：PIT + search_after。

