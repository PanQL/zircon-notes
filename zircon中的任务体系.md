该文档关注zircon中thread、process、job相关的设计思路和数据结构实现  

# thread、process、job的简单描述  
*  一个process：管理多个thread
* 一个job管理：多个process、若干个child job、application
* 各个job之间呈现以root job为根的树状结构，即除root job外每一个job都有且仅有一个parent job。
