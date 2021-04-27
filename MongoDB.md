[Window平台开启MongoDB安全认证](https://docs.mongodb.com/manual/tutorial/enable-authentication/)

1. 使用批处理一键开启安全认证

mongo_start.bat
```
:: 指定数据库文件存储路径
set database_path=%1
if "%database_path%"=="" (
	set database_path=.\data
)

:: 数据库文件存储路径不存在则创建
if not exist %database_path% mkdir %database_path%

:: 1. 开启mongo服务器
start /min mongod.exe --dbpath=%database_path%

:: 2. 创建超级管理员用户
tools\mongodb_win32\mongo.exe mongo_creat_user.js

:: 3. 重启mongo服务器，开启安全认证
start /min tools\mongodb_win32\mongod.exe --dbpath=%database_path% --auth
```

mongo_create_user.js [参考官网文档](https://docs.mongoing.com/the-mongo-shell/write-scripts-for-the-mongo-shell)
```
// 连接数据库
db = connect("localhost:27017/admin");

// 创建可读写任意数据库的超级管理员用户
// db.createUser({user: "admin", pwd: "123456", roles: [ { role: "userAdminAnyDatabase", db: "admin" }, "readWriteAnyDatabase" ]})
db.createUser(
  {
    user: "admin",
    pwd: "123456",
    roles: [ { role: "userAdminAnyDatabase", db: "admin" }, "readWriteAnyDatabase" ]
  }
)

// mongo服务器需重启并开启安全认证，这里可以先关闭它
db.shutdownServer();
quit();
```

2. 使用超级管理员账号认证登陆
mongo 127.0.0.1:27017/admin -u admin -p 123456

3. 创建游戏数据库game_db

4. 为游戏数据库game_db创建一个只读权限的普通用户，游戏上线后作为数据分析工具用的账号。
```
use game_db

// db.createUser({user: "xiaojl", pwd: "123456", roles: [ { role: "read", db: "game_db" } ]})
db.createUser(
  {
    user: "xiaojl",
    pwd: "123456",
    roles: [ { role: "read", db: "game_db" } ]
  }
)
```

[Linux平台开启MongoDB安全认证](https://docs.mongodb.com/manual/tutorial/enable-authentication/)

同样是需要在未开启认证的时候创建一个超级管理员账号，然后开启认证参数并重启mongo服务器。以下为linux下操作mongo的常用命令，后续遇到新的再补充。
```
// 查看进程状态
$ ps -ax | grep mongod
/usr/bin/mongod -f /etc/mongod.conf

// 查看配置详情
$ cat /etc/mongod.conf
#security:

// 开启认证参数
$ sudo vim /etc/mongod.conf
security:
  authorization: enabled

// 保存退出编辑
Esc + : + wq!

// 重启MongoDB
sudo service mongod restart
```
