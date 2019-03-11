## 1 通过pg_ctl关闭数据库
```
/usr/local/pgsql/bin/pg_ctl -D /pgdata/10/data stop
```
## 2 修改pg_hba.conf
```
vi /pgdata/10/data/pg_hba.conf
host    all             all             0.0.0.0/0               trust
```
## 3 通过pg_ctl启动数据库
## 4 修改密码