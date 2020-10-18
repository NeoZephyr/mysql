## mysql
### 安装
```sh
# 检查是否安装
sudo rpm -qa | grep mysql-server
sudo yum -y install mysql-server

# centos7 将 MYSQL 数据库从默认的程序列表中移除
sudo yum -y install mariadb-server mariadb

# 安全初始化
mysql_secure_installation
```

### 配置
```sh
# 字符集配置，在 mysqld 节点下添加
sudo vim /etc/my.cnf
# 5.1
default-character-set=utf8
# 5.5
character-set-server=utf8

# 配置 mysql 自启动
sudo chkconfig mysqld on
# 查看 2-5 为 on
sudo chkconfig --list mysqld

# 启动 mysql
sudo service mysqld start
sudo service mysqld restart

# mariadb
systemctl start mariadb
systemctl stop mariadb
systemctl enable mariadb
```

### 用户管理
#### root 用户
```sh
# root 登录
mysql -u root
select User,Host,Password from mysql.user;

# 为 root 用户重新设置密码
set password for root@localhost = password('root');
set password for root@127.0.0.1 = password('root');
set password for root@<machine name> = password('root');

# 刷新
flush privileges;
```
```sh
# 使用 udpate 更新密码
use mysql;
update user set password=password('root') WHERE User='root';
```

#### 创建用户
```sh
# 删除匿名用户
delete from mysql.user where user='';

# 添加用户
insert into mysql.user(Host,User,Password) values ("localhost", "pain", password("mall"));

create user pain identified by 'mall';
```

#### 用户授权
```sh
CREATE DATABASE `mall` DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;

grant all privileges on mall.* to mall@localhost identified by 'mall';

# 开通外网所有权限
grant all privileges on mall.* to mall@'%' identified by 'mall';

# 开通部分权限
grant select,insert,update on mall.* to mall@'192.11.11.11' identified by 'mall';

flush privileges;
```

```
# 导入 sql 文件
use mall;
source /developer/mall.sql;
```

### 防火墙
```sh
-A INPUT -p tcp -m tcp --dport 3306 -j ACCEPT
```