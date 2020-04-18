---
title: Mysql索引注意事项以及关键字优化
date: 2017-11-25 18:08:46
updated: 2017-11-29 11:10:33
categories: 
- 后端
tags:
- Mysql
---

# MySQL 索引使用的注意事项

1. 索引不会包含有NULL的列
只要列中包含有NULL值，都将不会被包含在索引中，复合索引中只要有一列含有NULL值，那么这一列对于此符合索引就是无效的。

2. 使用短索引
对串列进行索引，如果可以就应该指定一个前缀长度。例如，如果有一个char（255）的列，如果在前10个或20个字符内，多数值是唯一的，那么就不要对整个列进行索引。短索引不仅可以提高查询速度而且可以节省磁盘空间和I/O操作。

3. 索引列排序
mysql查询只使用一个索引，因此如果where子句中已经使用了索引的话，那么order by中的列是不会使用索引的。因此数据库默认排序可以符合要求的情况下不要使用排序操作，尽量不要包含多个列的排序，如果需要最好给这些列建复合索引。

4. like语句操作
一般情况下不鼓励使用like操作，如果非使用不可，注意正确的使用方式。like ‘%aaa%’不会使用索引，而like ‘aaa%’可以使用索引。

5. 不要在列上进行运算
6. 不使用NOT IN 、<>、！=操作，但<,<=，=，>,>=,BETWEEN,IN是可以用到索引的
7. 索引要建立在经常进行select操作的字段上。

>这是因为，如果这些列很少用到，那么有无索引并不能明显改变查询速度。相反，由于增加了索引，反而降低了系统的维护速度和增大了空间需求。

8. 索引要建立在值比较唯一的字段上。
9. 对于那些定义为text、image和bit数据类型的列不应该增加索引。因为这些列的数据量要么相当大，要么取值很少。
10. 在where和join中出现的列需要建立索引。
11. where的查询条件里有不等号(where column != …),mysql将无法使用索引。
12. 如果where字句的查询条件里使用了函数(如：where DAY(column)=…),mysql将无法使用索引。
13. 在join操作中(需要从多个数据表提取数据时)，mysql只有在主键和外键的数据类型相同时才能使用索引，否则及时建立了索引也不会使用。

# 关键字优化

## In关键字原理

```sql
SELECT * FROM `user`
WHERE id in (SELECT user_id FROM `order`)
```

* in()语句只会执行一次，它查出order表中的所有user_id字段并且缓存起来，之后，检查user表的id是否和order表中的user_id相当，如果相等则加入结果期，直到遍历完user的所有记录。
* in的查询过程类似于以下过程

```shell
$result = [];
$users = "SELECT * FROM `user`";
$orders = "SELECT user_id FROM `order`";
for($i = 0;$i < $users.length;$i++){
    for($j = 0;$j < $orders.length;$j++){
    // 此过程为内存操作，不涉及数据库查询。
        if($users[$i].id == $orders[$j].user_id){
            $result[] = $users[$i];
            break;
        }
    }
}
```

* 我想你已经看出来了，当order表数据很大的时候不适合用in，因为它最多会将order表数据全部遍历一次。
    
    如：user表有10000条记录,order表有1000000条记录,那么最多有可能遍历10000*1000000次,效率很差.
    
    再如：user表有10000条记录,order表有100条记录,那么最多有可能遍历10000*100次,遍历次数大大减少,效率大大提升.

## exists关键字原理

```sql
SELECT * FROM `user`
WHERE exists (SELECT * FROM `order` WHERE user.id = order.user_id)
```

* 在这里，exists语句会执行user.length次，它并不会去缓存exists的结果集，因为这个结果集并不重要，你只需要返回真假即可。
* exists的查询过程类似于以下过程

```shell
$result = [];
$users = "SELECT * FROM `user`";
for($i=0;$i<$users.length;$i++){
    if(exists($users[$i].id)){// 执行SELECT * FROM `order` WHERE user.id = order.user_id
        $result[] = $users[$i];
    }
}
```

* 你看到了吧，当order表比user表大很多的时候，使用exists是再恰当不过了，它没有那么多遍历操作,只需要再执行一次查询就行。

    如:user表有10000条记录,order表有1000000条记录,那么exists()会执行10000次去判断user表中的id是否与order表中的user_id相等.

    如:user表有10000条记录,order表有100000000条记录,那么exists()还是执行10000次,因为它只执行user.length次,可见B表数据越多,越适合exists()发挥效果.

    **但是**：user表有10000条记录,order表有100条记录,那么exists()还是执行10000次,还不如使用in()遍历10000*100次,因为in()是在内存里遍历,而exists()需要查询数据库,我们都知道查询数据库所消耗的性能更高,而内存比较很快.

    因此我们只需要记住口诀：“外层查询表小于子查询表，则用exists，外层查询表大于子查询表，则用in，如果外层和子查询表差不多，则爱用哪个用哪个。”

# 说说 SQL 优化之道

## 一些常见的SQL实践

1. 负向条件查询不能使用索引

>select from order where status!=0 and stauts!=1
>not in/not exists都不是好习惯

可以优化为in查询：

>select from order where status in(2,3)

2. 前导模糊查询不能使用索引

>select from order where desc like '%XX'

而非前导模糊查询则可以：

>select from order where desc like 'XX%'
3. 数据区分度不大的字段不宜使用索引
>select from user where sex=1

原因：性别只有男，女，每次过滤掉的数据很少，不宜使用索引。  
经验上，能过滤80%数据时就可以使用索引。对于订单状态，如果状态值很少，不宜使用索引，如果状态值很多，能够过滤大量数据，则应该建立索引。
4. 在属性上进行计算不能命中索引
>select from order where YEAR(date) < = '2017'

即使date上建立了索引，也会全表扫描，可优化为值计算：

>select from order where date < = CURDATE()

或者：

>select from order where date < = '2017-01-01'

## 并非周知的SQL实践

5. 如果业务大部分是单条查询，使用Hash索引性能更好，例如用户中心

>select from user where uid=?
select from user where login_name=?

原因：B-Tree索引的时间复杂度是O(log(n))；Hash索引的时间复杂度是O(1)
6. 允许为null的列，查询有潜在大坑
单列索引不存null值，复合索引不存全为null的值，如果列允许为null，可能会得到“不符合预期”的结果集

>select from user where name != 'shenjian'

如果name允许为null，索引不存储null值，结果集中不会包含这些记录。
所以，请使用not null约束以及默认值。
7. 复合索引最左前缀，并不是值SQL语句的where顺序要和复合索引一致
用户中心建立了(login_name, passwd)的复合索引

>select from user where login_name=? and passwd=?
select from user where passwd=? and login_name=?

都能够命中索引

>select from user where login_name=?

也能命中索引，满足复合索引最左前缀

>select from user where passwd=?

不能命中索引，不满足复合索引最左前缀
8. 使用ENUM而不是字符串
ENUM保存的是TINYINT，别在枚举中搞一些“中国”“北京”“技术部”这样的字符串，字符串空间又大，效率又低。

## 小众但有用的SQL实践

9. 如果明确知道只有一条结果返回，limit 1能够提高效率

>select from user where login_name=?

可以优化为：

>select from user where login_name=? limit 1

原因：你知道只有一条结果，但数据库并不知道，明确告诉它，让它主动停止游标移动
10.  把计算放到业务层而不是数据库层，除了节省数据的CPU，还有意想不到的查询缓存优化效果
>select from order where date < = CURDATE()

这不是一个好的SQL实践，应该优化为：

>$curDate = date('Y-m-d');
$res = mysqlquery(
'select from order where date < = $curDate');

原因：
释放了数据库的CPU
多次调用，传入的SQL相同，才可以利用查询缓存
11. 强制类型转换会全表扫描
>select from user where phone=13800001234

你以为会命中phone索引么？大错特错了，这个语句究竟要怎么改？
末了，再加一条，不要使用select *（潜台词，文章的SQL都不合格 ==），只返回需要的列，能够大大的节省数据传输量，与数据库的内存使用量哟。

整理自：https://cloud.tencent.com/developer/article/1054203

## limit 20000 加载很慢怎么解决

mysql的性能低是因为数据库要去扫描N+M条记录，然后又要放弃之前N条记录，开销很大
解决思略：
1. 前端加缓存，或者其他方式，减少落到库的查询操作，例如某些系统中数据在搜索引擎中有备份的，可以用es等进行搜索
2. 使用延迟关联，即先通用limit得到需要数据的索引字段，然后再通过原表和索引字段关联获得需要数据
>select a.* from a,(select id from table_1 where is_deleted='N' limit 100000,20) b where a.id = b.id
3. 从业务上实现，不分页如此多，例如只能分页前100页，后面的不允许再查了
4. 不使用limit N,M,而是使用limit N，即将offset转化为where条件。