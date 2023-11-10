---
title: 加密的ts+m3u8合并
p: others/ts_m3u8.md
date: 2019-07-06 18:20:41
tags: Others
categories: Others
---

加密后的ts文件不能直接合并或播放，需要使用key对每个ts文件进行解密。

分为两种情况：  
(1).如果ts文件已经全部下载好，则可以直接在本地通过ffmpeg快速解密合并。  
(2).如果ts文件没有下载好，则可以通过vlc直接下载整个视频，或者通过ffmpeg下载并转换。  

无论是哪种情况，都要去视频源地址下载m3u8文件。如果可以下载key(有些网站加密方式比较严谨，不那么容易获取到key)，把key文件也下载好。

下载m3u8文件的方式是去源地址网站，按F12找到m3u8文件，或者从右键-->网页源代码中找到地址。两种方式都试一试。

例如，从浏览器的F12中找：

![](/img/others/733013-20180513224409822-2019602854.jpg)

这里能找到两个m3u8和一个key文件，都下载好。记事本打开两个m3u8，其中有一个包含了ts文件列表，这个m3u8文件是我们所需要的。例如我这里的是HdNz1kaz.m3u8文件，以下是一小部分内容。

```
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:13
#EXT-X-MEDIA-SEQUENCE:0
#EXT-X-KEY:METHOD=AES-128,URI="/20180125/NfJJpxIH/1482kb/hls/key.key"
#EXTINF:12.5,
/20180125/NfJJpxIH/1482kb/hls/GBDYO3576000.ts
#EXTINF:12.5,
/20180125/NfJJpxIH/1482kb/hls/GBDYO3576001.ts
#EXTINF:12.5,
/20180125/NfJJpxIH/1482kb/hls/GBDYO3576002.ts
```

# 1.情形一：ts文件已经下载好

假如我的ts文件全部下载好，放在e:\20180125\目录下。

![](/img/others/733013-20180513224956824-1745077681.jpg)

同时假设key文件已经下载好，也放在e:\20180125\目录下。

修改m3u8文件中key的uri路径和ts文件的路径为本地路径。下面是HdNz1kaz.m3u8文件修改后的一小部分内容

```
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:13
#EXT-X-MEDIA-SEQUENCE:0
#EXT-X-KEY:METHOD=AES-128,URI="e:/20180125/key.key"
#EXTINF:12.5,
e:/20180125/GBDYO3576000.ts
#EXTINF:12.5,
e:/20180125/GBDYO3576001.ts
#EXTINF:12.5,
e:/20180125/GBDYO3576002.ts
```

然后用ffmpeg进行合并。
```
ffmpeg -allowed-extensions ALL -i HdNz1kaz.m3u8 -c copy new.mp4
```

我一般会把ts文件下载好，因为用下载工具(比如迅雷)下载比ffmpeg或者vlc下载速度要快的多，因为这两个工具都是串行下载的。

![](/img/referer.jpg)

# 2.情形二：ts文件没有下载

同样，下载好m3u8文件(key可下载可不下载，因为可以直接在m3u8文件中指定key的网络uri路径)。

修改m3u8文件中key和ts的uri路径。下面是HdNz1kaz.m3u8文件修改后的一小部分内容。

```
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:13
#EXT-X-MEDIA-SEQUENCE:0
#EXT-X-KEY:METHOD=AES-128,URI="http://www.example.com/20180125/key.key"
#EXTINF:12.5,
http://www.example.com/20180125/GBDYO3576000.ts
#EXTINF:12.5,
http://www.example.com/20180125/GBDYO3576001.ts
#EXTINF:12.5,
http://www.example.com/20180125/GBDYO3576002.ts
```

然后，使用ffmpeg下载并合并。
```
ffmpeg -i HdNz1kaz.m3u8 -c copy new.mp4
```


# 3.使用vlc下载并保存ts文件

![](/img/others/733013-20180513225947759-1382026162.jpg)


![](/img/others/733013-20180513230402970-792697766.jpg)


![](/img/others/733013-20180513231418312-1429844073.jpg)

![](/img/others/733013-20180513230450022-380416583.jpg)

![](/img/others/733013-20180513230550146-569149226.jpg)

![](/img/others/733013-20180513230630911-1242319478.jpg)

![](/img/others/733013-20180513230656649-1922810512.jpg)

播放列表的下方有播放进度条，它表示下载的进度，不要去动，也不要去点击播放、暂停、停止等，放着别管就是了，直到播放进度条完成了，就表示文件合并完成。去文件保存位置的地方看看就知道了。

![](/img/referer.jpg)


# ffmpeg从m3u8文件转mp4

```
# 如果已经下载好m3u8，且m3u8文件里的路径都是绝对路径而不是相对路径
# 例如：https://cnd.abc.com/video/70a/video000.ts
# 或者：E:/20180125/GBDYO3576000.ts
ffmpeg -protocol_whitelist file,http,https,tcp,tls,crypto -bsf:a aac_adtstoasc -i "./us.m3u8" -c copy filename.mp4

# 如果是相对路径，可以将m3u8文件修改，加上路径前缀，或者直接传递m3u8的URL
# 例如：/20180125/NfJJpxIH/1482kb/hls/GBDYO3576000.ts
ffmpeg -protocol_whitelist file,http,https,tcp,tls,crypto -bsf:a aac_adtstoasc -i "https://abc.def.com/us.m3u8" -c copy filename.mp4
```

# ffmpeg报错

错误：

> Malformed AAC bitstream detected: use the audio bitstream filter 'aac_adtstoasc' to fix it ('-bsf:a aac_adtstoasc' option with ffmpeg)

需要在合并视频的时候，加上`-bsf:a aac_adtstoasc`

```
ffmpeg -i index.m3u8 -c:a copy -bsf:a aac_adtstoasc new.mp4
```

错误：

>Codec for stream 0 does not use global headers but container format requires global headers
>Codec for stream 1 does not use global headers but container format requires global headers

需要加上global header

```
ffmpeg -i index.m3u8 -c:a copy -flags +global_header new.mp4
```







