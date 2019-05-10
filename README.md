# SORALIVE 部署指南

本文说明如何在一台全新的Ubuntu Server 18.04上部署SORALIVE。

## 准备

首先当然是准备一台全新安装的Ubuntu Server 18.04。并自行准备需要的小工具，如vim、git之类的。编译环境多半不需要，除非你的Linux系统版本比较低，我下面说的这些东西你找不到，那你只能去编译了。

目前的SORALIVE是**单机版**，虽然它在开发之初就设计为多机模式，但是目前版本还不支持，只能把所有组件部署在一台服务器上。但是组件选择依赖nginx的虚拟主机，这是靠不同的域名来区分的，所以SORALIVE仍然需要多个域名配合使用。

请在DNS处设置多个二级域名指向这台服务器。如果在本机部署测试或开发环境，请修改hosts文件将这些域名指向本机。本文以实际的SORALIVE域名为例来解释这些域名的作用，在实际部署时，请修改这些域名为你自己设定的域名。

| 域名 | 用途 |
| --- | --- |
| www.minyami.net | 前端服务器 |
| api.minyami.net | 后端服务器 |
| live.minyami.net | RTMP推流服务器 |
| stream.minyami.net | HLS视频片段存储服务器 |

SORALIVE是全HTTPS访问的，请为除了`live.minyami.net`以外的每个域名准备一张HTTPS证书，或者你也可以准备一张泛域名证书。如果你是在本机部署测试或开发环境，请参考[这篇文章](https://www.zyzsdy.com/article/83)准备一些自签名证书。

## 安装依赖组件

### 安装Nginx

SORALIVE推荐使用Nginx作为HTTP服务器，并且SORALIVE的RTMP直播组件是由[nginx-rtmp-module](https://github.com/arut/nginx-rtmp-module)提供的。

在Ubuntu 18.04中，这一组件以mod的形式在apt库中提供了。因此安装变得非常简单。

```bash
apt install nginx libnginx-mod-rtmp
```

若没有这样的组件，则需要尝试编译安装，请参考[官方文档](https://github.com/arut/nginx-rtmp-module/wiki/Installing-via-Build)的编译指南。

### 安装后端和数据库所需组件

SORALIVE的后端使用PHP，数据库使用Mariadb，并且使用Memcached作为缓存。

还是那句话，在Ubuntu 18.04中，这些东西在apt里全都有。没有的自己编译。

```bash
apt install php7.2-fpm php7.2-mysql php7.2-mbstring php7.2-curl
apt install mariadb-server mariadb-client
apt install memcached php-memcached
```

## 配置RTMP服务器

创建一个目录用来保存直播时的HLS片段：

```bash
cd /var/www/html
mkdir streaming
```

我们这里使用`/var/www/html/streaming`作为存储HLS片段的目录。

打开`/etc/nginx/nginx.conf`，在末尾加入下面内容：

```
rtmp {
        server {
                listen 1935;
                chunk_size 4000;
                notify_method get;

                application livesend {
                        live on;
                        hls on;
                        hls_path /var/www/html/streaming;
                        hls_fragment 3s;
                        hls_fragment_naming system;
                        hls_fragment_naming_granularity 200;
                        on_publish http://localhost/live/rtmp_server/publish;
                }
        }
}
```

请关注如下配置项：

* application的命名，此处命名为`livesend`，可根据自己需要修改；
* hls_path —— 该目录是存放直播中视频片段的目录；
* on_publish —— 该地址是后端允许推流的验证URL，只能使用HTTP；

接下来配置用来拉流的服务器：

在`/etc/nginx/sites-available`下新建一个配置文件`stream.conf`。内容参考[nginx-config-example/stream.conf](nginx-config-example/stream.conf)。

> **注意需要修改的部分：**
> * `root`后面的配置修改为的实际存储HLS片段的目录；
> * `server_name`修改为HLS视频片段存储服务器的实际域名；
> * `ssl_certificate`和`ssl_certificate_key`修改为实际的证书和密钥文件。

然后创建一个到`sites-enable`的软链接：

```bash
ln -s /etc/nginx/sites-available/stream.conf /etc/nginx/sites-enabled/stream.conf
```

## 设置数据库

我们先初始化一下数据库：

```bash
mysql_secure_installation
```

给数据库添加一个新用户，这里我们以用户名`soralive`，密码`soralive`，创建一个名为`soralive`的数据库并授予该新用户所有权限。

启动mysql命令行，输入如下查询：

```sql
CREATE USER 'soralive'@'localhost' IDENTIFIED BY 'soralive';
GRANT USAGE ON *.* TO 'soralive'@'localhost' REQUIRE NONE WITH MAX_QUERIES_PER_HOUR 0 MAX_CONNECTIONS_PER_HOUR 0 MAX_UPDATES_PER_HOUR 0 MAX_USER_CONNECTIONS 0;
CREATE DATABASE IF NOT EXISTS `soralive`;
GRANT ALL PRIVILEGES ON `soralive`.* TO 'soralive'@'localhost';
```

退出命令行，准备数据库结构文件[soralive.sql](database-structure/soralive.sql)，然后使用我们刚刚新建的用户导入这个sql文件。

```bash
mysql -usoralive -psoralive soralive < soralive.sql
```

这会默认创建一个名为Sora，密码为123456的初始用户。

## 部署后端

### 拉取代码

从主分支拉取后端代码到任意一个你存放后端代码的目录。我这里使用的是`/var/www/soralive-backend`。

```bash
cd /var/www
git clone https://github.com/minyami-net/soralive-backend.git
```

### 修改默认配置文件

编辑`/var/www/soralive-backend/live/Feely/config.php`（请根据你的后端代码部署位置找到这个文件）。这里有几处配置需要修改：

* `streaming_address` 配置项中的域名和application名；
* `livestreaming_prefix` 配置项中的域名和相对地址；
* `db_host`、`db_port`、`db_user`、`db_pass`、`db_database` 几项根据配置好的数据库连接情况修改。

### 配置nginx

在`/etc/nginx/sites-available`下新建一个配置文件`backend.conf`。内容参考[nginx-config-example/backend.conf](nginx-config-example/backend.conf)。

> **注意需要修改的部分：**
> * 两处`root`后面的配置修改为后端代码的实际存储位置；
> * SSL服务器中的`server_name`修改为后端api的实际域名；
> * SSL服务器中的`ssl_certificate`和`ssl_certificate_key`修改为实际的证书和密钥文件。

注意80端口的部分实际上只针对localhost开放了。本系统绝大多数时候使用443端口上的https连接，但是需要一个内部连接仅供rtmp服务器的callback使用。

不要忘了创建到`sites-enable`的软链接。之后重启php-fpm和nginx，后端部署就结束了。

```bash
ln -s /etc/nginx/sites-available/backend.conf /etc/nginx/sites-enabled/backend.conf
systemctl restart php7.2-fpm
systemctl restart nginx
```

## 部署前端

### 安装Node.js

众所周知，apt库里是没有node.js的，作为Ubuntu用户，一般比较喜欢使用NodeSource维护的第三方源，还挺好用的。如果你不爱的话可以下载官方编译的Node Linux二进制文件然后手工安装，也挺方便的。

```bash
curl -sL https://deb.nodesource.com/setup_12.x | bash -
apt install nodejs
```

### 拉取代码

从主分支拉取前端代码到任意一个你存放前端代码的目录。我这里使用的是`/var/www/soralive-frontend`。

```bash
cd /var/www
git clone https://github.com/minyami-net/soralive-frontend.git
```

### 修改配置并打包前端

前端只有一处配置需要修改：在`src/globalconst.js`里。修改apiRoot的值为对应的域名。

然后就可以进行前端的打包。回到`/var/www/soralive-frontend`目录：

```bash
cd /var/www/soralive-frontend
npm install
npm run build
```

打包好的文件会生成在dist中。下面直接发布dist这个目录即可。

### 配置Nginx


在`/etc/nginx/sites-available`下新建一个配置文件`frontend.conf`。内容参考[nginx-config-example/frontend.conf](nginx-config-example/frontend.conf)。

> **注意需要修改的部分：**
> * `root`后面的配置修改为`dist`目录；
> * `server_name`修改为前端实际域名；
> * `ssl_certificate`和`ssl_certificate_key`修改为实际的证书和密钥文件。

创建到`sites-enable`的软链接之后重新加载nginx配置。

```bash
ln -s /etc/nginx/sites-available/frontend.conf /etc/nginx/sites-enabled/frontend.conf
systemctl reload nginx
```

**恭喜你，至此SORALIVE的部署结束了。**