
* JobDetail      
  实现job接口  
* Scheduler（调度器）  
  scheduler主要由StdScheduler类、Scheduler接口、SchedulerFactory接口、StdSchedulerFactory类、QuartzScheduler类实现
* Trigger    
  cron：Seconds Minutes Hours DayofMonth Month DayofWeek Year  
  秒（0~59） 
  分钟（0~59） 
  小时（0~23） 
  天（月）（0~31，但是你需要考虑你月的天数） 
  月（0~11） 
  天（星期）（1~7 1=SUN 或 SUN，MON，TUE，WED，THU，FRI，SAT） 
  年份（1970－2099）

关于name和group

JobDetail 和 Trigger 都有 name 和 group。

name 是它们在这个sheduler里面的唯一标识。如果我们要更新一个JobDetail定义，只需要设置一个name相同的JobDetail实例即可。

group 是一个组织单元，sheduler会提供一些对整组操作的 API，比如 scheduler.resumeJobs()。

### trigger触发原理
qsRsrcs.getJobStore().acquireNextTriggers【查找即将触发的Trigger】 ----> 
sigLock.wait(timeUntilTrigger)【等待执行】 ----> 
qsRsrcs.getJobStore().triggersFired(triggers)【执行】----> 

qsRsrcs.getJobStore().releaseAcquiredTrigger(triggers.get(i)) 【释放Trigger】



## Linux
crontab任务配置基本格式

command
分钟(0-59)　小时(0-23)　日期(1-31)　月份(1-12)　星期(0-6,0代表星期天)　 命令
第1列表示分钟1～59 每分钟用*或者 */1表示
第2列表示小时1～23（0表示0点）
第3列表示日期1～31
第4列表示月份1～12
第5列标识号星期0～6（0表示星期天）
第6列要运行的命令

-----------------------------

调度器
job 、job detail
触发器（SimpleTrigger、CronTrigger）

cron 表达式
Seconds (秒)         ：可以用数字0－59 表示，
Minutes(分)          ：可以用数字0－59 表示，
Hours(时)             ：可以用数字0-23表示,
Day-of-Month(天) ：可以用数字1-31 中的任一一个值，但要注意一些特别的月份
Month(月)            ：可以用0-11 或用字符串  “JAN, FEB, MAR, APR, MAY, JUN, JUL, AUG, SEP, OCT, NOV and DEC” 表示
Day-of-Week(每周)：可以用数字1-7表示（1 ＝星期日）或用字符口串“SUN, MON, TUE, WED, THU, FRI and SAT”表示
“/”：为特别单位，表示为“每”如“0/15”表示每隔15分钟执行一次,“0”表示为从“0”分开始, “3/20”表示表示每隔20分钟执行一次，“3”表示从第3分钟开始执行
“?”：表示每月的某一天，或第周的某一天
“L”：用于每月，或每周，表示为每月的最后一天，或每个月的最后星期几如“6L”表示“每月的最后一个星期五”
“W”：表示为最近工作日，如“15W”放在每月（day-of-month）字段上表示为“到本月15日最近的工作日”
““#”：是用来指定“的”每月第n个工作日,例在每周（day-of-week）这个字段中内容为"6#3" or "FRI#3" 则表示“每月第三个星期五”

每隔5秒执行一次：*/5 * * * * ?
每隔1分钟执行一次：0 */1 * * * ?
每天23点执行一次：0 0 23 * * ?
每天凌晨1点执行一次：0 0 1 * * ?
每月1号凌晨1点执行一次：0 0 1 1 * ?
每月最后一天23点执行一次：0 0 23 L * ?
 每周星期天凌晨1点实行一次：0 0 1 ? * L
 在26分、29分、33分执行一次：0 26,29,33 * * * ?
 每天的0点、13点、18点、21点都执行一次：0 0 0,13,18,21 * * ?