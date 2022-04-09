# 6.824关键问题记录

1. Server处理结束一个请求后宕机，Client认为请求失败重新发起请求，怎么避免Server操作重复？
2. Raft选举机制怎么描述？
3. 哪些信息需要持久化？为什么要持久化这些信息？([参考答案](https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/lecture-07-raft2/7.4-chi-jiu-hua-persistent))
4. applyIndex 和 commitIndex 的关系以及区别，是否需要持久化?
5. 如何实现数据迁移？
6. 乱序RPC怎么解决？
7. Figure8的理解？
8. 日志压缩的过程？
9. RPC怎么实现的？
10. 持久化如何编码的？

