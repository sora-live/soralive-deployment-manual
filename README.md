# SORALIVE 部署指南

本文说明如何在一台全新的Ubuntu Server 18.04上安装SORALIVE

## 安装Nginx

SORALIVE的rtmp直播服务是由[nginx-rtmp-module](https://github.com/arut/nginx-rtmp-module)提供的。

在Ubuntu 18.04中，这一组件以mod的形式在apt库中提供了。因此安装变得非常简单。

```bash
apt install nginx libnginx-mod-rtmp
```

若没有这样的组件，则需要尝试编译安装，请参考[官方文档](https://github.com/arut/nginx-rtmp-module/wiki/Installing-via-Build)的编译指南。

## 安装后端和数据库

SORALIVE的后端使用PHP，数据库使用Mariadb，并且使用Memcached作为缓存。

```bash
apt install php7.2-fpm php7.2-mysql php7.2-mbstring php7.2-curl
apt install mariadb-server mariadb-client
apt install memcached php-memcached
```

## 配置RTMP服务器

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

1. hls_path —— 该目录是存放直播中视频片段的目录
2. on_publish —— 该地址是后端允许推流的验证URL，只能使用HTTP