![CLEVER DATA GIT REPO](https://raw.githubusercontent.com/LiCongMingDeShujuku/git-resources/master/0-clever-data-github.png "李聪明的数据库")

# 指定在某月某天运行作业
#### Run SQL Job On Certain Day Of The Month
**发布-日期: 2016年09月23日 (评论)**



## Contents

- [中文](#中文)
- [English](#English)
- [SQL Logic](#Logic)
- [Build Info](#Build-Info)
- [Author](#Author)
- [License](#License) 


## 中文
最近有人问我如何设置让作业指定在某一天运行。在下面的例子中，它是该月的第3天。有一个简单的解决方案可以做到这一点。但是前提是这意味着工作必须每天运行，但这并不是什么大不了的事。

你只需创建第1步即可立即查看。如果今天不是本月的第3天，那么停止当前的作业。如果是当天，那么什么都不做，作业将进入下一步。


## English
Someone recently asked me how to setup a job to run on a certain day. In this case it’s the 3rd day of the month. There is one simple solution which can do that. Unfortunately; it means a job has to run every day, but that’s not a big deal really.

You just have to create Step 1 to check on today. If today is NOT the 3rd day of the month then STOP the current job. If it is the current day, then do nothing and the job will proceed to the next step.


---
## Logic
```SQL
use master;
set nocount on
 
declare @today datetime = (select dateadd(d, -0, datediff(d, 0, getdate())))
declare @3rd   datetime = (select dateadd(day, 2, dateadd(month, datediff(month, 0, getdate()), 0)))
 
if @today <> @3rd
    begin
        print 'today is NOT the 3rd day'
        exec msdb..sp_stop_job 'my job name'
    end
        else
            print 'today is the 3rd'


```
你可能会想， “这会不会引发错误？或者在作业上发出警报？”答案是并没有，基本上发生的是作业都没有“失败”。它只是“取消”。每当你右键单击某个作业并选择“停止作业”时，这基本上是相同的。因此，虽然代理历史记录会在运行历史旁边显示一个令人讨厌的红色图标，这并不是“失败”。它仍然只意味着它已被停止（取消），你可以通过作业历史在下面看到它。


You might be thinking… “Doesn’t this throw an error, or alert on the job?” Nope :) Basically what happens is the Job does not ‘fail’. It’s simply ‘cancelled’. This is basically the same thing whenever you right-click a job and select ‘Stop Job’. So although the agent history will show a nasty red icon next to the run history; it’s not a ‘failure’. It still only means it was stopped (cancelled) as you can see below by the Job history.

If you’re interested; here’s the quick test I created.
如果你有兴趣，这是我创建的快速测试。
I created a 2 step Job.
Step 1: Check to see if it’s the 3rd day. If not; Stops the Job. If so… do nothing and continue to the next step as usual.
Step 2: Backup the Model database.
Here’s the results of the Job after it’s cancelled from within the Step 1.

我创建了一个2步的作业。 
第1步：检查是否是第3天。如果不是，停止工作。如果是，什么也不做，像往常一样继续下一步。 
第2步：备份Model数据库。 这是作业从步骤1中取消后的结果。

![#](images/Run-SQL-Job-On-Certain-Day-Of-The-Month-a.png?raw=true "#")

I ran the Job a second time where @today = @today just to confirm the success, and it worked no problem.
我第二次运行作业 @today = @today只是为了确认是成功的，它没有问题。

Now; if you’re looking for a business day; you can try this:
现在，如果你想找一个工作日，你可以试试这个：


```SQL
use master;
set nocount on
 
declare @year_start        datetime
declare @year_finish       datetime
declare @today             datetime = (select dateadd(d, -0, datediff(d, 0, getdate())))
declare @daydiff           int
declare @holidays          table (Holiday datetime)
declare @3rd_biz_day       table
       (
              [Date]               datetime
       ,      [MonthName]          varchar (50)
       ,      [DateName]           varchar (50)
       ,      [BusinessDay] int
       )
set           @year_start   ='01 jan 2016'
set           @year_finish  ='31 dec 2016'
set           @daydiff      = datediff (day,@year_start,@year_finish)+1
 
insert into @holidays                    -- US Holidays
       select '1/1/2016'    union all     -- New Year's Day
       select '1/18/2016'   union all     -- Martin Luther King, Jr. Day
       select '2/15/2016'   union all     -- George Washington’s Birthday
       select '3/30/2016'   union all     -- Memorial Day
       select '6/4/2016'    union all     -- Independence Day
       select '9/5/2016'    union all     -- Labor Day
       select '10/10/2016'  union all     -- Columbus Day
       select '11/11/2016'  union all     -- Veterans Day
       select '11/24/2016'  union all     -- Thanksgiving Day
       select '12/26/2016'               -- Christmas Day
 
insert into @3rd_biz_day
select * from 
       (
              select 
                     'Date'        = dateadd(day,daynum,@year_start)
              ,      'MonthName'   = datename(month,(dateadd(day,daynum,@year_start)))
              ,      'DateName'    = datename(weekday,(dateadd(day,daynum,@year_start)))
              ,      'BusinessDay' = 
                     row_number() over (partition by (datepart(month,(dateadd(day,daynum,@year_start)))) 
                     order by (dateadd(day,daynum,@year_start)))
                     from (select top (@daydiff) row_number() over(order by (select 1))-1 as daynum 
              from 
                     sys.syscolumns sc_a cross join sys.syscolumns sc_b) dates
              where
                     datename(weekday,(dateadd(day,daynum,@year_start))) not in ('saturday','sunday') 
                     and dateadd(day,daynum,@year_start) not in (select holiday from @holidays)
       )      sc_a
where BusinessDay = 3
 
if     @today not in(select [date] from @3rd_biz_day)
       begin
              print 'today is NOT the 3rd business day'
              exec  msdb..sp_stop_job 'my job name'
       end
       else
              print 'today is the 3rd business day - continue to next job step'


```


[![WorksEveryTime](https://forthebadge.com/images/badges/60-percent-of-the-time-works-every-time.svg)](https://shitday.de/)

## Build-Info

| Build Quality | Build History |
|--|--|
|<table><tr><td>[![Build-Status](https://ci.appveyor.com/api/projects/status/pjxh5g91jpbh7t84?svg?style=flat-square)](#)</td></tr><tr><td>[![Coverage](https://coveralls.io/repos/github/tygerbytes/ResourceFitness/badge.svg?style=flat-square)](#)</td></tr><tr><td>[![Nuget](https://img.shields.io/nuget/v/TW.Resfit.Core.svg?style=flat-square)](#)</td></tr></table>|<table><tr><td>[![Build history](https://buildstats.info/appveyor/chart/tygerbytes/resourcefitness)](#)</td></tr></table>|

## Author

- **李聪明的数据库 Lee's Clever Data**
- **Mike的数据库宝典 Mikes Database Collection**
- **李聪明的数据库** "Lee Songming"

[![Gist](https://img.shields.io/badge/Gist-李聪明的数据库-<COLOR>.svg)](https://gist.github.com/congmingshuju)
[![Twitter](https://img.shields.io/badge/Twitter-mike的数据库宝典-<COLOR>.svg)](https://twitter.com/mikesdatawork?lang=en)
[![Wordpress](https://img.shields.io/badge/Wordpress-mike的数据库宝典-<COLOR>.svg)](https://mikesdatawork.wordpress.com/)

---
## License
[![LicenseCCSA](https://img.shields.io/badge/License-CreativeCommonsSA-<COLOR>.svg)](https://creativecommons.org/share-your-work/licensing-types-examples/)

![Lee Songming](https://raw.githubusercontent.com/LiCongMingDeShujuku/git-resources/master/1-clever-data-github.png "李聪明的数据库")

