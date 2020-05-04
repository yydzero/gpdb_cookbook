# Trick plan 

## set plan stats to use specific plan

如何获得 HashRightJoin

      update pg_class set (relpages, reltuples) = (1000000, 1)
      where relname = 'tmp_r';
    
      update pg_class set (relpages, reltuples) = (1, 1000000)
      where relname = 'tmp_q';
    
    after which I get a plan like this:
    
       Hash Right Join
         Hash Cond: (...)
         ->  Seq Scan on tmp_q q
         ->  Hash
               ->  Seq Scan on tmp_r r

## 使用随机数据

    create table small (id bigint, val text);
    create table large (id bigint, val text);

    insert into large select 1000000000 * random(), md5(i::text)
    from generate_series(1, 700000000) s(i);

    insert into small select 1000000000 * random(), md5(i::text)
    from generate_series(1, 10000) s(i);

    vacuum analyze large;
    vacuum analyze small;

    update pg_class set (relpages, reltuples) = (1000000, 1)
    where relname = 'large';

    update pg_class set (relpages, reltuples) = (1, 1000000)
    where relname = 'small';

    set work_mem = '1MB';

    explain analyze select * from small join large using (id);

## composite index

    create table t1 (a int, b int);
    create table t2 (a int, b int);


    insert into t1 select mod(i,1000), mod(i,1000)
      from generate_series(1,100000) s(i);

    insert into t2 select mod(i,1000), mod(i,1000)
      from generate_series(1,100000) s(i);

    analyze t1;
    analyze t2;

    explain analyze select * from t1 join t2 on (t1.a = t2.a and t1.b = t2.b);

                                       QUERY PLAN
    --------------------------------------------------------------------
     Merge Join  (cost=19495.72..21095.56 rows=9999 width=16)
                 (actual time=183.043..10360.276 rows=10000000 loops=1)
       Merge Cond: ((t1.a = t2.a) AND (t1.b = t2.b))
       ...

    create type composite_id as (a int, b int);

    create index on t1 (((a,b)::composite_id));
    create index on t2 (((a,b)::composite_id));

    analyze t1;
    analyze t2;

    explain analyze select * from t1 join t2
      on ((t1.a,t1.b)::composite_id = (t2.a,t2.b)::composite_id);
                                        QUERY PLAN
    --------------------------------------------------------------------------
     Merge Join  (cost=0.83..161674.40 rows=9999460 width=16)
                 (actual time=0.020..12726.767 rows=10000000 loops=1)
       Merge Cond: (ROW(t1.a, t1.b)::composite_id = ROW(t2.a, t2.b)::composite_id)
