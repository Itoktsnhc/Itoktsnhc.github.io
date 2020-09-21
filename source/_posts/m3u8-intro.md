---
title: ffmpeg使用与m3u8文件
date: 2020-09-21 14:12:12
category:
- ffmpeg
tags:
- ffmpeg
- m3u8
- 流媒体

---



ffmpeg使用与m3u8文件
<!-- more -->
ffmpeg是一个可以用来对视频、音频做各种处理的工具。参数很多，但是常用的也就几个。

例子
```bat
ffmpeg {全局参数} {输入文件参数} -i {输入文件} {输出文件参数} {输出文件}
```

-c：指定编码器
-c copy：直接复制，不经过重新编码
-c:v：指定视频编码器
-an：去除音频
-vn： 去除视频
-b: 指定码率 例如[-b:v 0.9M] 指定视频的码率

最简单的使用方法为:
``` bat
ffmpeg -i "原始文件.mkv" "输出文件.mp4"
```
需要注意的是当原始文件/输出文件的文件名中存在特殊字符的时候(空格等)，需要使用双引号。


在前段时间尝试抓取一些视频平台的视频内容时，发现很多网站使用了m3u8(播放列表)+xxx.ts(视频文件)的方式.其中m3u8文件为播放列表文件，随后的*.ts文件为实际的视频文件。  
其中m3u8大致内容为：

```m3u8
#EXTM3U 
#EXT-X-VERSION:3 
#EXT-X-TARGETDURATION:4 
#EXT-X-PLAYLIST-TYPE:VOD
#EXT-X-MEDIA-SEQUENCE:0
#EXT-X-KEY:METHOD=AES-128,URI="https://meiju10.zzwc120.com/20200506/LSub3Wch/3000kb/hls/key.key"
#EXTINF:2.419,
https://meiju10.zzwc120.com/20200506/LSub3Wch/3000kb/hls/rcfPCjXD.js
#EXTINF:2.085,
https://meiju10.zzwc120.com/20200506/LSub3Wch/3000kb/hls/CGh71YAo.js
#EXTINF:2.085,
https://meiju10.zzwc120.com/20200506/LSub3Wch/3000kb/hls/sd7ZiHFL.js
.
.
.
.
.
#EXT-X-ENDLIST //结束

```

几个重要的tag：
1. EXTM3U：必须包含的头
2. EXT-X-VERSION：版本
3. EXT-X-KEY：
    1. 加密方法以及密钥，这个tag有可能出现在文件头，或者每个分片上面
        1. METHOD: 加密方法,一般为aes-128
        2. URI: AES解密使用的KEY文件的地址。
        3. IV: AES解密使用的IV
4. EXTINF: 各个分片

基本下载的流程就是：下载m3u8->解析->下载对应的ts->看是否按需解密->合并ts文件.

ffmpeg可以直接下载m3u8:
``` bat
ffmpeg -i https://XXXX/video.m3u8 -c copy  output.mp4
```

需要注意的是ffmpeg目前版本对于ts文件的下载是单线程的，效率很低。如果存在加密的问题，也会比较麻烦，因为一般的网站都会对对应的key做访问限制。可能需要添加对应auth的cookie。


AesDecrypt

``` CSharp
public static byte[] AesDecrypt(byte[] data, byte[] key, string iv = "")
{
    using (var rm = new RijndaelManaged())
    {
        if (key == null || key.Length == 0)
            return data;
        rm.Key = key;
        if (!string.IsNullOrWhiteSpace(iv))
        {
            var tmpIv = iv.Replace("0x", "").Trim();
            var buff = new byte[tmpIv.Length / 2];
            for (var index = 0; index < buff.Length; index++)
            {
                var vStr = tmpIv.Substring(index * 2, 2);
                buff[index] = Convert.ToByte(vStr, 16);
            }

            rm.IV = buff;
        }

        var cTransform = rm.CreateDecryptor();
        var resultArray = cTransform.TransformFinalBlock(data, 0, data.Length);
        return resultArray;
    }
}
```