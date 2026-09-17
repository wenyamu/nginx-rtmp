## 一、创建服务器
> 阿里云 轻量应用服务器
> 通用型 2vCPU 2GiB ESSD云盘 40GiB
> 北京
> Debian12.10
### 编译安装 nginx
> 加入支持 rtmp 协议的模块

> 如果不行，可以试试 https://github.com/sergey-dryabzhinsky/nginx-rtmp-module.git

> ./configure --add-module=../nginx-rtmp-module-src 是新增第三方模块的相对地址，记得要有相应的修改
```
apt update && \
apt install -y build-essential git libpcre3 libpcre3-dev libssl-dev zlib1g-dev && \
git clone https://github.com/nginx-with-docker/nginx-rtmp-module-src.git && \
wget http://nginx.org/download/nginx-1.31.6.tar.gz && \
tar -xf nginx-1.31.6.tar.gz && \
cd nginx-1.31.6 && \
./configure --with-http_ssl_module --add-module=../nginx-rtmp-module-src --with-file-aio && \
make -j 1 && \
make install
```

### 修改nginx配置文件，放在 http{......} 前
> application show {......} 中的 show 与服务器ip 组成服务器推流地址 rtmp://x.x.x.x:1935/show

> 编译安装nginx后，配置文件路径 /usr/local/nginx/conf/nginx.conf
```
# RTMP 实时消息传输协议配置
rtmp {
    server {
        listen 1935; # 监听标准RTMP端口
        chunk_size 4000;

        application show {
            live on;
            hls on; # 开启HLS
            hls_path /mnt/hls/; # 流缓存地址
            hls_fragment 3;
            hls_playlist_length 60;
            # 禁用以 RTMP 协议从 Nginx 服务器拉取视频流。如果开启就无法通过 VLC播放器、http网页播放器观看直播
            #deny play all;
        }
    }
}
```

### 开放服务器端口
> 阿里云 轻量应用服务器 支持后台设置端口

> 如果是其它服务器 Ubuntu 系统，可以使用以下命令

> 此项目只需要用到 80 22 1935 这些端口
```
# 查看防火墙是否开启，以及开放的端口
ufw status verbose

# 允许 TCP 协议的 80 端口，开放端口后一般不用重新加载
ufw allow 80/tcp

# 重新加载规则，开放端口后一般不用重新加载
ufw reload

# 一次性开启多个端口
ufw allow 80/tcp && ufw allow 1935/tcp

# 开放 8000 到 9000 之间的所有 TCP 端口
ufw allow 8000:9000/tcp
```

## 二、运行 nginx
```
#查看配置文件有没有错误
/usr/local/nginx/sbin/nginx -t

#启动 nginx
/usr/local/nginx/sbin/nginx

#停止 nginx
/usr/local/nginx/sbin/nginx -s stop

#重启 nginx
/usr/local/nginx/sbin/nginx -s reload
```

## 三、设置 obs 直播
```
服务器 rtmp://x.x.x.x:1935/show

# 这里的推流码，是在推流软件上直接设置的，不是在服务器上，别人想看你的直播就必须有这个密码才行
推流码 stream

# 完整的 rtmp 协议，可以在支持 串流播放的本地播放器上直接看，比如：VLC media player
rtmp://x.x.x.x:1935/show/stream
```

## 四、使用浏览器打开直播
### 改一下配置文件
>  注意：要把配置文件 `/usr/local/nginx/conf/nginx.conf` 中 `http{......}` 监听 80 端口的配置

```
......
location / {
            root   html; # 对应的是 /usr/local/nginx/html/ 目录
            index  index.html index.htm;
        }
......
```
修改成
```
......
location / {
            root   /mnt/; # 只把这里改成这样即可，对应服务器的 /mnt/
            index  index.html index.htm;
        }
......
```

### 创建 html 文件
```
<html>
<head>
	  <meta charset="utf-8">
	    <meta http-equiv="X-UA-Compatible" content="IE=edge">
	      <meta name="apple-mobile-web-app-capable" content="yes">
	        <meta name="viewport" content="width=device-width, initial-scale=1">
		  <link href="https://vjs.zencdn.net/7.10.2/video-js.css" rel="stylesheet" />
		    <!-- If you'd like to support IE8 (for Video.js versions prior to v7) -->
		      <script src="https://vjs.zencdn.net/ie8/1.1.2/videojs-ie8.min.js"></script>
</head>

<body style="background-color: black">
	  <video
          id="stream"
	      class="video-js vjs-default-skin vjs-fluid"
           controls
           preload="auto"
	       poster="stream.png"
	    data-setup="{}"
         >
	     <source src="hls/stream.m3u8" type="application/x-mpegURL" />
		         <p class="vjs-no-js">
			     To view this video please enable JavaScript, and consider upgrading to a
			         web browser that
				     <a href="https://videojs.com/html5-video-support/" target="_blank">supports HTML5 video</a>
				         </p>
					   </video>
					     <script src="https://vjs.zencdn.net/7.10.2/video.js"></script>
</body>
</html>
```

### http://x.x.x.x 直接打开网页看直播
> http://x.x.x.x/mnt/hls/stream.m3u8
