1.[查询锁](https://stackoverflow.com/questions/11034504/show-all-current-locks-from-get-lock/42356351#42356351)
- 开启锁日志
    ```sql
    UPDATE performance_schema.setup_instruments
    SET enabled = 'YES'
    WHERE name = 'wait/lock/metadata/sql/mdl';
    ``` 
- 查出exclusive锁或者pending
    ```sql
    SELECT * FROM performance_schema.metadata_locks;
    ```
- 查询process id
    ```sql
     SELECT PROCESSLIST_ID FROM performance_schema.threads  WHERE THREAD_ID=35;
    ```
- 杀死排它锁
    ```sql
    KILL 10;
    ```
  
2. 按时间排序正在处理的sql
```sql
Select id,user,host,db,command,time,state from information_schema.processlist order by time desc;

select * from information_schema.processlist where info like '%RMC_UPLOAD_RPM%' order by time desc;
```
3. 杀死正在处理的线程
```
SELECT group_concat(concat('KILL ',id,';') SEPARATOR ' \n') AS KILL_EVERYTHING FROM information_schema.processlist where user ='rmc' and info like '%RMC_UPLOAD_RPM%'
```