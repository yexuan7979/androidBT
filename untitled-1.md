---
description: 查询媒体音频类设备支持的编码
---

# A2dp codec check

## 说明

目前商店相关参数一般仅写明是否支持apt-x，将其作为一个高音质卖点。然而，aptX HD、AAC等等却少有说明；那么，对于一款耳机除了必需支持的SBC编码，怎么查还支持哪种类型的编码呢？

## 准备材料

手机

 一台可以获取蓝牙日志（hci log）的手机，开此日志一般需要进入开发者模式，然后启用蓝牙HCI日志收集；生成的文件Android 原生格式为btsnoop\_hci.log，默认在sdcard根目录下，部分厂商在data目录下。

PS：启用日志收集后，如果未生成hci log，尝试开关一下蓝牙进行确认。

## 操作步骤

连接蓝牙耳机，播放音乐

## 检查编码

打开Hci log，打开工具可选择wireshark或者frontline comprobe protocol analysis system，两者呈现方式略有差异；这里选用后者进行说明

在avdtp signal标签页可以看到discover, get capability, set configuration数据包交互，下面一一说明此交互的意义；顺带提一下，里面的灰色包表示底层（controller）发送给蓝牙协议栈（stack），白色为stack 发往controller.

![](.gitbook/assets/undefined%20%288%29.png)

1、discover命令

discover命令旨在找到对方设备是否有可连接的音频端（stream）

    endpoint ID：用于区分支持的stream

    in-use：    是否在使用

    media type：音频类型

    TSEP：音源端 or 接收端；SNK表示接收端，接收stream流；SRC表示音源端，发送stream流

如上截图表示对方设备支持2个sink端，2个source端，且不在使用

2、get capability

用于获取音频端的详细信息（所支持编码格式就可以从此处得出），由于通用手机端是作为source端，所以查询的是对方的sink端信息

如下截图中，endpoint = 1的stream为SBC编码（media codec type字段）

![](.gitbook/assets/undefined%20%287%29.png)

endpoint = 2 的stream为vendor specific codec，从codec info 0x 0a来看，暂时未查到是何种编码

![](.gitbook/assets/undefined%20%289%29.png)

下面是从另一设备获取到的不同编码，如下为vendor specific codec，从codec info 0x 4f 可以知道为apt-x编码（高通csr官网查询）

![](.gitbook/assets/undefined%20%286%29.png)

media codec type：MPEG-1,2 Audio

![](.gitbook/assets/undefined%20%285%29.png)

media type：MPEG-2,4 AAC

![](.gitbook/assets/undefined%20%284%29.png)



3、set configuration

双方协商音频编码，如下图，选择为apt-x编码；手机端一般根据对方支持的音频编码，结合本身所支持的编码格式进行选择，通常选择优先级为apt-x &gt; AAC &gt; SBC

![](.gitbook/assets/undefined%20%282%29.png)



## 小结

1、设备支持的编码从hci log中的get capability中获取；

2、最终所选编码可从set configuration中得到。



另外，从logcat中打印的log也能获知一二：

```text
bt_btif : bta_av_co_audio_peer_supports_codec codecId = 1
bt_btif : bta_av_co_audio_peer_supports_codec vendorId = 4f
--> apt-x 编码

bt_btif : bta_av_co_audio_peer_supports_codec codecId = 36
bt_btif : bta_av_co_audio_peer_supports_codec vendorId = 7
--> apt-x hd编码

bt_btif : bta_av_co_audio_peer_supports_codec AAC
-->AAC编码

bt_btif : bta_av_co_audio_peer_supports_codec SBC
-->SBC编码
```

当然，部分厂商可能不打印类似log。

