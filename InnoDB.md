#innoDB体系架构
后台线程的主要作用是负责刷新内存池中的数据，保证缓冲池中的内存缓冲是最近的数据。此外，将已修改的数据文件刷新到磁盘中。
##后台线程
* InnoDB储存引擎主要是在一个master thread线程上实现的所有功能
* 默认情况下InnoDB储存引擎的后台线程有七个，4个IO thread，1个master thread，一个锁监控线程一个错误监控线程。
##内存
* InnoDB储存引擎内存由：<b>缓冲池，重做日志池，额外的内存池</b>
* 缓冲占用的内存最大，用来存放各种数据的缓存。
* InnoDB储存引擎总是将数据库文件按页读取到缓冲池中，然后按照最近最少使用算法来保留在缓冲池中的缓冲数据。如果数据库文件需要修改，总是修改在缓冲池中的页（发生修改后的页为脏页）然后按照一定的频率将缓冲池中的脏页刷新到文件。
* 缓冲池中的缓冲数据类型有：索引页，数据页，undo页，插入缓冲页，自适应哈希索引页，InnoDB储存的锁信息，数据字典信息
##master源码分析
* master线程级别最高，其内部由几个循环组成：主循环（loop），后台循环（background loop），刷新循环（flush loop），暂停循环（suspend loop）
* loop 主要有两个操作：每秒操作和每十秒的操作
void mster_thread(){
    loop:
    for(int i=0;i<10;i++){
    do thing once per secend
    sleep 1 second if necessary
    }
    do thing one per ten second
    goto loop
}
* 每秒的操作包括
1. 日志缓冲刷新到磁盘，即使这个事务还没有提交（总是）
2. 合并插入缓冲（可能）
3. 至多刷新100个InnoDB的缓存池中的脏页到到磁盘（可能）
4. 如果当前没有活动切换到background loop（可能）
* 即使某个事务还没有提交，innoDB储存引擎任然会每秒将重做日志缓冲中的内容刷新到重做日志文件中。这个很好的解释了为什么再打的事务commit的时间也很快