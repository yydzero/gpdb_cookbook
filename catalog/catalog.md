# catalog

## 修复系统表

有的时候需要进入 utility 模式，以便修复损坏的系统表。

```
$GPHOME/bin/postgres --single -P -O -p 5432 -D $MASTER_DATA_DIRECTORY -c gp_session_role=utility gpadmin

  -P: set ignore_system_indexes as true

  -O: set allow_system_table_mods as ALL
```

然后 psql 设置相应的 GUC：`allow_system_table_mods` 和 `ignore_system_indexes`

## check database age

```SELECT datname, age(datfrozenxid) FROM pg_database```
