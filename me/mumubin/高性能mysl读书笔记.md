## 第一章 Mysql架构和历史
1.什么是间隙锁，怎样防止幻读的？
2.什么是聚族索引？
3.InnoDB事务模型和锁 补充阅读
4.日志类型适合MyISAM压缩表
5.尽量不要使用多个存储引擎
6.InnoDB vs MyISAM
- InnoDB 支持事务
- InnoDB 支持行锁，MVCC
- MyISAM崩溃后发生损坏的概率比InnoDB要高很多（MyISAM只将数据写到内存中，然后等待操作系统定期将数据刷出到磁盘上）
- InnoDB支持热备份
- MyISAM支持地理空间(only)
- InnoDB 3-5TB
- 不同类型存储引擎转换
    -   锁表：ALTER TABLE mytable ENGINE=InnoDB;（如果将一张InnoDB表转换为MyISAM，然后再转换回InnoDB，原InnoDB表上所有的外键将丢失。）
    -   mysqldump,因为会drop不安全
    -   mysql> CREATE TABLE innodb_table LIKE myisam_table;
        mysql> ALTER TABLE innodb_table ENGINE=InnoDB;
        mysql> INSERT INTO innodb_table SELECT * FROM myisam_table;
    -   大量数据分批:  mysql> START TRANSACTION;
          mysql> INSERT INTO innodb_table SELECT * FROM myisam_table
              -> WHERE id BETWEEN x AND y;
          mysql> COMMIT;
    - Percona Toolkit提供了一个pt-online-schema-change的工具（基于Facebook的在线schema变更技术

7.日志类应用:MyISAM支持地理空间
8.SELECT COUNT（*） FROM table; 执行速度受存储引擎影响
9.Infobright是MySQL数据仓库最成功的解决方案
## 第二章 基准测试
10：基准测试 sysbench
收集测试数据
```shell script
    #!/bin/sh
    
    INTERVAL=5
    PREFIX=$INTERVAL-sec-status
    RUNFILE=/home/benchmarks/running
    mysql -e 'SHOW GLOBAL VARIABLES' >> mysql-variables
    while test -e $RUNFILE; do
       file=$(date +％F_％I)
       sleep=$(date +％s.％N | awk "{print $INTERVAL - (\$1 ％ $INTERVAL)}")
       sleep $sleep
       ts="$(date +"TS ％s.％N ％F ％T")"
       loadavg="$(uptime)"
       echo "$ts $loadavg" >> $PREFIX-${file}-status
       mysql -e 'SHOW GLOBAL STATUS' >> $PREFIX-${file}-status &
       echo "$ts $loadavg" >> $PREFIX-${file}-innodbstatus
       mysql -e 'SHOW ENGINE INNODB STATUS\G' >> $PREFIX-${file}-innodbstatus &
       echo "$ts $loadavg" >> $PREFIX-${file}-processlist
       mysql -e 'SHOW FULL PROCESSLIST\G' >> $PREFIX-${file}-processlist &
       echo $ts
    done
    echo Exiting because $RUNFILE does not exist.
```
分析测试数据
```shell script

    #!/bin/sh
    
    # This script converts SHOW GLOBAL STATUS into a tabulated format, one line
    # per sample in the input, with the metrics divided by the time elapsed
    # between samples.
    awk '
        BEGIN {
           printf "#ts date time load QPS";
           fmt = " %.2f";
        }
        /^TS/ { # The timestamp lines begin with TS.
           ts = substr($2, 1, index($2, ".") - 1);
           load = NF - 2;
           diff = ts - prev_ts;
           prev_ts = ts;
           printf "\n%s %s %s %s", ts, $3, $4, substr($load, 1, length($load)-1);
        }
        /Queries/ {
            printf fmt, ($2-Queries)/diff;
            Queries=$2
        }
        ' "$@"

```
绘制图形
```shell script
 gnuplot> plot "QPS-per-5-seconds" using 5 w lines title"QPS"
```
尽可能地使所有测试过程都自动化，包括装载数据、系统预热、执行测试、记录结果等。
11。 pt-diskstats 捕获/proc/diskstats的数据为后续分析磁盘I/O使用
12. 现有工具
  - Web 服务
    - ab
    - http_load(简单测试推荐)
    ```shell script
      $ http_load -parallel 5 -seconds 10 urls.txt
        94 fetches, 5 max parallel, 4.75565e+06 bytes, in 10.0005 seconds
        50592 mean bytes/connection
        9.39953 fetches/sec, 475541 bytes/sec
        msecs/connect: 65.1983 mean, 169.991 max, 38.189 min
        msecs/first-response: 245.014 mean, 993.059 max, 99.646 min
        HTTP response codes:
          code 200 - 94
    ```
    - JMeter
  - Mysql
    - mysqlslap
    - Mysql Benchmark Suite
    - Super Smack
    - Database Test Suite
    - Percona's [TPCC-MySQL](https://github.com/Percona-Lab/tpcc-mysql) Tool（复杂测试推荐）
    - sysbench(简单测试推荐)
    ```shell script
        CPU测试
        [server1 ~]$ sysbench --test=cpu --cpu-max-prime=20000 run
        sysbench v0.4.8: multithreaded system evaluation benchmark
        ...
        Test execution summary:    total time:             61.8596s
    ```
    ```shell script
        IO测试
        sysbench --test=fileio --file-total-size=150G prepare
        sysbench --test=fileio --file-total-size=150G --file-test-mode=rndrw/
            --init-rng=on--max-time=300--max-requests=0 run
        sysbench --test=fileio --file-total-size=150G cleanup
     ```
     ```shell script
        OLTP基准测试
        生成数据
       $ sysbench --test=oltp --oltp-table-size=1000000 --mysql-db=test/
        --mysql-user=root prepare
        sysbench v0.4.8: multithreaded system evaluation benchmark
        
        No DB drivers specified, using mysql
        Creating table 'sbtest'...
        Creating 1000000 records in table 'sbtest'...
       执行测试
        $ sysbench --test=oltp --oltp-table-size=1000000 --mysql-db=test --mysql-user=root/
        --max-time=60 --oltp-read-only=on --max-requests=0 --num-threads=8 run
        sysbench v0.4.8: multithreaded system evaluation benchmark
     ```
    
## 第三章 服务器性能剖析
1. slow log - pt-query-digest
2. show profiles
3. Performance Schema
4. 间歇性问题排查
    - 以较高的频率比如一秒执行一次SHOW GLOBAL STATUS命令捕获数据
    ```shell script
        $ mysqladmin ext -i1 | awk '
            /Queries/{q=$4-qp;qp=$4}
            /Threads_connected/{tc=$4}
            /Threads_running/{printf "%5d %5d %5d\n", q, tc, $4}'
        2147483647   136  7
           798  136    7
           767  134    9
           828  134    7
           683  134    7
           784  135    7
           614  134    7
           108  134   24
           187  134   31
           179  134   28
          1179  134    7
          1151  134    7
          1240  135    7
          1000  135    7
    
    ```
   - SHOW PROCESSLIST
   ```shell script
        $ mysql -e 'SHOW PROCESSLIST\G' | grep State: | sort | uniq -c | sort -rn
            744  State:
            67   State: Sending data
            36   State: freeing items
             8   State: NULL
             6   State: end
             4   State: Updating
             4   State: cleaning up
             2   State: update
             1   State: Sorting result
             1   State: logging slow query
    
    ```
   大量的线程处于“freeing items”状态是出现了大量有问题查询的很明显的特征和指示。上面演示的这个例子是由于InnoDB内部的争用和脏块刷新所导致
   - 使用查询日志
5. pt-stalk: 触发器收集数据，当达到触发条件时能收集数据
6. 执行时间包括用于工作的时间和等待的时间。当一个未知问题发生时，一般来说有两种可能：服务器需要做大量的工作，从而导致大量消耗CPU；或者在等待某些资源被释放
7. pt-pmp:穷人解析器，分析gdb信息
8. pt-collect收集信息，通过ps-stalk来调用，系统中需要安装gdb和oprofile，mysqld也需要有调试符号信息
9. pt-mysql-summary和pt-summary的输出结果打包
10. pt-sift：快速检查收集到的样本数据的工具
11. oprofile怎么用？
12. gdb怎么用？
13.对于固态硬盘来说，其I/O平均等待时间一般不会超过1/4秒
14.vmstat/iostat
15.各种统计信息表USER_STATISTICS
16. strace -cfp pid
17. pt-ioprofile的工具就是使用strace来生成I/O活动的剖析报告的
18. Performance Schema

## 第4章　Schema与数据类型优化
1. 阅读Clare Churcher的Beginning Database Design
2. 通常情况下最好指定列为NOT NULL，除非真的需要存储NULL值。
3. MySQL可以为整数类型指定宽度，例如INT（11），对大多数应用这是没有意义的：它不会限制值的合法范围，只是规定了MySQL的一些交互工具（例如MySQL命令行客户端）用来显示字符的个数。对于存储和计算来说，INT（1）和INT（20）是相同的。
4. 因为Memory引擎不支持BLOB和TEXT类型，所以，如果查询使用了BLOB或TEXT列并且需要使用隐式临时表，有一个技巧是在所有用到BLOB字段的地方都使用SUBSTRING（column，length）将列值转换为字符串（在ORDER BY子句中也适用），这样就可以使用内存临时表了。但是要确保截取的子字符串足够短，不会使临时表的大小超过max_heap_table_size或tmp_table_size，超过以后MySQL会将内存临时表转换为MyISAM磁盘临时表
5. 避免使用临时表，如果EXPLAIN执行计划的Extra列包含”Using temporary”，则说明这个查询使用了隐式临时表。
6. 使用枚举（ENUM）代替字符串类型，排序是安装内置数字来的。
    - 枚举最不好的地方是，字符串列表是固定的，添加或删除字符串必须使用ALTER TABLE。
    - 当VARCHAR列和ENUM列进行关联时则慢很多
    ```shell script
       mysql> CREATE TABLE enum_test(
            -> e ENUM ('fish', 'apple', 'dog') NOT NULL
            -> );
        mysql> INSERT INTO enum_test(e) VALUES('fish'), ('dog'), ('apple')                                                            
    ```                                                 
7. IP地址不要用varchar(15),应该是32位无符号整数，INET_ATON()和INET_NTOA()来转换
8. Mysql 索引中存储NULL值，oracle不会
9. 范式化数据库，每个数据只会存在一次，反范式化数据库，存在数据冗余
    - 反范式化数据库，当数据比内存大时，因为数据冗余避免join，查询可能会快很多
10. Flexview帮忙处理view的工具
11. ON DUPLICATE KEY UPDATE，有则更新无则insert
11. Alter Table
    - https://launchpad.net/mysqlatfacebook
    - http://code.openark.org
    - Percona Toolkit
##第五章 创建高性能索引
1. 处理blob text 长char的查询索引
  - 建立一个冗余的hash(),crc32()，fnv64()等int函数字段，创建triger自动搞定
    ```shell script
    mysql> SELECT id FROM url WHERE url="http://www.mysql.com"
        ->    AND url_crc=CRC32("http://www.mysql.com");
    ```
  - 创建前缀索引，缺点：MySQL无法使用前缀索引做ORDER BY和GROUP BY，也无法使用前缀索引做覆盖扫描
  ```
    mysql> ALTER TABLE sakila.city_demo ADD KEY (city(7));
  ```
2. 索引合并策略有时候是一种优化的结果，但实际上更多时候说明了表上的索引建得很糟糕：  
    - 当出现服务器对多个索引做相交操作时（通常有多个AND条件），通常意味着需要一个包含所有相关列的多列索引，而不是多个独立的单列索引。
    - 当服务器需要对多个索引做联合操作时（通常有多个OR条件），通常需要耗费大量CPU和内存资源在算法的缓存、排序和合并操作上。特别是当其中有些索引的选择性不高，需要合并扫描返回的大量数据的时候。
    - 更重要的是，优化器不会把这些计算到“查询成本”（cost）中，优化器只关心随机页面读取。这会使得查询的成本被“低估”，导致该执行计划还不如直接走全表扫描
 
3. 聚簇索引并不是一种单独的索引类型，而是一种数据存储方式
4. Extra有filesort代表排序没有用到索引，花了大量时间
5. 重复索引是指在相同的列上按照相同的顺序创建的相同类型的索引。应该避免这样创建重复索引，发现以后也应该立即移除
    - Percona Toolkit中的pt-duplicate-key-checker
    - Percona Toolkit中的pt-duplicate-key-checker
6. 未使用的索引
    - 使用Percona Toolkit中的pt-index-usage，该工具可以读取查询日志，并对日志中的每条查询进行EXPLAIN操作，然后打印出关于索引和查询的报
    - 在Percona Server或者MariaDB中先打开userstates服务器变量（默认是关闭的），然后让服务器正常运行一段时间，再通过查询INFORMATION_SCHEMA.INDEX_STATISTICS就能查到每个索引的使用频率。
7. InnoDB在二级索引上使用共享（读）锁，但访问主键索引需要排他（写）锁。这消除了使用覆盖索引的可能性，并且使得SELECT FOR UPDATE比LOCK IN SHARE MODE或非锁定查询要慢很多。
8. 将需要做范围查询的列放到索引的后面，以便优化器能使用尽可能多的索引列
9. 避免多个范围条件
10. 如果InnoDB引擎的表出现了损坏，常见的类似错误通常是由于尝试使用rsync备份InnoDB导致的
11. 数据修复：InnoDB数据恢复工具箱（InnoDB Data Recovery Toolkit）
12. 执行计划是根据INFORMATION_SCHEMA的数据来估算的
13. 一旦关闭索引统计信息的自动更新，那么就需要周期性地使用ANALYZE TABLE来手动更新。否则，索引统计信息就会永远不变。如果数据分布发生大的变化，可能会出现一些很糟糕的执行计划。
14. 消除碎片化，可以通过执行OPTIMIZE TABLE或者导出再导入的方式来重新整理数据，不过最新版本InnoDB新增了“在线”添加和删除索引的功能，可以通过先删除，然后再重新创建索引的方式来消除索引的碎片化。
    可用先删除所有索引，然后重建表，最后重建索引，消除表和索引的碎片
15. 编写查询语句时应该尽可能选择合适的索引以避免单行查找、尽可能地使用数据原生顺序从而避免额外的排序操作，并尽可能使用索引覆盖查询。

## 第六章 查询性能优化
1. 查询优化、索引优化、库表结构优化需要齐头并进，一个不落。
2. 在尝试编写快速的查询之前，需要清楚一点，真正重要是响应时间。如果把查询看作是一个任务，那么它由一系列子任务组成，每个子任务都会消耗一定的时间。如果要优化查询，实际上要优化其子任务，要么消除其中一些子任务，要么减少子任务的执行次数，要么让子任务运行得更快
3. 每次看到SELECT *的时候都需要用怀疑的眼光审视，是不是真的需要返回全部的列？很可能不是必需的。取出全部列，会让优化器无法完成索引覆盖扫描这类优化，还会为服务器带来额外的I/O、内存和CPU的消耗。因此，一些DBA是严格禁止SELECT *的写法的，这样做有时候还能避免某些列被修改带来的问题
4. 查询开销
    - 响应时间：服务时间和排队时间，可以使用“快速上限估计”法来估算查询的响应时间是否合理
    - 扫描的行数： 扫描的行数对返回的行数的比率通常很小，一般在1:1和10:1之间，不过有时候这个值也可能非常非常大。
    - 返回的行数
5. 一般MySQL能够使用如下三种方式应用WHERE条件，从好到坏依次为：
   - 在索引中使用WHERE条件来过滤不匹配的记录。这是在存储引擎层完成的。
   - 使用索引覆盖扫描（在Extra列中出现了Using index）来返回记录，直接从索引中过滤不需要的记录并返回命中的结果。这是在MySQL服务器层完成的，但无须再回表查询记录。
   - 从数据表中返回数据，然后过滤不满足条件的记录（在Extra列中出现Using Where）。这在MySQL服务器层完成，MySQL需要先从数据表读出记录然后过滤。
   
6. Explain-Extra: 
    - “Using Where”表示MySQL将通过WHERE条件来筛选存储引擎返回的记录。
    - "Using index"使用索引覆盖扫描（在Extra列中出现了）来返回记录，直接从索引中过滤不需要的记录并返回命中的结果。这是在MySQL服务器层完成的，但无须再回表查询记录。
    - "Select tables optimized away" 它表示优化器已经从执行计划中移除了该表，并以一个常数取而代之。e.g. 要找到某一列的最小值，只需要查询对应B-Tree索引最左端的记录
    - "Impossible WHERE noticed after reading const tables" 提前终止查询
    - "Using filesort”" Mysql单表自己排序，未能使用索引排序，MySQL在进行文件排序的时候需要使用的临时存储空间可能会比想象的要大得多。原因在于MySQL在排序时，对每一个排序记录都会分配一个足够长的定长空间来存放。
                这个定长空间必须足够长以容纳其中最长的字符串，例如，如果是VARCHAR列则需要分配其完整长度；如果使用UTF-8字符集，那么MySQL将会为每个字符预留三个字节。我们曾经在一个库表结构设计不合理的案例中看到，排序消耗的临时空间比磁盘上的原表要大很多倍
    - "Using temporary;Using filesort" order by 多个列且跨表
    - "Not exists” 提前终止查询
    - “Using index for group-by”，表示这里将使用松散索引扫描(EXPLAIN SELECT actor_id, MAX(film_id)
                                                       -> FROM sakila.film_actor
                                                       -> GROUP BY actor_id\G)
                                    
    
7. 如果发现查询需要扫描大量的数据但只返回少数的行，那么通常可以尝试下面的技巧去优化它：
   - 使用索引覆盖扫描，把所有需要用的列都放到索引中，这样存储引擎无须回表获取对应行就可以返回结果了（在前面的章节中我们已经讨论过了）。
   - 改变库表结构。例如使用单独的汇总表（这是我们在第4章中讨论的办法）。
   - 重写这个复杂的查询，让MySQL优化器能够以更优化的方式执行这个查询
   
8. 切分大SQL
    ```shell script
    mysql> DELETE FROM messages WHERE created < DATE_SUB(NOW(),INTERVAL 3 MONTH);
    ```
    ```shell script
      rows_affected = 0
        do {
           rows_affected = do_query(
              "DELETE FROM messages WHERE created < DATE_SUB(NOW(),INTERVAL 3 MONTH)
              LIMIT 10000")
        } while rows_affected > 0 
    ```
9. 分解关联查询（减少join）
10. Mysql 执行sql的步骤
   1.客户端发送一条查询给服务器。
   2.服务器先检查查询缓存，如果命中了缓存，则立刻返回存储在缓存中的结果。否则进入下一阶段。
   3.服务器端进行SQL解析、预处理，再由优化器生成对应的执行计划。
   4.MySQL根据优化器生成的执行计划，调用存储引擎的API来执行查询。
   5.将结果返回给客户端。
   通信模式：半双工，max_allowed_packet就
11. SHOW FULL PROCESSLIST：State
    - sleep: 线程正在等待客户端发送新的请求。
    - query: 线程正在执行查询或者正在将结果发送给客户端。
    - lock: 在MySQL服务器层，该线程正在等待表锁。在存储引擎级别实现的锁，例如InnoDB的行锁，并不会体现在线程状态中。对于MyISAM来说这是一个比较典型的状态，但在其他没有行锁的引擎中也经常会出现。
    - Analyzing and statistics: 线程正在收集存储引擎的统计信息，并生成查询的执行计划
    - Copying to tmp table [on disk]: 线程正在执行查询，并且将其结果集都复制到一个临时表中，这种状态一般要么是在做GROUP BY操作，要么是文件排序操作，或者是UNION操作。如果这个状态后面还有“on disk”标记，那表示MySQL正在将一个内存临时表放到磁盘上
    - The thread is: 线程正在对结果集进行排序
    - Sending data: 这表示多种情况：线程可能在多个状态之间传送数据，或者在生成结果集，或者在向客户端返回数据                                   
   
12.MySQL能够处理的优化类型
    - 重新定义关联表的顺序
    - 将外连接转化成内连接   
    - 使用等价变换规则 （5=5 AND a>5）将被改写为a>5。
    - 优化COUNT()、MIN()和MAX()
    - 预估并转化为常数表达式(执行计划中ref为const)
    - 覆盖索引扫描
    - 子查询优化
    - 提前终止查询 （LIMIT，没有返回值）
    - 等值传播
    - 列表IN()的比较
13. 当前MySQL关联执行的策略很简单：MySQL对任何关联都执行嵌套循环关联操作，即MySQL先在一个表中循环取出单条数据，然后再嵌套循环到下一个表中寻找匹配的行，依次下去，直到找到所有表中匹配的行为止
14. MySQL的临时表是没有任何索引的，在编写复杂的子查询和关联查询的时候需要注意这一点。这一点对UNION查询也一样。
15. OUTER JOIN 什么意思
16. 预估成本。show  status  like  'last_query_cost';
17. 若是10个表的关联，那么共有3628800种不同的关联顺序！当搜索空间非常大的时候，优化器不可能逐一评估每一种关联顺序的成本。这时，优化器选择使用“贪婪”搜索的方式查找“最优”的关联顺序。实际上，当需要关联的表超过optimizer_search_depth的限制的时候，就会选择“贪婪”搜索模式了
18. 当查询需要所有列的总长度不超过参数max_length_for_sort_data时，MySQL使用“单次传输排序”，可以通过调整这个参数来影响MySQL排序算法的选择
19. 并不是所有关联子查询的性能都会很差。如果有人跟你说：“别用关联子查询”，那么不要理他。先测试，然后做出自己的判断
20. UNION的限制
```shell script
    (SELECT first_name, last_name
     FROM sakila.actor
     ORDER BY last_name)
    UNION ALL
    (SELECT first_name, last_name
     FROM sakila.customer
     ORDER BY last_name)
    LIMIT 20;

===>

  (SELECT first_name, last_name
     FROM sakila.actor
     ORDER BY last_name
     LIMIT 20)
    UNION ALL
    (SELECT first_name, last_name
     FROM sakila.customer
     ORDER BY last_name
     LIMIT 20)
    LIMIT 20;

从临时表中取出数据的顺序并不是一定的，所以如果想获得正确的顺序,还需要加上一个全局的ORDER BY和LIMIT操作。
```
21. MySQL无法利用多核特性来并行执行查询。很多其他的关系型数据库能够提供这个特性
22. MySQL并不支持哈希关联——MySQL的所有关联都是嵌套循环关联
23. min() 
```shell script
    mysql> SELECT actor_id FROM sakila.actor USE INDEX(PRIMARY)
        -> WHERE first_name = 'PENELOPE' LIMIT 1;

```
24. MySQL不允许对同一张表同时进行查询和更
25. 查询优化器的提示（hint）
    - DELAYED: 这个提示对INSERT和REPLACE有效。MySQL会将使用该提示的语句立即返回给客户端，并将插入的行数据放入到缓冲区，然后在表空闲时批量将数据写入。日志系统使用这样的提示非常有效，或者是其他需要写入大量数据但是客户端却不需要等待单条语句完成I/O的应用。这个用法有一些限制
    - STRAIGHT_JOIN: 这个提示可以放置在SELECT语句的SELECT关键字之后，也可以放置在任何两个关联表的名字之间。第一个用法是让查询中所有的表按照在语句中出现的顺序进行关联。第二个用法则是固定其前后两个表的关联顺序
                    当MySQL没能选择正确的关联顺序的时候，或者由于可能的顺序太多导致MySQL无法评估所有的关联顺序的时候，STRAIGHT_JOIN都会很有用。在后面这种情况，MySQL可能会花费大量时间在“statistics”状态，加上这个提示则会大大减少优化器的搜索空间。
                    
    - SQL_SMALL_RESULT和SQL_BIG_RESULT:这两个提示只对SELECT语句有效。它们告诉优化器对GROUP BY或者DISTINCT查询如何使用临时表及排序。SQL_SMALL_RESULT告诉优化器结果集会很小，可以将结果集放在内存中的索引临时表，以避免排序操作。如果是SQL_BIG_RESULT，则告诉优化器结果集可能会非常大，建议使用磁盘临时表做排序操作。
    - SQL_BUFFER_RESULT: 这个提示告诉优化器将查询结果放入到一个临时表，然后尽可能快地释放表锁。这和前面提到的由客户端缓存结果不同。当你没法使用客户端缓存的时候，使用服务器端的缓存通常很有效
    - SQL_CACHE和SQL_NO_CACHE 这个提示告诉MySQL这个结果集是否应该缓存在查询缓存中   
    - SQL_CALC_FOUND_ROWS 严格来说，这并不是一个优化器提示。它不会告诉优化器任何关于执行计划的东西。它会让MySQL返回的结果集包含更多的信息。查询中加上该提示MySQL会计算除去LIMIT子句后这个查询要返回的结果集的总数，而实际上只返回LIMIT要求的结果集
    
26. Mysql 优化参数

 - optimizer_search_depth：这个参数控制优化器在穷举执行计划时的限度。如果查询长时间处于“Statistics”状态，那么可以考虑调低此参数。
 - optimizer_prune_level ：该参数默认是打开的，这让优化器会根据需要扫描的行数来决定是否跳过某些执行计划。
 - optimizer_switch：这个变量包含了一些开启/关闭优化器特性的标志位。例如在MySQL 5.1中可以通过这个参数来控制禁用索引合并的特性
 
27. Mysql 升级可能造成hint不好用，或者参数不好用，建议使用Percona Toolkit中的pt-upgrade工具，就可以检查在新版本中运行的SQL是否与老版本一样，返回相同的结果
28. Count()
    - Count(列)不计NULL值
    - 可以利用业务上加统计表来优化
    - Explain的估计值来近似  
29. 关于子查询优化我们给出的最重要的优化建议就是尽可能使用关联查询代替，
30. 当无法使用索引的时候，GROUP BY使用两种策略来完成：使用临时表或者文件排序来做分组。对于任何查询语句，这两种策略的性能都有可以提升的地方。可以通过使用提示SQL_BIG_RESULT和SQL_SMALL_RESULT来让优化器按照你希望的方式运行。
31. 可以使用ORDER BY NULL，让MySQL不再进行文件排序。也可以在GROUP BY子句中直接使用DESC或者ASC关键字，使分组的结果集按需要的方向排序
32. 什么意思？GROUP BY WITH ROLLUP？汇总函数 https://cloud.tencent.com/developer/article/1345742
33. Percona Toolkit中的pt-query-advisor能够解析查询日志、分析查询模式，然后给出所有可能存在潜在问题的查询，并给出足够详细的建
34. 赋值符号:=的优先级非常低，所以需要注意，赋值表达式应该使用明确的括号。使用未定义变量不会产生任何语法错误，如果没有意识到这一点，非常容易犯错
    - 我们将赋值语句放到LEAST()函数中，这样就可以在完全不改变排序顺序的时候完成赋值操作（在上面例子中，LEAST()函数总是返回0），这样的函数还有GREATEST()、LENGHT()、ISNULL()、NULLIFL()、IF()和COALESCE()，
35. Mysql不支持 UPDATE RETURNING
36. 书籍推荐
    - SQL and Relational Theory:How to Write Accurate SQL Code（http://shop.xreilly.com/product/0636920022879.do）
    - Bill Karwin的书SQL Antipatterns
37. select  for update，都应该用乐观锁代替
```sql
    SET AUTOCOMMIT = 1;
    COMMIT;
    UPDATE unsent_emails
       SET status = 'claimed', owner = CONNECTION_ID()
       WHERE owner = 0 AND status = 'unsent'
       LIMIT 10;
    SET AUTOCOMMIT = 0;
    SELECT id FROM unsent_emails
       WHERE owner = CONNECTION_ID() AND status = 'claimed';

```

38. PostgreSQL擅长空间信息处理

## 第七章 Mysql高级特性
1. MySQL实现分区表的方式——对底层表的封装——意味着索引也是按照分区的子表定义的，而没有全局索引。这和Oracle不同，在Oracle中可以更加灵活地定义索引和表是否进行分区。
```sql

    CREATE TABLE sales (
       order_date DATETIME NOT NULL,
       -- Other columns omitted
    ) ENGINE=InnoDB PARTITION BY RANGE(YEAR(order_date)) (
        PARTITION p_2010 VALUES LESS THAN (2010),
        PARTITION p_2011 VALUES LESS THAN (2011),
        PARTITION p_2012 VALUES LESS THAN (2012),
        PARTITION p_catchall VALUES LESS THAN MAXVALUE );
```
2. 在下面的场景中，分区可以起到非常大的作用：
   - 表非常大以至于无法全部都放在内存中，或者只在表的最后部分有热点数据，其他均是历史数据。
   - 分区表的数据更容易维护。例如，想批量删除大量数据可以使用清除整个分区的方式。另外，还可以对一个独立分区进行优化、检查、修复等操作。
   - 分区表的数据可以分布在不同的物理设备上，从而高效地利用多个硬件设备。
   - 可以使用分区表来避免某些特殊的瓶颈，例如InnoDB的单个索引的互斥访问、ext3文件系统的inode锁竞争等。
   - 如果需要，还可以备份和恢复独立的分区，这在非常大的数据集的场景下效果非常好
3. 分区表本身也有一些限制，下面是其中比较重要的几点：
   - 一个表最多只能有1024个分区。
   - 在MySQL 5.1中，分区表达式必须是整数，或者是返回整数的表达式。在MySQL 5.5中，某些场景中可以直接使用列来进行分区。
   - 如果分区字段中有主键或者唯一索引的列，那么所有主键列和唯一索引列都必须包含进来。
   - 分区表中无法使用外键约束。
   
   
4. 应该避免建立和分区列不匹配的索引，除非查询中还同时包含了可以过滤分区的条件。
5. 使用EXPLAIN PARTITION可以观察优化器是否执行了分区过滤
6. 视图查询（尽量不用视图）
    - 合并算法
    - 临时表算法：果视图中包含GROUY BY、DISTINCT、任何聚合函数、UNION、子查询等，只要无法在原表记录和视图记录中建立一一映射的场景中，MySQL都将使用临时表算法来实现视图
                select_type为“DERIVED”，说明该视图是采用临时表算法实现的。
                所有使用临时表算法实现的视图都无法被更新。
    - 因为MySQL在处理视图和处理子查询的代码路径完全不同，所以它们的性能也不同  
7. 外键约束：
    - update子表会去查询父表，开销大，
    - 外键约束使得查询需要额外访问一些别的表，这也意味着需要额外的锁。如果向子表中写入一条记录，外键约束会让InnoDB检查对应的父表的记录，也就需要对父表对应记录进行加锁操作，可能造成死锁
    - 如果只是使用外键做约束，那通常在应用程序里实现该约束会更好
8. 不要使用触发器
9. 不要使用存储过程
10. 定时任务事件
```sql
      CREATE EVENT optimize_somedb ON SCHEDULE EVERY 1 WEEK
        DO
        CALL optimize_tables('somedb')

    CREATE EVENT optimize_somedb ON SCHEDULE EVERY 1 WEEK
    DO
    BEGIN
       DECLARE CONTINUE HANLDER FOR SQLEXCEPTION
          BEGIN END;
       IF GET_LOCK('somedb', 0) THEN
          DO CALL optimize_tables('somedb');
       END IF;
       DO RELEASE_LOCK('somedb');
    END
```
11. 用户自定义函数UDF
      - http://tangent.org/586/Memcached_Functions_for_MySQL.html
      - http://www.mysqludf.org
12. 统一Mysql字符集 建议UTF-8
13. 全文索引，Match Against 需要列上有FULLTEXT索引
14. 不要用分布式事务XA
15. 尽量不要使用数据库缓存，可以用redis来实现第三方缓存。“命中和写入”的比率，即Qcache_hits和Qcache_inserts的比值。根据经验来看，当这个比值大于3:1时通常查询缓存是有效的，不过这个比率最好能够达到10:1。
  - query_cache_type 是否打开查询缓存。可以设置成OFF、ON或DEMAND。DEMAND表示只有在查询语句中明确写明SQL_CACHE的语句才放入查询缓存
  - query_cache_size 查询缓存使用的总内存空间，单位是字节。这个值必须是1 024的整数倍，否则MySQL实际分配的数据会和你指定的略有不同。
  - query_cache_min_res_unit 在查询缓存中分配内存块时的最小单位
  - query_cache_limit MySQL能够缓存的最大查询结果。如果查询结果大于这个值，则不会被缓存。如果你事先知道有很多这样的情况发生，那么建议在查询语句中加入SQL_NO_CACHE来避免查询缓存带来的额外消耗  
    - query_cache_limit太小 Qcache_lowmem_prunes。如果这个值增加得很快
    - 通过检查状态值Qcache_free_memory来查看还有多少没有使用的内存。
  - query_cache_wlock_invalidate 如果某个数据表被其他的连接锁住，是否仍然从查询缓存中返回结果。这个参数默认是OFF，这可能在一定程序上会改变服务器的行为，因为这使得数据库可能返回其他线程锁住的数据。将参数设置成ON，则不会从缓存中读取这类数据，但是这可能会增加锁等待。对于绝大数应用来说无须注意这个细节，所以默认设置通常是没有问题的
  - Qcache_free_blocks反映了查询缓存中空闲块的多少，如果Qcache_free_blocks大小恰好达到Qcache_total_blocks/2，那么查询缓存就有严重的碎片问题。而如果你还有很多空闲块，而状态值Qcache_lowmem_prunes还不断地增加，则说明由于碎片导致了过早地在删除查询缓存结果
  - FLUSH QUERY CACHE 完成碎片整理，会访问所有的查询缓存，在这期间任何其他的连接都无法访问查询缓存，从而会导致服务器僵死一段时间，使用这个命令的时候需要特别小心这点
  - RESET QUERY CACHE 清空缓存     
  
## 第8章 优化服务器设置            

1. 动态设置变量 
```sql

    SET           sort_buffer_size  = <value>;
    SET GLOBAL    sort_buffer_size  = <value>;
    SET         @@sort_buffer_size := <value>;
    SET @@session.sort_buffer_size := <value>;
    SET  @@global.sort_buffer_size := <value>;
```                 
如果动态地设置变量，要注意MySQL关闭时可能丢失这些设置。如果想保持这些设置，还是需要修改配置文件。  