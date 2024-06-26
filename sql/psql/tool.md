
数据去重[[cte]]

``` sql
ALTER TABLE table  ADD COLUMN id SERIAL PRIMARY KEY;  -- 如果表中没有主键则添加一个主键

with cte as (
	select *,row_number() over (partition by 序号 order by id) as an from drg_relation dr
) delete from drg_relation
where id in (select id from cte where an > 1 );  -- 删除重复的列
```
