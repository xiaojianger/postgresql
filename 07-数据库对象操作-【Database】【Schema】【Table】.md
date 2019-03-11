### 1 数据库基本操作
#### 1.1 特性
- 一个PostgreSQL数据库服务（实例）可以包含多个数据库（Database），当连接一个数据库时只能访问当前数据库的数据，无法访问其他数据库中的数据（除非使用dblink）
- pg一个实例可以包含多个数据库，一个数据库不能属于多个实例；oracle却相反，一个实例只能有一个数据库，一个数据库却可以在多个实例中（RAC）
#### 1.2 创建数据库
```
postgres=# \h create database
Command:     CREATE DATABASE
Description: create a new database
Syntax:
CREATE DATABASE name
    [ [ WITH ] [ OWNER [=] user_name ]
           [ TEMPLATE [=] template ]
           [ ENCODING [=] encoding ]
           [ LC_COLLATE [=] lc_collate ]
           [ LC_CTYPE [=] lc_ctype ]
           [ TABLESPACE [=] tablespace_name ]
           [ ALLOW_CONNECTIONS [=] allowconn ]
           [ CONNECTION LIMIT [=] connlimit ]
           [ IS_TEMPLATE [=] istemplate ] ]
```
- OWNER：指定新建的数据库属于哪个用户。如果不指定，新建的数据库就属于当前执行命令的用户
- TEMPLATE：模板名（从哪个模板创建新数据库）。如果不指定，将使用默认模板数据库（template1）：
```
postgres=# \l template1
                             List of databases
   Name    |  Owner   | Encoding | Collate | Ctype |   Access privileges   
-----------+----------+----------+---------+-------+-----------------------
 template1 | postgres | UTF8     | C       | C     | =c/postgres          +
           |          |          |         |       | postgres=CTc/postgres
(1 row)
```
- ENCODING：创建新数据库使用的字符编码。指定的字符编码必须和模板中的字符编码一致，默认模板是template1。指定非UTF8字符时需要指定模板为template0，template0不包含任何字符信息（UTF8字符集来支持中文，不支持常用的GBK、GBK1830）
```
postgres=# CREATE DATABASE testdb1 TEMPLATE template0;
CREATE DATABASE
```
- TABLESPACE：指定和新数据库关联的表空间名称。
- CONNECTION LIMIT：数据库可以接受多少并发的连接。默认值-1，表示没有限制。
- IS_TEMPLATE：指定新建数据库是模板数据库。

#### 1.2 修改数据库
```
postgres=# \h alter database;
Command:     ALTER DATABASE
Description: change a database
Syntax:
ALTER DATABASE name [ [ WITH ] option [ ... ] ]

where option can be:

    ALLOW_CONNECTIONS allowconn
    CONNECTION LIMIT connlimit
    IS_TEMPLATE istemplate

ALTER DATABASE name RENAME TO new_name

ALTER DATABASE name OWNER TO { new_owner | CURRENT_USER | SESSION_USER }

ALTER DATABASE name SET TABLESPACE new_tablespace

ALTER DATABASE name SET configuration_parameter { TO | = } { value | DEFAULT }
ALTER DATABASE name SET configuration_parameter FROM CURRENT
ALTER DATABASE name RESET configuration_parameter
ALTER DATABASE name RESET ALL
```
#### 1.3 删除数据库
```
postgres=# \h drop database;
Command:     DROP DATABASE
Description: remove a database
Syntax:
DROP DATABASE [ IF EXISTS ] name
```
- 如果一个数据库存在，则将其删除，如果不存在，使用删除命令时也不会报错。
- 如果还有人连接在这个数据库上，将不能删除该数据库
```
postgres=# drop database testdb1;
ERROR:  database "testdb1" is being accessed by other users
DETAIL:  There are 2 other sessions using the database.
```

### 2 Schema基本操作
#### 2.1 Schema定义
- 一个Database可以包含多个Schema，不同的Schema之间可以互相访问
- 允许多个用户在使用同一个数据库时彼此互不干扰
- 把数据库对象放在不同的模式下，然后组织成逻辑组，让它们更便于管理
#### 2.2 创建Schema
```
postgres=# \h create schema
Command:     CREATE SCHEMA
Description: define a new schema
Syntax:
CREATE SCHEMA schema_name [ AUTHORIZATION role_specification ] [ schema_element [ ... ] ]
CREATE SCHEMA AUTHORIZATION role_specification [ schema_element [ ... ] ]
CREATE SCHEMA IF NOT EXISTS schema_name [ AUTHORIZATION role_specification ]
CREATE SCHEMA IF NOT EXISTS AUTHORIZATION role_specification

where role_specification can be:

    user_name
  | CURRENT_USER
  | SESSION_USER
```
- AUTHORIZATION role_specification：可以指定给用户创建schema，如果不指定就属于当前执行命令的用户
```
postgres=# create schema testschema1;
CREATE SCHEMA
postgres=# create schema testschema2 AUTHORIZATION jiangzq;
CREATE SCHEMA
postgres=# \dn
    List of schemas
    Name     |  Owner   
-------------+----------
 osdba       | postgres
 public      | postgres
 testschema1 | postgres
 testschema2 | jiangzq
(4 rows)
```
- schema_element：创建schema的同时可以创建一些表或者视图对象
```
postgres=# create schema testschema3 create table t1 (id int,name text) create table t2 (id int,name text);
CREATE SCHEMA
```
#### 2.3 修改和删除
```
##修改
postgres=# \h alter schema;
Command:     ALTER SCHEMA
Description: change the definition of a schema
Syntax:
ALTER SCHEMA name RENAME TO new_name
ALTER SCHEMA name OWNER TO { new_owner | CURRENT_USER | SESSION_USER }

##删除
postgres=# \h drop schema;
Command:     DROP SCHEMA
Description: remove a schema
Syntax:
DROP SCHEMA [ IF EXISTS ] name [, ...] [ CASCADE | RESTRICT ]
```
#### 2.4 Schema的搜索路径
- 登录到数据库后，如果没有指定，默认情况下都是在“public”模式下
- 查看当前搜索路径
```
postgres=# show search_path;
   search_path   
-----------------
 "$user", public
```
- 设置search_path
```
##设置当前会话的路径
postgres=# select * from testschema3.t1;
postgres=# set search_path to testschema3;
SET
postgres=# show search_path;
 search_path 
-------------
 testschema3
(1 row)
##长久设置路径
postgres=# ALTER ROLE jiangzq SET search_path=testschema3;
ALTER ROLE
##官方建议是这样的：在管理员创建一个具体数据库后，应该为所有可以连接到该数据库的用户分别创建一个与用户名相同的模式，然后，将search_path设置为"$user"，即默认的模式是与用户名相同的模式。
```
#### 2.5 Schema的权限
- USAGE：Schema访问权限
- CREATE：在Schema中创建对象的权限
- 默认情况下，每个用户含有“public”Schema的USAGE和CREATE权限，如果需要取消这种默认值，如下：
```
revoke create on schema public from public;
```
- 在PostgreSQL中为每个用户都创建了一个与用户名同名的模式，那么就能与Oracle数据库相兼容了。

### 3 表操作
#### 3.1 创建数据表
```
postgres=# \h create table;
Command:     CREATE TABLE
Description: define a new table
Syntax:
CREATE [ [ GLOBAL | LOCAL ] { TEMPORARY | TEMP } | UNLOGGED ] TABLE [ IF NOT EXISTS ] table_name ( [
  { column_name data_type [ COLLATE collation ] [ column_constraint [ ... ] ]
    | table_constraint
    | LIKE source_table [ like_option ... ] }
    [, ... ]
] )
[ INHERITS ( parent_table [, ... ] ) ]
[ PARTITION BY { RANGE | LIST } ( { column_name | ( expression ) } [ COLLATE collation ] [ opclass ] [, ... ] ) ]
[ WITH ( storage_parameter [= value] [, ... ] ) | WITH OIDS | WITHOUT OIDS ]
[ ON COMMIT { PRESERVE ROWS | DELETE ROWS | DROP } ]
[ TABLESPACE tablespace_name ]
```
- column_name data_type：列名称和数据类型
- column_constraint|table_constraint：列和表约束（主键、索引）
- INHERITS：继承表的信息
- WITH OIDS：系统列
- 默认值
```
CREATE TABLE products (
product_no integer,
name text,
price numeric DEFAULT 9.99
);
##使用序列号
CREATE TABLE products (
product_no integer DEFAULT nextval('products_product_no_seq'),
...
);
```
- 约束
```
##检查约束
CREATE TABLE products (
product_no integer,
name text,
price numeric CHECK (price > 0),
#给约束命名为positive_price
price numeric CONSTRAINT positive_price CHECK (price > 0),
discounted_price numeric CHECK (discounted_price > 0),
CHECK (price > discounted_price),
);

##非空约束
CREATE TABLE products (
product_no integer NOT NULL,
name text NOT NULL,
price numeric
);

##唯一约束
CREATE TABLE products (
product_no integer UNIQUE,
name text,
price numeric,
#给约束命名为must_be_different
test_no CONSTRAINT must_be_different UNIQUE,
UNIQUE (name,price)
);

#主键
CREATE TABLE products (
product_no integer PRIMARY KEY,
name text,
price numeric
);
CREATE TABLE example (
a integer,
b integer,
c integer,
PRIMARY KEY (a, c)
);

#外键
CREATE TABLE orders (
order_id integer PRIMARY KEY,
product_no integer REFERENCES products (product_no),
quantity integer
);
CREATE TABLE orders (
order_id integer PRIMARY KEY,
product_no integer REFERENCES products,
quantity integer
);
#因为如果缺少列的列表，则被引用表的主键将被用作被引用列。
```
#### 3.3 表的存储属性
- PostgreSQL的页面大小是固定的（通常是8k），并且不允许行跨越多个页面。因此不能执行存储非常大的字段值
- 大字段被压缩或者切片成多个物理行存到另外一张系统表中（TOAST表）
- 支持TOAST的数据类型必须是可变长，在变长的类型中，前4个字节称为长度字，长度字后面存储具体的内容或指针
- 如果一个表中任何一个字段是可以TOAST的，那么PostgreSQL会自动为该表建一个相关联的TOAST表 ，其OID存在pg_class.reltoastrelid

#### 3.4 临时表
- 默认情况创建的是会话级的临时表
```
create temporary table tmp_t1(id int primary key,note text);
```
- 创建事务级的临时表
```
create temporary table tmp_t2(id int primary key,note text) on commit delete rows;
```

#### 3.5 修改表
- 增加列
```
ALTER TABLE products ADD COLUMN description text;
```
- 删除列
```
ALTER TABLE products DROP COLUMN description;
#如果有外键，需要加上CASCADE
ALTER TABLE products DROP COLUMN description CASCADE;
```
- 增加约束
```
ALTER TABLE products ADD CHECK (name <> '');
ALTER TABLE products ADD CONSTRAINT some_name UNIQUE (product_no);
ALTER TABLE products ADD FOREIGN KEY (product_group_id) REFERENCES
product_groups
ALTER TABLE products ALTER COLUMN product_no SET NOT NULL;
```
- 移除约束
```
ALTER TABLE products DROP CONSTRAINT some_name;
ALTER TABLE products ALTER COLUMN product_no DROP NOT NULL;
```
- 更改列的默认值
```
ALTER TABLE products ALTER COLUMN price SET DEFAULT 7.77;
ALTER TABLE products ALTER COLUMN price DROP DEFAULT;
```
- 重命名
```
ALTER TABLE products RENAME COLUMN product_no TO product_number;
ALTER TABLE products RENAME TO items;
```
#### 3.6 表继承
```
CREATE TABLE cities (
name text,
population float,
altitude int -- in feet
);
CREATE TABLE capitals (
state char(2)
) INHERITS (cities);
```
- 查询父表时可以查到子表的数据，如果只查出父表加上only
```
select * from only cities;
```
- 查询子表不能查出父表数据
- 所有父表的检查约束和非空约束都会自动被所有子表继承，不过其他类型的约束（唯一、主键、外键）则不会继承
- 一个子表可以继承多个父表
- 采用select、update、delete等命令操作父表时，也会操作子表，修改父表结构时也会影响子表
- 唯一约束、外键的使用域也不会扩大到子表

#### 3.7 分区表
- 表的大小超过数据库服务器的物理内存大小则应该使用
- PostgreSQL9.6及之前是通过表继承来实现分区表
```
#一般会让父表为空，数据都存储在子表中：
#建表步骤：
#创建父表
CREATE TABLE measurement (
    city_id         int not null,
    logdate         date not null,
    peaktemp        int,
    unitsales       int
);

#为每个月创建分区表，提供不重叠的表约束
CREATE TABLE measurement_y2006m02 (
    CHECK ( logdate >= DATE '2006-02-01' AND logdate < DATE '2006-03-01' )
) INHERITS (measurement);
CREATE TABLE measurement_y2006m03 (
    CHECK ( logdate >= DATE '2006-03-01' AND logdate < DATE '2006-04-01' )
) INHERITS (measurement);
...
CREATE TABLE measurement_y2007m11 (
    CHECK ( logdate >= DATE '2007-11-01' AND logdate < DATE '2007-12-01' )
) INHERITS (measurement);
CREATE TABLE measurement_y2007m12 (
    CHECK ( logdate >= DATE '2007-12-01' AND logdate < DATE '2008-01-01' )
) INHERITS (measurement);
CREATE TABLE measurement_y2008m01 (
    CHECK ( logdate >= DATE '2008-01-01' AND logdate < DATE '2008-02-01' )
) INHERITS (measurement);

#通过触发器来将数据正确的插入到相应的分区表
CREATE OR REPLACE FUNCTION measurement_insert_trigger()
RETURNS TRIGGER AS $$
BEGIN
    IF ( NEW.logdate >= DATE '2006-02-01' AND
         NEW.logdate < DATE '2006-03-01' ) THEN
        INSERT INTO measurement_y2006m02 VALUES (NEW.*);
    ELSIF ( NEW.logdate >= DATE '2006-03-01' AND
            NEW.logdate < DATE '2006-04-01' ) THEN
        INSERT INTO measurement_y2006m03 VALUES (NEW.*);
    ...
    ELSIF ( NEW.logdate >= DATE '2008-01-01' AND
            NEW.logdate < DATE '2008-02-01' ) THEN
        INSERT INTO measurement_y2008m01 VALUES (NEW.*);
    ELSE
        RAISE EXCEPTION 'Date out of range.  Fix the measurement_insert_trigger() function!';
    END IF;
    RETURN NULL;
END;
$$
LANGUAGE plpgsql;

CREATE TRIGGER insert_measurement_trigger
    BEFORE INSERT ON measurement
    FOR EACH ROW EXECUTE PROCEDURE measurement_insert_trigger();
```
- 继承式分区表优化：需要将constraint_exclusion参数设置为on，默认值是partition，已经是on的状态
- PostgreSQL10.1及之后可以使用声明式分区
```
#创建分区表
CREATE TABLE measurement (
city_id int not null,
logdate date not null,
peaktemp int,
unitsales int
) PARTITION BY RANGE (logdate);

#添加分区
CREATE TABLE measurement_y2006m02 PARTITION OF measurement
FOR VALUES FROM ('2006-02-01') TO ('2006-03-01')
CREATE TABLE measurement_y2006m03 PARTITION OF measurement
FOR VALUES FROM ('2006-03-01') TO ('2006-04-01')
...
CREATE TABLE measurement_y2007m11 PARTITION OF measurement
FOR VALUES FROM ('2007-11-01') TO ('2007-12-01')
```