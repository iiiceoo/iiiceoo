# YApi 部署

>YApi 是一款高效、易用、功能强大的 API 管理软件，旨在为开发、产品、测试人员提供更优雅的接口管理服务。可以帮助开发者轻松创建、发布、维护 API。本文简要的记录了 YApi 的部署流程。



## Nodejs 安装

访问 nodejs 官网，下载最新的[安装包](https://nodejs.org/en/download/)。注意 nodejs 版本需 7.6+。

![image-20200808160253517](./img/image001.png)



上传包至服务器，解压安装。

```bash
tar -xvf node-v12.18.2-linux-x64.tar.xz
mv node-v12.18.2-linux-x64 /usr/local/nodejs
```



为 `node` 以及 `npm` 命令建立软连接。

```bash
ln -s /usr/local/nodejs/bin/node /usr/local/bin/node
ln -s /usr/local/nodejs/bin/npm /usr/local/bin/npm
```



执行 `node -v` 显示版本号即可。

![image-20200808151112665](./img/image002.png)



## Mongodb 安装

访问 mongodb 官网，下载最新[安装包](https://www.mongodb.com/try/download)。注意 mongodb 版本需 2.6+。

![image-20200808151250206](./img/image003.png)



![image-20200808151259095](./img/image004.png)



上传包至服务器，解压安装。

```bash
tar -xzvf mongodb-linux-x86_64-rhel70-4.2.8.tgz
mv mongodb-linux-x86_64-rhel70-4.2.8 /usr/local/mongodb
```



新增数据、日志、配置目录，修改 mongodb.config 配置文件，添加如下内容。

```bash
cd /usr/local/mongodb/
mkdir data log config
cd config/
vim mongodb.config
```

![image-20200808151610638](./img/image005.png)



以指定配置文件启动 `mongod`。

```bash
mongod -f /usr/local/mongodb/conf/mongodb.conf
```

![image-20200808151717736](./img/image006.png)



若需要停止 mongod 服务。

```bash
mongod -f /usr/local/mongodb/conf/mongodb.conf –-shutdown
```

或者在客户端中停止mongod 服务。

```mongodb
> use admin
> db.auth("account","password")
> db.shutdownServer()
```



连接 mongodb。

```bash
mongo
```

![image-20200808152338953](./img/image007.png)



查看数据库。

```mongodb
> show dbs
```

![image-20200808152448014](./img/image008.png)

这里并未看到 mongodb 初始的三个数据库，这是由于预先定义的配置文件 mongodb.conf 中已开启了**权限认证**，所以必须先为mongodb 添加相关用户，通过权限认证后即可看到相关数据库。



在 admin 库创建一个超级用户，角色为 `userAdminAnyDatabase`。

```mongodb
> use admin
> db.createUser({user: "admin", pwd: "123456", roles: [{ role: "userAdminAnyDatabase", db: "admin"}]})
```

![image-20200808153104790](./img/image009.png)



建立 yapi 库，作为后续 YApi 使用的数据库，同时为其也建立一个管理员。

```mongodb
> use yapi
> db.createUser({user: "yapi", pwd: "yapi", roles: [{role: "dbOwner", db: "yapi"}]})
```

![image-20200808153145158](./img/image010.png)

这里选择为 yapi 用户赋予 `dbOwner` 角色而并非普通的 `readWrite` 角色。原因是后续 YApi 部署时初始化表结构时存在相关**建立索引**的操作，普通的 `readWrite` 角色可能由于权限不足导致**无法创建索引而出错**。



现在就可以通过权限认证进而看到相关表信息了。

```mongodb
> use admin
> db.auth("admin", "123456")
> show dbs
```

![image-20200808153145158](./img/image011.png)

注意，这里仍然看不到 yapi 库是由于 yapi 库中并没有数据，mongodb 中的数据库被设计为只有在有**数据被真正添加时才会创建**。



## YApi 部署

这里采用[官方部署文档](https://hellosean1025.github.io/yapi/devops/index.html)中的命令行方式进行 YApi 部署。



创建 YApi 目录，git 拉取项目。

```bash
mkdir /usr/local/yapi
cd /usr/local/yapi
git clone https://gitee.com/mirrors/YApi vendors
```



修改 YApi 配置文件 config.json。

```bash
cp vendors/config_example.json ./config.json
vim config.json
```

![image-20200808153700946](./img/image012.png)



安装生产依赖，执行部署。

```bash
cd vendors
npm install --production --registry https://registry.npm.taobao.org
```

![image-20200808153744445](./img/image013.png)

这里有报错，提示 mysql 用户没有权限去创建相关目录，这是由于**npm 命令禁止直接 root 操作**，所以此时自动切换成了 mysql 用户来执行命令，导致权限不够。可在 npm 命令后加上 `--unsafe-perm` 来忽略这种安全策略。



npm 启动 YApi 服务。

```bash
npm run install-server
```

![image-20200808154333241](./img/image014.png)



访问 YApi。

![image-20200808154659862](./img/image015.png)



同时可见 yapi 目录下已生成了初始化锁。

![image-20200808154714919](./img/image016.png)

若需要重新初始化 yapi 服务，请先删除该文件 `init.lock`，同时清空 mongodb 中 yapi 库的所有数据。最后**重新在 vendors 目录下**执行 `npm run install-server` 即可。



常见问题。

![image-20200808155042072](./img/image017.png)

连接 mongodb 认证失败，这是由于 mongodb.conf 默认的 `bind_ip=127.0.0.1`。而此时上文中 yapi 配置文件 config.josn 中的 `servername=192.168.33.100` 配置项有误。

可以采用更改 `bind_ip=0.0.0.0` 即不限制 mongodb 的远程连接来解决这个问题。当然也可以修改 config.json 中 `servername=127.0.0.1` 亦可。



## 后台运行

这里可以采用 pm2 对 node 来进行管理，也可以简单点，直接使用 `nohup + &` 即可。

```bash
nohup node vendor/server/app.js &
```

![image-20200808155513910](./img/image018.png)



在执行 `nohup` 命令的文件夹下生成了 nohup.out 文件。

![image-20200808155527394](./img/image019.png)



退出终端，检查 YApi 是否还能访问。

![image-20200808155538849](./img/image020.png)

这里最好使用 `exit` 命令结束 Shell，直接叉掉终端有 mobaxterm 的谜之 BUG 导致 nohup 失效。



没啥问题，部署成功。

![image-20200808155621166](./img/image021.png)

