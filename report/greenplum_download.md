

    create table downloads (email text, firstname text, lastname text, org text, country text, product text, category text, file text, version text, type text, release text, download_date date, method text, agent text);

    copy downloads from '/Users/yydzero/Downloads/pivotal-greenplum-downloads-external.csv' (format 'csv', header true, null '', quote '"');

    select country, extract(year from download_date) as year, count(*) from downloads where country = 'US' group by 1, 2 order by year;

    select substring(email from '@(.*)$') as suffix, count(1) from cn_us 
    where country = 'US' group by 1 order by count desc;


    create table cn_us as select email, firstname, lastname, org, country, file, version, download_date from downloads where country = 'US' or country = 'CN';

    create table cn as select email, firstname, lastname, org, country, file,
    version, download_date from downloads where country = 'CN' or email like '%.cn'
    or substring(email from '@(.*)$') IN ('21cn.com', 'qq.com', '163.com', '126.com', 'sina.com', 'sohu.com', '139.com', 'neusoft.com', 'landasoft.com', 'chinasie.com');


    select extract(year from download_date) as year, count(*) from cn group by 1 order by year;

    create table cn_distinct as select distinct * from cn;


    copy (   select extract(year from download_date) as year, count(*) from cn_distinct group by 1 order by year) to '/tmp/distinct.csv' (format 'csv');


    copy (    select extract(year from download_date) as year, count(*) from cn group by 1 order by year) to '/tmp/cn.csv' (format 'csv');
