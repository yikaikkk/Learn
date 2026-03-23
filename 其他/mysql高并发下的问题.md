# 高并发下的mysql问题

## 读写混杂的情况
对于读写混杂的情况，一般使用读写分离可以减少某一个节点的负载，从而降低可能出现的问题，在之前的一个博客上提到过，但是如果遇到大事务，可能会导致主从延迟
[mysql读写分离](http://guilinz.online/articles/181)

## 写多的情况

### CPU + IO 飙升

现象
	•	CPU 直接打满（尤其是 user + sys）
	•	磁盘 IO（尤其是写 IO）飙升
	•	iowait 明显上升

原因

MySQL 的写不是简单写一次：
	1.	写 redo log（顺序写）
	2.	写 binlog（顺序写）
	3.	修改 buffer pool（内存）
	4.	后台刷脏页到磁盘（随机写）

本质是：

一次写请求 ≈ 多次 IO + 内存操作

### 

现象
	•	RT（响应时间）突然变高
	•	TPS 上不去（甚至下降）
	•	大量线程处于：

### redo log 写满

现象
	•	MySQL 突然卡住
	•	QPS 掉到接近 0
	•	TPS 波动剧烈

原因

redo log 是循环写的（固定大小，比如 4GB）

当写入速度 > 刷盘速度：

redo log 被写满 → 必须等刷盘

### binlog 成为瓶颈（主从 / CDC 场景）

现象
	•	主库还能扛
	•	从库严重延迟（seconds behind master 很大）

原因

binlog：
	•	单线程写（早期更明显）
	•	从库是单线程 apply（或有限并行）

👉 10000 QPS 更新：
	•	主库：写 binlog OK
	•	从库：apply 跟不上

结果：
	•	数据延迟
	•	读写不一致
