---
layout:     post
title:      "音视频 流和协议整理"
subtitle:   ""
date:       2019-09-02 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - media
---



当前音视频方面了解到的各种流和协议的概念，中间有很多容易混淆，整理一下以免忘记。

## 流协议

M2V、 WAV、AC3 等后缀名的音视频文件，属 ES 流范畴。

M2P、MPG、MPEG 等后缀名的文件，属 PS 流。

MTS、M2TS、TS 等后缀名的文件，属 TS 流。

### ES 流 (Elementary Stream)

也叫基本码流，包含视频、音频或数据的连续码流。分视频 ES 流，以及音频 ES 流等。

### PES 流 (Packet Elementary Stream)

也叫打包的基本码流，是将基本的码流 ES 流根据需要分成长度不等的数据包，并加上包头就形成了打包的基本码流 PES 流。封装时不对其进行改变，只在前面添加头部，如私有头，解码时，将私有头剥掉，将原始 ES 码流送进解码器解码。PES 包是由固定包头，可选包头和负载三部分组成，其中固定包头固定 6 个字节；PES 包长度字段占位 16bit，最大值为 65536，因此一帧数据可能被分为多个 PES 包存放。

PTS、DTS
PTS–PresentationTime Stamp（显示时间标记）表示显示单元出现在系统目标解码器（H.264、MJPEG 等）的时间。

DTS–Decoding Time Stamp（解码时间标记）表示将存取单元全部字节从解码缓存器移走的时间。

PTS/DTS 是打在 PES 包的包头里面的，这两个参数是解决音视频同步显示，防止解码器输入缓存上溢或下溢的关键。每一个 I（关键帧）、P（预测帧）、B（双向预测 帧）帧的包头都有一个 PTS 和 DTS，但 PTS 与 DTS 对于 B 帧不一样，无需标出 B 帧的 DTS，对于 I 帧和 P 帧，显示前一定要存储于视频解码器的重新排序缓存器中，经过延迟（重新排序）后再显示，所以一定要分别标明 PTS 和 DTS。

最后，根据 MPEG-2 的规范， 可以在下面最终生成的 2 种格式数据流中选择其一再封装。

### TS 流 (Transport Stream)

也叫传输流，是由固定长度为 188 字节的包组成，含有独立时基的一个或多个 program, 一个 program 又可以包含多个视频、音频、和文字信息的 ES 流; 每个 ES 流会有不同的 PID 标示. 而又为了可以分析这些 ES 流, TS 有一些固定的 PID 用来间隔发送 program 和 ES 流信息的表格: PAT 和 PMT 表。适用于误码较多的环境。既使丢帧. 或者是从任意时段开始（播放），都能准确同步。

### PS 流 (Program Stream)

PS 流有一个结束码 MPEG_program_end_code：占位 32bit，其值为 0x000001B9，PS 流总是以 0x000001BA 开始，以 0x000001B9 结束，对于一个 PS 文件，有且只有一个结束码 0x000001B9，不过对于网传的 PS 流，则应该是没有结束码的。PS 包主要由固定包头，系统头，和 PES 包组成。目前的系统头部好像是没有用到的，所以对于系统头部的解析，我们一般只要先首先判断是否存在系统头（根据系统头的起始码 0x000001BB），然后我们读取系统头的头部长度, 即 header_length 部分，然后根据系统头部的长度，跳过 PS 系统头，进入下一个部分，即 PS 的 payload，PES 包；在固定包头和系统头之后，就是 PS 包的 payload，即 PES 包；若 PSM 存在，则第一个 PES 包即为 PSM。

#### PSM (program_stream_map)

PSM 提供了对 PS 流中的原始流和他们之间的相互关系的描述信息；PSM 是作为一个 PES 分组出现，当 stream_id == 0xBC 时，说明此 PES 包是一个 PSM；PSM 是紧跟在系统头部后面的；PSM 是作为 PS 包的 payload 存在的；

解析 PS 包，要先找到 PS 包的的起始码 0x000001BA 位串，然后解析出系统头部字段，之后进入 PS 包的负载，判断是否有 PSM，根据 PSM 确定 payload 的 PES 包中所负载的 ES 流类型；然后再根据 ES 流类型和 ID 从 PES 包中解析出具体的 ES 流；解包过程则相反；若要从 PS 流中找出来帧类型，必须将 PS 包解析成 ES 并组成完整的帧，然后在帧数据开始根据 NAL 头来进行帧的类型的判断；

PSM 只有在关键帧打包的时候，才会存在；IDR 包含了 SPS,PPS 和 I 帧；每个 IDR NALU 前一般都会包含 SPS、PPS 等 NALU，因此将 SPS、PPS、IDR 的 NALU 封装为一个 PS 包，包括 PS 头，PS system header，PSM，PES；所以一个 IDR NALU PS 包由外到内顺序是：PS header| PS system header | PSM| PES。对于其它非关键帧的 PS 包，就简单多了，直接加上 PS 头和 PES 头就可以了。顺序为：PS header | PES header | h264raw data。以上是对只有视频 video 的情况，如果要把音频 Audio 也打包进 PS 封装，只需将数据加上 PES header 放到视频 PES 后就可以了。顺序如下：PS 包 = PS 头 | PES(video)|PES(audio)；


Reference:

1. https://blog.csdn.net/Guofengpu/article/details/52035557
2. https://blog.csdn.net/appledurian/article/details/73134558
3. http://bbs.ivideostar.com/forum.php?mod=viewthread&tid=156


## 传输协议


### RTP 实时传输协议 (Real-time Transport Protocol)

RTP 协议详细说明了在互联网上传递音频和视频的标准数据包格式。它一开始被设计为一个多播协议，但后来被用在很多单播应用中。RTP 协议常用于流媒体系统（配合 RTSP 协议），视频会议和一键通（Push to Talk）系统（配合 H.323 或 SIP），使它成为 IP 电话产业的技术基础。RTP 协议和 RTP 控制协议 RTCP 一起使用，而且它是创建在 UDP 协议上的。

### RTCP 实时传输控制协议 (RTP Control Protocol)

RTCP 为 RTP 媒体流提供信道外（out-of-band）控制。RTCP 本身并不传输数据，但和 RTP 一起协作将多媒体数据打包和发送。RTCP 定期在流多媒体会话参加者之间传输控制数据。RTCP 收集相关媒体连接的统计信息，例如：传输字节数，传输分组数，丢失分组数，jitter，单向和双向网络延迟等等，网络应用程序即可利用 RTCP 的统计信息来控制传输的品质。RTP 使用一个 偶数 UDP port ；而 RTCP 则使用 RTP 的下一个 port，也就是一个奇数 port。

### RTSP 实时流传输协议 (Real Time Streaming Protocol）

![](/resource/rtsp_flow.webp)

RTSP 对流媒体提供了诸如暂停，快进等控制，而它本身并不传输数据，RTSP 的作用相当于流媒体服务器的远程控制。服务器端可以自行选择使用 TCP 或 UDP 来传送串流内容，它的语法和运作跟 HTTP 1.1 类似，但并不特别强调时间同步，所以比较能容忍网络延迟。应用程序对应的是 play, seek, pause, stop 等命令，RTSP 则是处理这些命令，在 UDP 传输时并使用 RTP(RTCP) 来完成。如果是 TCP 连接则不会使用 RTP(RTCP)。

协商过程

C->S:OPTION request // 询问 S 有哪些方法可用
S->C:OPTION response //S 回应信息中包括提供的所有可用方法

C->S:DESCRIBE request // 要求得到 S 提供的媒体初始化描述信息
S->C:DESCRIBE response //S 回应媒体初始化描述信息，主要是 sdp

C->S:SETUP request // 设置会话的属性，以及传输模式，提醒 S 建立会话
S->C:SETUP response //S 建立会话，返回会话标识符，以及会话相关信息

C->S:PLAY request //C 请求播放
S->C:PLAY response //S 回应该请求的信息

Reference:

1. https://blog.csdn.net/chen495810242/article/details/39207305
2. https://www.jianshu.com/p/4e3925f98e84


### RTMP 实时消息传输协议 (Real Time Messaging Protocol)

该协议基于 TCP，是一个协议族，包括 RTMP 基本协议及 RTMPT/RTMPS/RTMPE 等多种变种。RTMP 是一种设计用来进行实时数据通信的网络协议，主要用来在 Flash/AIR 平台和支持 RTMP 协议的流媒体 / 交互服务器之间进行音视频和数据通信。支持该协议的软件包括 Adobe Media Server/Ultrant Media Server/red5 等。


### HLS (HTTP Live Streaming)

HLS (HTTP Live Streaming)是 Apple 的动态码率自适应技术。主要用于 PC 和 Apple 终端的音视频服务。包括一个 m3u(8)的索引文件，TS 媒体分片文件和 key 加密串文件。新型点播服务器系统，独创了内存缓存数据实时切片技术，不将 TS 切片文件存到磁盘，而是存在内存当中，这种技术使得服务器的磁盘上面不再会有 “数以吨计” 的文件碎片，极大提高了服务器运行的稳定性。同时，由于使用这种技术，使得终端请求数据时直接从服务器的内存中获取，极大提高了对终端数据请求的反应速度，优化了视频观看体验。


## 视频格式

### H.264

H.264，同时也是 MPEG-4 第十部分，是由 ITU-T 视频编码专家组（VCEG）和 ISO/IEC 动态图像专家组（MPEG）联合组成的联合视频组（JVT，Joint Video Team）提出的高度压缩数字视频编解码器标准。

H.264 为解决不同应用中的网络传输的差异。定义了两层：视频编码层（VCL：Video Coding Layer）负责高效的视频内容表示，网络提取层（NAL：Network AbstractionLayer）负责以网络所要求的恰当的方式对数据进行打包和传送(如图所示： 标准的整体框架)。

市面上绝大多数的高清视频均是采用 H.264 的格式编码，它又分为四个最主要步骤，分别是流处理，逆变换，动态补偿，去方块滤波，这四步也是资源消耗的主要四个部分。

#### 概念

H.264 是一次概念的革新，它打破常规，完全没有 I 帧、P 帧、B 帧的概念，也没有 IDR 帧的概念。对于 H.264 中出现的一些概念从大到小排序依次是：序列、图像、片组、片、NALU、宏块、亚宏块、块、像素。这里有几点值得说明：

1. 在 H.264 协议中图像是个集合概念，顶场、底场、帧都可以称为图像（本文图像概念时都是集合概念）。因此我们可以知道，对于 H.264 协议来说，我们平常所熟悉的那些称呼，例如：I 帧、P 帧、B 帧等等，实际上都是我们把图像这个概念具体化和细小化了。我们在 H.264 里提到的 “帧” 通常就是指不分场的图像；
2. 如果不采用 FMO（灵活宏块排序） 机制，则一幅图像只有一个片组；
3. 如果不使用多个片，则一个片组只有一个片；
4. 如果不采用 DP（数据分割）机制，则一个片就是一个 NALU，一个 NALU 也就是一个片。否则，一个片由 三个 NALU 组成（即标准 “表 7-1” 中 nal_unit_type 值为 2、3、4 的三个 NALU 属于一个片）
   - 2:  编码条带数据分割块 A slice_data_partition_a_layer_rbsp()
   - 3:  编码条带数据分割块 B slice_data_partition_b_layer_rbsp( )
   - 4:  编码条带数据分割块 C slice_data_partition_c_layer_rbsp( )

一幅图像由 1～N 个片组组成，而每一个片组又由一个或若干个片组成一个片由一个 NALU 或三个 NALU（假如有数据分割）组成。图像解码过程中总是按照片进行解码，然后按照片组将解码宏块重组成图像。从这种意义上讲，片实际是最大的解码单元。

#### 冗余处理

H.264 与以前的国际标准如 H.263 和 MPEG-4 相比，为达到高效的压缩，充分利用了各种冗余，统计冗余和视觉生理冗余。

1．统计冗余：频谱冗余（指色彩分量之间的相关性），空间冗余，还有时间冗余。这是视频压缩区别于静止图像的根本点，视频压缩主要利用时间冗余来实现大的压缩比。
2．视觉生理冗余
视觉生理冗余是由于人类的视觉系统（HVS）特性造成的，比如人眼对色彩分量的高频分量没有对亮度分量的高频分量敏感，对图像高频（即细节）处的噪声不敏感等。
针对这些冗余，视频压缩算法采用了不同的方法加以利用，但主要的考虑是集中在空间冗余和时间冗余上。H.264 也采用混合 (hybrid) 结构，即对空间冗余和时间冗余分别进行处理。对空间冗余，标准通过变换及量化达到消除的目的，这样编码的帧叫 I 帧；而时间冗余则是通过帧间预测，即运动估计和补偿来去除，这样编码的帧叫 P 帧或 B 帧。与以前标准不同的是，H.264 在编码 I 帧时，采用了帧内预测，然后对预测误差进行编码。这样就充分利用了空间相关性，提高了编码效率。H.264 帧内预测以 16x16 的宏块为基本单位。首先，编码器将与当前宏块同一帧的邻近像素作为参考，产生对当前宏块的预测值，然后对预测残差进行变换与量化，再对变换与量化后的结果做熵编码。熵编码的结果就可以形成码流了。由于在解码器端能够得到的参考数据都是经过反变换与反量化后的重建图像，因此为了使编解码一致，编码器端用于预测的参考数据就和解码器端一样，也是经过反变换与反量化后的重建图像。

Reference:

1. https://blog.csdn.net/zhymxt/article/details/6654891


### FLV (FlashVideo)

FLV 是 FLASH VIDEO 的简称，FLV 流媒体格式是随着 Flash MX 的推出发展而来的视频格式。由于它形成的文件极小、加载速度极快，使得网络观看视频文件成为可能，它的出现有效地解决了视频文件导入 Flash 后，使导出的 SWF 文件体积庞大，不能在网络上很好的使用等问题。FLV 文件体积小巧，清晰的 FLV 视频 1 分钟在 1MB 左右，一部电影在 100MB 左右，是普通视频文件体积的 1/3。Flash 8 Video Encoder 支持转换的视频格式相当广泛，如：AVI、WMV、MPEG、ASF、MOV 等常见视频格式。