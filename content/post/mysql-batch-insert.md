---
title: "MySQL使用存储过程批量插入数据" 

date: 2021-02-05T16:00:00+08:00 

categories: ["MySQL"]

tags: ["MySQL", "数据库"] 

author: "Me" 

showToc: true 

TocOpen: false 

draft: false 

hidemeta: false 

comments: false 

description: "" 

disableHLJS: true  

disableShare: false 

cover:
  image: "https://raw.githubusercontent.com/jiayanju/imgrepo/main/mysql.png"
  alt: ""
  caption: ""
  relative: false
  hidden: false


---

最近有个需求需要实现LBS服务，基于用户当前的位置信息，返回用户周边的一些位置。这个需求现在使用很多方法都可以实现，比如基于redis，elasticsearch，mongodb。由于目前系统的用户数比较少，初步预估数据量不会很大，也不想因此引入额外的依赖增加系统复杂度。所以打算使用MySQL实现。因此想要批量的插入一批数据，测试一下使用MySQL查找位置信息的性能。

### 1. 创建测试表

```sql
CREATE TABLE IF NOT EXISTS `test`.`locations_mall` (
    `id` bigint NOT NULL PRIMARY KEY AUTO_INCREMENT,
    `name` varchar(100),
    `position` POINT NOT NULL SRID 4326,
    SPATIAL INDEX (`position`)
);
```

### 2. 测试脚本1 - 单条插入批量commit

```sql
DROP PROCEDURE IF EXISTS proc_batch_insert;
delimiter //
CREATE PROCEDURE proc_batch_insert()
begin
    declare i int default 0 ;
    dd:loop
        set i = i+1 ;
        INSERT INTO `test`.locations_mall(name, position)
        values
        (CONCAT('point_test', i), ST_GeomFromText(concat('POINT(',(-90 + RAND()*(90 - (-90))), ' ', (-180 + RAND() * (180 - (-180))),')'), 4326));
        if i % 1000 = 0 then
            commit;
        end if;
        if  i >= 100000 then leave dd;
        end if;
    end loop dd;
end;
//
delimiter ;

call proc_batch_insert();
```

本来想着批量1000条，然后commit一次。但是这种方法不行，和单条一个一个插入效果差不条，commit并没有起作用。

插入十万条数据，十多分钟都没有跑完。

### 3. 测试脚本 - 单条插入语句批量值

```sql
drop procedure if exists batch_insert;
delimiter //
create procedure batch_insert()
begin
    declare i int;
    set i = 0;
    set @sqlstr='insert into `test`.locations_mall(name, position) values ';
    while i<=3000000 do
            set i=i+1;
            set @sqlstr=concat(@sqlstr,'(concat(''point-test-'',',i,'),', 'ST_GeomFromText(concat(''POINT('',(-90 + RAND()*(90 - (-90))), '' '', (-180 + RAND() * (180 - (-180))),'')''), 4326)', ')');
            if mod(i,10000)=0 then
                prepare stmt from @sqlstr;
                execute stmt;
                deallocate prepare stmt;
                set @sqlstr='insert into `test`.locations_mall(name, position) values ';
            else
                set @sqlstr=concat(@sqlstr,',');
            end if;
        end while;
end;
//
delimiter ;

CALL batch_insert();
```

批量的将要插入的数据值放入单条insert语句

十万条数据插入耗时 76s，还可以接受。

---  ---

PS：

网上查找资料的时候，有提到可以设置如下参数：

set global bulk_insert_buffer_size=104857600;
set session autocommit=off;
set session unique_checks=off; 

当使用以上优化之后，循环插入单条记录的办法也可以很大的提高性能，比第二种的还要高。