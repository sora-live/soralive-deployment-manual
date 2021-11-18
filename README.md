# SORALIVE 部署指南

本文说明如何部署SORALIVE

以Ubuntu 20.04为例

## 准备

推荐从全新安装的Ubuntu 20.04开始，并安装好vim、git之类的小工具。如果你用的不是Ubuntu 20.04，尽量用该发行版的最新稳定版本，以保证下面的东西都能在自带包管理中找到。如果自带包管理没有，请按满足条件的最新稳定版编译安装。

目前的SORALIVE部署在两台Ubuntu 20.04上，其中一台作为Web服务器，部署前端和后端，另一台作为推流服务器。不过，前端服务器只是一个静态文件服务器，可以独立部署。

当然也可以把所有组件部署在同一台服务器上。注意组件选择依赖Nginx的虚拟主机功能，通过不同的域名来区分，所以SORALIVE需要多个域名配合使用。

如果在本机部署测试或开发环境，请修改hosts文件将这些域名指向本机。本文以实际的SORALIVE域名为例来解释这些域名的作用，在实际部署时，请修改这些域名为你自己设定的域名。

| 域名 | SORALIVE使用中的域名 | 用途 |
| --- | --- | --- |
| www | www.minyami.net | 前端服务器 |
| api | api.minyami.net | 后端服务器 |
| rtmp | rtmp.minyami.net | RTMP推流服务器 |
| stream | stream.minyami.net | HLS视频片段服务器 |
| api-callback | api-callback.minyami.net:41071 | （可选）推流服务器回调（安全原因解析到内网地址）
| rtmp-stat | rtmp-stat.minyami.net | （可选）RTMP服务器状态查询（安全原因解析到内网地址）

注意用户推流到推流服务器后，会将视频片段保存在本机，所以我们把推流服务器和视频片段服务器部署在同一台服务器上，并使用CDN来缓解这台服务器提供大量流量时的压力。你也可以通过NFS等形式将产生的视频片段主动推送给另一台专用的视频片段服务器。

SORALIVE是全HTTPS访问的（除了一个例外情况，下面会提到），请为除了rtmp域名和api-callback域名以外的每个域名准备一张HTTPS证书，或者你也可以准备一张泛域名证书。如果你是在本机部署测试或开发环境，请参考[这篇文章](https://www.zyzsdy.com/article/83)准备一些自签名证书。

## 1. 部署RTMP推流服务器和HLS视频片段服务器

这是我们的第一台服务器，RTMP推流服务器和HLS视频片段服务器将部署在这台服务上面。

设置rtmp域名和stream域名指向这台服务器的IP。

### 安装Nginx

SORALIVE推荐使用Nginx作为HTTP服务器，并且SORALIVE的RTMP直播组件是由[nginx-rtmp-module](https://github.com/arut/nginx-rtmp-module)提供的。

从Ubuntu 18.04开始，这一组件以mod的形式在apt库中提供了。因此安装变得非常简单。

```bash
apt install nginx libnginx-mod-rtmp
```

若没有这样的组件，则需要尝试编译安装，请参考[官方文档](https://github.com/arut/nginx-rtmp-module/wiki/Installing-via-Build)的编译指南。

### 配置RTMP推流服务器

创建一个目录用来保存直播时的HLS片段，我们这里就使用默认的`/var/www/html`，在其中建立子目录`streaming`存储HLS片段：

```bash
cd /var/www/html
mkdir streaming
```

打开`/etc/nginx/nginx.conf`，在末尾加入下面内容：

```conf
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
                        hls_playlist_length 30s;
                        hls_fragment_naming system;
                        hls_fragment_naming_granularity 200;
                        on_publish http://api-callback.minyami.net:41071/apiv2/rtmp_server/publish;
                }
        }
}
```

如下配置项可根据需要修改：

* application的命名

此处命名为`livesend`，实际RTMP推流地址为`rtmp://rtmp.minyami.net/livesend`。如果修改后注意之后配置后端的时候需要对应修改。

* hls_path

该目录是存放直播中视频片段的目录。

* hls_fragment

当前配置值`3s`表示HLS分片的每一片长度为3秒。降低这个数字可以降低延迟，但是会增加客户端卡顿的概率。注意配置时一定要带上单位`s`。

>注意因为HLS只会按照关键帧切分，所以如果推流者的GOP（关键帧间距）设置为大于分片长度的值，那么最小分片长度只能是GOP长度。

* hls_playlist_length

生成的m3u8播放列表的长度。设置为`30s`表示m3u8播放列表中有30秒长度的直播流分片信息。注意配置时一定要带上单位`s`。

* on_publish

该地址是后端允许推流的验证URL，只能使用HTTP。

>注意如果使用仅HTTPS的CDN，会无法通过HTTPS访问到这个API，可能需要给这个API单独开一条通道，设置一个独立的域名。

### HLS视频片段服务器

接下来在同一台服务器上，配置用来拉流的服务器：

在`/etc/nginx/sites-available`下新建一个配置文件`stream.conf`。内容参考[nginx-config-example/stream.conf](nginx-config-example/stream.conf)。

> **注意需要修改的部分：**
> * `root`后面的配置修改为的实际存储HLS片段的目录，注意要设置在`streaming`子目录的上级；
> * `server_name`修改为HLS视频片段存储服务器的实际域名；
> * `ssl_certificate`和`ssl_certificate_key`修改为实际的证书和密钥文件。

然后创建一个到`sites-enable`的软链接：

```bash
ln -s /etc/nginx/sites-available/stream.conf /etc/nginx/sites-enabled/stream.conf
```

### RTMP服务器状态查询（可选）

可以在同一台服务器上配置RTMP服务器状态查询。当访问指定的接口时，会以XML格式返回当前推流状态的信息。包括正在推流的频道名（推流码的一部分，由SORALIVE自动管理）、IP地址，以及正在推送的视频和音频的编码、比特率、总流量大小、网络流量等信息。

可以安装官方自带的XSL文件渲染为独立查询页面；也可以作为API使用，自行开发查询界面（SORALIVE目前还没有此功能）。

新建一个目录`/var/www/rtmp-stat`，然后下载[stat.xsl](https://github.com/arut/nginx-rtmp-module/blob/master/stat.xsl)文件放置在这个目录中。

```bash
mkdir rtmp-stat
cd /var/www/rtmp-stat
wget https://raw.githubusercontent.com/arut/nginx-rtmp-module/master/stat.xsl
```

在`/etc/nginx/sites-available`下新建一个配置文件`show-rtmp-stat.conf`。内容参考[nginx-config-example/show-rtmp-stat.conf](nginx-config-example/show-rtmp-stat.conf)。

> **注意需要修改的部分：**
> * `root`后面的配置修改为的实际存储`stat.xsl`文件的目录，这里是`/var/www/rtmp-stat`；
> * `server_name`修改为状态查询的域名；
> * `ssl_certificate`和`ssl_certificate_key`修改为实际的证书和密钥文件。
> * `rtmp_stat_stylesheet`设置的是`stat.xsl`的文件名，如果修改了这个文件名或者放置在子目录里，需要修改这里的配置。

> 虽然一般来说`listen`都设置为对整个网络监听，但是这里是特殊数据，可能不希望被任意用户查询，可以设置IP监听范围，仅允许特定服务器才能访问。也可以正常设置，在防火墙处进行安全限制。

注意其中有注释掉的`/control`段落，nginx-rtmp-module自带一个control模块，通过control模块提供的API，程序可以控制录像、断开客户端的连接或者重定向到另一个连接。如果需要这种外部控制能力，可以查看[Control模块的API文档](https://github.com/arut/nginx-rtmp-module/wiki/Control-module)，并自行开发功能（SORALIVE目前没有这类功能）。

如果需要使用Control模块，你需要取消这三行的注释。

然后创建一个到`sites-enable`的软链接：

```bash
ln -s /etc/nginx/sites-available/show-rtmp-stat.conf /etc/nginx/sites-enabled/show-rtmp-stat.conf
```

### 重启nginx

你可以通过`nginx -t`命令检查配置文件是否有错误。全部配置好后，你需要通过

```bash
systemctl restart nginx
```

重启Nginx来让全部配置生效。

这样我们就配置好了第一台服务器。

## 2. 部署后端服务器

我们将在这台服务器上部署后端服务器。后端服务器上将部署数据库、作为缓存和消息队列的Redis服务器，以及提供API服务的Node.js服务器。

当然，我们还需要Nginx作为Node.js服务器的反向代理。

### 安装Nginx、数据库和Redis和依赖工具

SORALIVE使用Mariadb作为数据库，使用Redis作为缓存和消息队列。

```bash
apt install nginx
apt install mariadb-server mariadb-client
apt install redis-server redis-cli
```

这台服务器上的Nginx只用来作为反向代理，所以不需要额外安装其他的库。如果你是单机部署，不需要重复安装Nginx，可以直接使用之前安装的那个。

由于我们不需要远程访问数据库和Redis，所以不需要修改它们的配置。安装完毕后就可以使用了。

后端使用OpenSSL进行加密，所以也需要安装OpenSSL，不过Ubuntu 20.04上，OpenSSL作为系统级依赖被预装。

你可以使用`openssl version`和`whereis openssl`命令确定系统中是不是存在OpenSSL。

如果没有，可以用以下命令安装：

```bash
apt install openssl
```

### 设置数据库

我们先初始化一下数据库：

```bash
mysql_secure_installation
```

然后我们需要给数据库添加一个新用户，这里我们以用户名`soralive`，密码`soralive`，创建一个名为`soralive`的数据库并授予该新用户所有权限。

启动mysql命令行，输入如下查询：

```sql
CREATE USER 'soralive'@'localhost' IDENTIFIED BY 'soralive';
GRANT USAGE ON *.* TO 'soralive'@'localhost' REQUIRE NONE WITH MAX_QUERIES_PER_HOUR 0 MAX_CONNECTIONS_PER_HOUR 0 MAX_UPDATES_PER_HOUR 0 MAX_USER_CONNECTIONS 0;
CREATE DATABASE IF NOT EXISTS `soralive`;
GRANT ALL PRIVILEGES ON `soralive`.* TO 'soralive'@'localhost';
FLUSH PRIVILEGES;
```

### 安装Node.js

众所周知，apt库里是没有Node.js的，作为Ubuntu用户，我们一般比较喜欢使用NodeSource维护的第三方源，还挺好用的。如果你不爱的话可以通过nvm安装，也可以下载官方编译的Linux二进制文件手工安装。

```bash
curl -sL https://deb.nodesource.com/setup_16.x | bash -
apt install nodejs
```

### 拉取后端代码

从主分支拉取后端代码到存放后端代码的目录。我这里使用的是`/var/www/soralive-backend-koa`。

```bash
cd /var/www
git clone git@github.com:minyami-net/soralive-backend-koa.git
```

### 修改默认配置文件

复制`/var/www/soralive-backend/config.default.json`（请根据你的后端代码部署位置找到这个文件），并重命名为`/var/www/soralive-backend/config.json`。

然后根据实际情况修改这里的配置：

* `streaming_address` 是你的推流地址，格式为`rtmp://rtmp域名/application名`，注意这里结尾没有斜杠。
* `livestreaming_prefix` 是用户下载直播流m3u8的地址前缀，格式为`https://stream域名/streaming/`，注意最后结尾的斜杠必须带上。
* `opensslbin` 是本机OpenSSL可执行文件的路径。注意后端的运行环境不会读取PATH环境变量，所以必须写完整路径。
* `dbSync` 是启动时数据库操作模式，`enabled drop`会删除原来的数据库并重建，`enabled alter`会修改当前数据库来保证一致（升级时使用），`enabled`会创建不存在的数据库，`disabled`不会进行操作。首次安装时需要初始化数据库，我们保持`enabled drop`不变。
* `mysql` 是数据库连接参数，根据上面初始化数据库时的配置修改。
* `redis` 是Redis连接参数，同样根据实际情况修改。
* `passHashKeys` 是一个两个字符串的数组，用于加强密码哈希。请任意修改这两个随机字符串。
* `streamHaskKey` 是一个随机字符串，用于加强推流码哈希，请任意修改这个字符串。
* `open_user_reg` 是一个布尔值，控制系统是否允许注册，`true`为允许注册。我们初始化阶段保持这个值为`true`，如果需要关闭注册，请在初始化完成后再修改为`false`。

### 编译后端并完成数据库初始化

使用以下代码安装Node.js依赖并编译后端：

```bash
npm install --allow-root
npm run build
```

> 即使是使用root用户执行`npm install`，仍然可能碰到权限不足的问题。
> 添加`--allow-root`参数来让npm自己使用root账户执行命令。

如果在运行期间没有报错，会建立dist目录，编译后的文件会出现在dist目录中。

进入后端安装目录，并直接运行主程序，首次启动时将完成数据库初始化。

```bash
cd /var/www/soralive-backend-koa
node ./dist/app.js
```

在看到提示`Table users has been synchronized.`后按下`Ctrl-C`键结束后端主程序。此时登录数据库，如果看到指定数据库中出现`publish_logs`和`users`两张表，则说明数据库初始化成功。

接下来重新打开`config.json`编辑`dbSync`，修改为`disabled`，以避免下次打开程序重新初始化数据库。

### （可选）安装Systemd Unit

我们把SORALIVE后端安装为Systemd Unit来实现后台启动和自启动。当然你也可以使用自己的方式进行后台启动和自启动，有非常多的人使用*pm2*进行管理，如果你不喜欢Systemd Unit的方式，推荐尝试一下[pm2](https://pm2.keymetrics.io/)。

当然本文还是以Systemd Unit的模式为例。

在安装目录下创建一个文件，名为`soralive.service`。内容参考[systemd-unit/soralive.service](systemd-unit/soralive.service)。

然后使用软链接的方式将文件放到`/etc/systemd/system`目录下

```bash
ln -s /var/www/soralive-backend-koa/soralive.service /etc/systemd/system/soralive.service
```

然后重新加载配置文件

```bash
systemctl daemon-reload
```

使用`systemctl status soralive`查看服务状态。如果成功识别到，说明配置正确，接下来就可以使用`systemctl`对其进行控制了。

设置开机自动启动，并启动服务：

```bash
systemctl enable soralive
systemctl start soralive
```

### 配置Nginx

后端服务器仍然使用Nginx作为反向代理处理HTTP请求。

在`/etc/nginx/sites-available`下新建一个配置文件`backend.conf`。内容参考[nginx-config-example/backend.conf](nginx-config-example/backend.conf)。

> **注意需要修改的部分：**
> * 两处`server_name`修改为后端api的实际域名；
> * SSL服务器中的`ssl_certificate`和`ssl_certificate_key`修改为实际的证书和密钥文件。

注意HTTP服务器的部分实际上只针对内网地址监听。本系统绝大多数时候使用443端口上的HTTPS连接，但是需要一个内部连接仅供RTMP服务器的回调使用。

不要忘了创建到`sites-enable`的软链接。之后重启Nginx，后端部署就大功告成。

```bash
ln -s /etc/nginx/sites-available/backend.conf /etc/nginx/sites-enabled/backend.conf
systemctl restart nginx
```

## 3. 部署前端服务器

### 安装Node.js和Nginx

我们把前端部署在和后端的同一台服务器上，因为上面安装过了，所以这些操作忽略不做。如果你也是这样或者干脆All in one的话，那也可以跳过这一节。

如果你是安装一台全新的服务器，就再把这些操作重新来一遍：

```bash
curl -sL https://deb.nodesource.com/setup_16.x | bash -
apt install nodejs nginx
```

### 拉取前端代码

从主分支拉取前端代码到任意一个你存放前端代码的目录。我这里使用的是`/var/www/soralive-frontend`。

```bash
cd /var/www
git clone git@github.com:minyami-net/soralive-frontend.git
```

### 修改配置并打包前端

在`src/globalconst.js`里。修改apiRoot和wsHost的域名。

* `apiRoot` 是后端API地址，格式为`https://api域名/apiv2/`，注意必须带有最后的斜杠。
* `wsHost` 是WebSocket服务器的地址，格式为`wss://api域名`，注意此处最后没有斜杠。

然后就可以打包前端。回到`/var/www/soralive-frontend`目录，然后执行：

```bash
cd /var/www/soralive-frontend
npm install --allow-root
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

## 4. 建立第一个用户并设置为管理员

现在你可以打开你的前端首页，检查是否可以看到页面。如果部署成功你应该能看到SORALIVE的首页了。

为你自己注册第一个用户，注册成功后，该用户为普通用户权限。

连接后端服务器的数据库，打开`users`表，你应该能在其中看到自己刚刚建立的用户，使用SQL语句将这个用户的`type`修改为`17`。

SORALIVE使用`type`字段的值，作为一个简易的权限系统。`type`取值和用户权限的关系，请参考[user-auth-type.md](user-auth-type.md)。

然后回到浏览器首页，退出登录再重新登录。现在这个用户已经具有了管理员权限。

**恭喜你，至此SORALIVE的部署结束了。**