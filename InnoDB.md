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
* 合并插入缓冲（insert buffer）并不是每秒都发生的，InnoDB储存引擎会判断前一秒内发生的IO次数是否小于五次，如果小于5此，InnoDB认为当前的IO压力很小，可以执行合并插入缓冲操作。
* 刷新100个脏页也不是每秒都发生的，InnoDB储存引擎会判断当想·前缓冲池中脏页的比例是否超过配置文件innodb_max_dirty_pages_pct这个参数，如果超过这个阈值innodb储存引擎认为需要做磁盘同步操作，将100个脏页写入磁盘。
* 接下来十秒的操作：
* 刷新100个脏页到磁盘（可能）
* 合并至多5个插入缓冲（总是）
* 将日志缓冲刷新到磁盘（总是）
* 删除无用的undo页（总是）
* 刷新100个脏页或者10个脏页到磁盘（总是）
* 产生一个检查点
### master thread 的潜在问题
* 从master thread 的伪代码来看，无论何时inndb的储存引擎最多都只会刷新100个赃页到磁盘，合并20个插入缓冲。如果在密集写的应用程序中，每秒可能会产生大于100个脏页或是产生大于20个的插入缓冲，此时master thread会应付不过来，即使磁盘能在一秒处理多余100个页的写入和20个插入缓冲的合并，由于hard coding,master thread 也只会选择100个脏页和合并20个插入缓冲。同时当发生宕机时需要回复是，由于很多数据还没有刷新会磁盘，所以导致恢复要很快的时间，尤其是对insert buffer
* 另一个问题是参数innodb_max_dirty_page_pct的默认值，该值得默认值为90，意味着脏页占缓冲池的90%，innodb在没一秒刷新缓冲池和flush loop时会判断这个值，如果大于innodb_max_dirty_pages_pct才刷新100个脏页。如果你有很大的内存或者你的数据库服务器压力很大，这时脏页的舒心速度反而可能会降低。
* InnoDB Plugin 带来innodb_adaptive_flush(自适应刷新)，该值影响每一秒脏页刷新的数量。原来刷新的规则是：如果脏页在缓冲池所占的比例小于——innodb_max_dirty_pages_pct时不刷新脏页。大于innodb_max_dirty_pages_pct是刷新100个脏页。innodb_adaptive_flushing参数的引入，innodb会通过一个buf_flush_get_desired_flush_rate的函数来判断需要刷新脏页最合适的数量。
* buf_flush_get_desired_flush_rate是通过判断产生重做日志的速度来判断最合适刷新脏页的数量。
