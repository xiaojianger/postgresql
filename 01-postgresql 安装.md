
## 1.1 RPM安装
### Redhat
```
yum install postgresql-server.x86_64
```
### Debian/ubuntu
```
sudo apt-get install postgresql
```
### 新版本安装
```
yum install https://download.postgresql.org/pub/repos/yum/10/redhat/rhel-6-x86_64/pgdg-centos10-10-2.noarch.rpm
##server端
yum install postgresql10-server
##client端
yum install postgresql10
```
## 1.2 源代码安装（推荐方式）
```
tar -zxf /opt/postgresql-10.1.tar.gz
mv /opt/postgresql-10.1 /usr/local/postgresql-10.1
ln -s /usr/local/postgresql-10.1 /usr/local/pgsql
cd /usr/local/pgsql
./configure --prefix=/usr/local/pgsql
make
make install
adduser postgres
mkdir -p /data/pgsql/data
chown -R postgres /data/pgsql/ /usr/local/pgsql
su - postgres
/usr/local/pgsql/bin/initdb -D /data/pgsql/data
/usr/local/pgsql/bin/postgres -D /data/pgsql/data >logfile 2>&1 &
/usr/local/pgsql/bin/createdb test
/usr/local/pgsql/bin/psql test
```
## 1.3 二进制安装
官方不提供（不推荐使用），需要去第三方下载二进制包
https://www.enterprisedb.com/download-postgresql-binaries
```
#下载
wget https://get.enterprisedb.com/postgresql/postgresql-10.1-1-linux-x64-binaries.tar.gz 

tar -zxvf  postgresql-10.1-1-linux-x64-binaries.tar.gz
ln -s /opt/pgsql /usr/local/pgsql
groupadd postgres
useradd -g postgres postgres
mkdir -p /data/pgsql/{data,log}
chown -R postgres.postgres /data/pgsql
su - postgres
./initdb -E utf8 -D /data/pgsql/data
exit
echo 'export PATH=/usr/local/pgsql/bin:$PATH' >> /etc/profile && source /etc/profile
su - postgres
#启动数据库
pg_ctl -D /data/pgsql/data -l /data/pgsql/log/postgres.log start
或
./postgres -D /opt/pgsql/data/ > /opt/pgsql/log/postgres.log &
#登陆数据库
psql
```