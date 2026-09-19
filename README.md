## 搭建 nginx + rtmp 服务器，注意此服务器不支持H265编码格式的视频推流（拉流端只能听到声音，看不到画面）
> 阿里云 轻量应用服务器
> 通用型 2vCPU 2GiB ESSD云盘 40GiB
> 北京
> Debian12.10

## 省时版
> 开启端口和obs直播设置，以及使用 curl 录制命令，要向下看一看
```
apt update && \
apt install -y build-essential git libpcre3 libpcre3-dev libssl-dev zlib1g-dev unzip && \
cd /root && \
git clone https://github.com/wenyamu/nginx-rtmp.git && \
cd nginx-rtmp && \
tar -xf nginx-1.31.6.tar.gz && \
unzip nginx-rtmp-module-src-master.zip && \
cd nginx-1.31.6 && \
./configure --with-http_ssl_module --add-module=../nginx-rtmp-module-src-master --with-file-aio && \
make -j 1 && \
make install && \
cd /root/nginx-rtmp && \
cp -f index.html /mnt/index.html && \
cp -f nginx.conf /usr/local/nginx/conf/nginx.conf && \
cp -f nginx-rtmp-module-src-master/stat.xsl /mnt/stat.xsl && \
mkdir -p /mnt/recordings && \
mkdir -p /mnt/manual_recordings && \
mkdir -p /mnt/back_recordings && \
chown -R nobody:nogroup /mnt/recordings && \
chown -R nobody:nogroup /mnt/manual_recordings && \
chown -R nobody:nogroup /mnt/back_recordings && \
chmod 755 /mnt/recordings && \
chmod 755 /mnt/manual_recordings && \
chmod 755 /mnt/back_recordings && \
/usr/local/nginx/sbin/nginx
```

## 1. 编译安装 nginx
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

## 2. 修改nginx配置文件，放在 http{......} 前
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
            
            #自动录制，视频和音频, 用于监测移动物体时合成使用 1分钟一个文件
            recorder auto_full_60 {
              record all; # 自动录制
              record_path /mnt/recordings; # 指定存储路径
              record_unique on;        # 文件名加时间戳，相当于自定义中的 %s，避免覆盖
              
              #record_suffix .flv; # 指定后缀名（可选，开启时间戳时，默认名是 推流码-1789636031.flv）
              #record_suffix -%Y%m%d-%H%M%S.flv; # 自定义文件名格式（可选，开启时间戳时，默认是 推流码-1789636031-自定义.flv）
              
              #这二个同时启用好像有点问题，好像不是连续录，后续再研究把前后两个视频拼起来看是否连续
              record_interval 60; # 录制单个文件的时长（秒）自动切割一次文件，仅自动录制时有效
              #record_max_size 10M; # 每个文件最大 10MB，超过后自动新建文件，仅自动录制时有效
              
            }
            
            
            #自动录制，视频和音频 用于正常的监控备份 30分钟一个文件
            recorder auto_full_1800 {
              record all; # 自动录制
              record_path /mnt/back_recordings; # 指定存储路径
              record_unique on;        # 文件名加时间戳，相当于自定义中的 %s，避免覆盖
              
              #record_suffix .flv; # 指定后缀名（可选，开启时间戳时，默认名是 推流码-1789636031.flv）
              #record_suffix -%Y%m%d-%H%M%S.flv; # 自定义文件名格式（可选，开启时间戳时，默认是 推流码-1789636031-自定义.flv）
              
              #这二个同时启用好像有点问题，好像不是连续录，后续再研究把前后两个视频拼起来看是否连续
              record_interval 1800; # 录制单个文件的时长（秒）自动切割一次文件，仅自动录制时有效
              #record_max_size 10M; # 每个文件最大 10MB，超过后自动新建文件，仅自动录制时有效
              
            }
            
            #手动录制，视频和音频 all
            recorder full_rec {
              record all manual; # 手动录制
              record_path /mnt/manual_recordings; # 指定存储路径
              record_unique on;        # 文件名加时间戳，相当于自定义中的 %s，避免覆盖
              #record_suffix .flv; # 指定后缀名（可选，开启时间戳时，默认名是 推流码-1789636031.flv）
              record_suffix -%Y%m%d-%H%M%S.av.flv; # 自定义文件名格式（可选，开启时间戳时，默认是 推流码-1789636031-自定义.flv）
            }
            
            #手动录制，只录制视频无音频 video
            recorder video_rec {
              record video manual; # 手动录制
              record_path /mnt/manual_recordings; # 指定存储路径
              record_unique on;        # 文件名加时间戳，相当于自定义中的 %s，避免覆盖
              #record_suffix .flv; # 指定后缀名（可选，开启时间戳时，默认名是 推流码-1789636031.flv）
              record_suffix -%Y%m%d-%H%M%S.v.flv; # 自定义文件名格式（可选，开启时间戳时，默认是 推流码-1789636031-自定义.flv）
            }
            
            #手动录制，只录制音频 audio
            recorder audio_rec {
              record audio manual; # 手动录制
              record_path /mnt/manual_recordings; # 指定存储路径
              record_unique on; # 文件名加时间戳，相当于自定义中的 %s，避免覆盖
              #record_suffix .flv; # 指定后缀名（可选，开启时间戳时，默认名是 推流码-1789636031.flv）
              record_suffix -%Y%m%d-%H%M%S.a.flv; # 自定义文件名格式（可选，开启时间戳时，默认是 推流码-1789636031-自定义.flv）
            }
            
            hls on; # 开启HLS
            hls_path /mnt/hls/; # HLS 切片存放路径
            hls_fragment 3; # 每个切片时长（秒），越小延迟越低，但请求越多
            hls_playlist_length 60; # 设置 HLS 播放列表（推流码.m3u8 文件）中包含的视频总时长（秒）
            # 禁用以 RTMP 协议从 Nginx 服务器拉取视频流。禁用后无法通过 VLC播放器、http网页播放器观看直播
            #deny play all;
        }
    }
}
```

## 3. 创建录制文件存放目录
```
# 1. 创建目录（如果不存在）
mkdir -p /mnt/recordings && \
mkdir -p /mnt/manual_recordings && \
mkdir -p /mnt/back_recordings

# 2. 查看当前 Nginx 运行用户
ps aux | grep nginx
# 输出类似：nobody  1234 ... nginx: worker process
# 记住这个用户名，假设是 nobody

# 3. 查看 nobody 用户所属的主组
id nobody
# 输出类似：uid=65534(nobody) gid=65534(nogroup) groups=65534(nogroup)
注意看 gid=...(...) 括号里的名字，那就是你应该使用的组名
如果显示的是 nogroup，就用 nobody:nogroup

# 4. 修改目录所有者为 Nginx 运行用户 nobody
chown -R nobody:nogroup /mnt/recordings && \
chown -R nobody:nogroup /mnt/manual_recordings && \
chown -R nobody:nogroup /mnt/back_recordings

# 5. 赋予写入权限
chmod 755 /mnt/recordings && \
chmod 755 /mnt/manual_recordings && \
chmod 755 /mnt/back_recordings
```

## 4. nginx 配置文件中新增监听 http 8080 端口
```
server {
        listen 8080; # 监听一个 HTTP 端口，避免与 Web 服务冲突
        
        # 【关键】在这里配置控制接口
        location /control {
            rtmp_control all;
        }
        
        # 可选：查看统计信息
        location /stat {
            rtmp_stat all;
            rtmp_stat_stylesheet stat.xsl;
        }
        
        location /stat.xsl {
            root /mnt; # stat.xsl 文件的实际路径，此文件是从编译时下载的 nginx-rtmp-module-src 目录下复制过来的
            
            # 强制设置 Content-Type 为 text/xml，告诉浏览器这是可阅读的 XML，防止下载
            default_type text/xml;
            
            # 或者使用 add_header (注意：如果 default_type 生效，add_header 可能不需要，但加上更保险)
            add_header Content-Type "text/xml; charset=utf-8";
        }
    }
```

## 5. 发送控制命令录制直播
> x.x.x.x:8080/control 与8080监听块中的设置的控制接口对应
>
> /record 录制命令入口
> 
> /start 开始录制
> 
> /stop  结束录制
> 
> app   表示 rtmp://x.x.x.x:1935/show 中的 show
> 
> name  表示 推流码
> 
> rec   表示 recorder full_rec { record all manual; ......} 块中的 full_rec
> 
```
# 开始录制
curl "http://x.x.x.x:8080/control/record/start?app=show&name=abc123456&rec=full_rec"
# 结束录制
curl "http://x.x.x.x:8080/control/record/stop?app=show&name=abc123456&rec=full_rec"
```

## 6. 开放服务器端口
> 阿里云 轻量应用服务器 支持后台设置端口

> 如果是其它服务器 Ubuntu 系统，可以使用以下命令

> 此项目只需要用到 80 22 1935 8080 这些端口
```
# 查看防火墙是否开启，以及开放的端口
ufw status verbose

# 允许 TCP 协议的 80 端口，开放端口后一般不用重新加载
ufw allow 80/tcp

# 重新加载规则，开放端口后一般不用重新加载
ufw reload

# 一次性开启多个端口
ufw allow 80/tcp && ufw allow 1935/tcp && ufw allow 8080/tcp

# 开放 8000 到 9000 之间的所有 TCP 端口
ufw allow 8000:9000/tcp
```

## 7. 运行 nginx
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

## 8. 推流客户端设置
> 推流客户端有很多，电脑端常用的 obs, ffmpeg
```
服务器 rtmp://x.x.x.x:1935/show

# 这里的推流码，是在推流软件上直接设置的，不是在服务器上，别人想看你的直播就必须有这个密码才行
推流码 abc123456

# 完整的 rtmp 协议，可以在支持 串流播放的本地播放器上直接看，比如：VLC media player
rtmp://x.x.x.x:1935/show/abc123456

# ffmpeg 推流设置
ffmpeg -loglevel verbose \
  -stream_loop -1 \
  -re \
  -fflags +genpts \
  -avoid_negative_ts make_zero \
  -i /usr/local/output-superfast5-crf18.mp4 \
  -c copy \
  -f flv rtmp://x.x.x.x:1935/show/abc123456

# 香橙派配合usb摄像头推流（如果想采集声音，ffmpeg 必须支持 alsa）
ffmpeg \
-f v4l2 -input_format mjpeg -video_size 1280x720 -framerate 30 -i /dev/video0 \
-f alsa -ar 48000 -ac 2 -i hw:3,0 \
-c:v libx264 -tune zerolatency -preset ultrafast -pix_fmt yuv420p -g 30 \
-b:v 4M -maxrate 8M -bufsize 16M \
-c:a aac -b:a 64k \
-map 0:v -map 1:a \
-f flv "rtmp://x.x.x.x:1935/show/abc123456"

```

## 9. 使用浏览器打开直播
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
	     <source src="hls/abc123456.m3u8" type="application/x-mpegURL" />
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

### 看直播
> 可以直接打开网页  http://x.x.x.x 点击看直播
>
> 也可以通过播放器打开 http://x.x.x.x/hls/abc123456.m3u8 或 rtmp://x.x.x.x:1935/show/abc123456 看直播
>
> 还可以通过 http://x.x.x.x:8080/stat 查看数据
