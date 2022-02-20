---
layout: post
title:  "探索 Web 视频编解码"
date:   2022-02-20 17:16:37 +0800
categories: tech
---

## 视频编解码
### [What If](https://movie.douban.com/subject/33408073/)... 没有视频编码？

以 1920x1080 的高清视频为例，每一帧大约 8MB，一秒 30 帧大概 240MB，一部两小时电影需要 1.7TB。即便是 Wi-Fi 6 AX3000 路由器也几乎被打满 (2400Mbps > 240*8Mbps)。只有最新的[万兆网络](https://www.ruanyifeng.com/blog/2022/02/10g-ethernet.html) 才能支持。更别说 4K、8K 清晰度。

即便是 360P 视频 (1/9 大小) 也需要至少 250Mbps 带宽，无论是网络传输还是持久化存储都无法接受。

### 为什么视频可以被压缩？
视频信息之所以可被压缩很多，因为其本身存在大量的数据冗余。主要类型有：

1. 时间冗余：视频相邻的两帧之间内容相似，存在运动关系
2. 空间冗余：视频的某一帧内部的相邻像素存在相似性
3. 编码冗余：视频数据出现的概率不同
4. 视觉冗余：人的视觉系统对视频中不同的部分敏感度有差异

针对这些不同类型的冗余信息，在各种视频编码算法中都有不同技术专门应对，以通过不同的角度提高压缩的比率。

### 编 (压) 码(缩)过程
现代视频编码器一般会采用以下几种方法压缩视频数据：

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16453514175614.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


- 预测压缩
  - 帧间预测，去掉时间冗余，采用运动估计与补偿算法
  - 帧内预测，去掉空间冗余，因为相邻像素的亮度和色度一般比较接近，存在相关性
- 变换压缩
  - 去掉视觉冗余，利用人眼对总体亮度的敏感程度远大于细节信息，利用正交变换去掉图片中的细 (高) 节(频)信息，常见的有 DCT(离散余弦变换)，类似还有傅立叶变换等等。
- 熵编码
  - 去掉编码冗余，编码后数据出现的概率更加均等，H.264 采用了 [CABAC](https://www.google.com/search?q=CABAC)，除此之外常见的还有哈夫曼 (Huffman) 等等，** 因此编码后的视频文件使用 gzip 等基于熵编码的压缩算法收效甚微 **。

  
## 发展史
### Timeline
从 1929 年英国的 R.D. Kell 提出用帧间压缩模拟视频至 2020 年 H.266/VVC 多功能视频编码标准确定将近百年的视频压缩发展史，中间有提出过 PCM(脉冲编码调制) 、行程长度编码(AAABBB 存为 3A3B)，再到加入预测、运动估计，最后融合变换压缩(DCT)，现代视频编码融合了多种压缩算法。

从是否免版权税和影响力来看，主要有两种编码：

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16453515263065.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)

**h.26x 系列 **

以 H.264 为例，乃由国际标准化组织 (ISO) 动态图像专家组和国际电信联盟 (ITU) 共同制定的压缩标准，并收录到 ISO 的 MPEG-4 Part 10 中。ISO 称呼其为 AVC(高级视频编码)，ITU 则以 H.264 命名，一般写成 H.264/AVC。

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16453516331879.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)

H.264 专利费由 [MPEG LA](https://zh.wikipedia.org/wiki/MPEG_LA) 代收，一年收取上限总额 500 万美元。而最新的 H.265/HEVC 不仅有 MPEG LA 收取，里面的公司杜比、飞利浦和通用电气等等新成立了一个组织 HEVC Advance，不仅向硬件制造商收取专利费，内容提供商 (Netflix、YouTube) 也得把 0.2% 收入上交，高昂的许可费用是 H.265 迟迟无法普及的主要原因。

> mp4 并非 MPEG-4 的缩写，而是 MPEG-4 Part 14 标准的产物；同理 mp3 则来自于 MPEG-1 Audio Layer III 标准，不是 MPEG-3 的缩写。


** 免版权税系列 **

在 2003 年 H.264/AVC 的第一版被完成。在同一年，一家叫做 TrueMotion 的公司发布了他们的免版税有损视频压缩的视频编解码器，称为 VP3。后被 Google 收购，12 年发布了 VP9，为了抗衡 HEVC，Google 和 Mozilla、Netflix，以及硬件厂商 AMD、Intel，还有网络设备制造商 Cisco 等公司组成了 AOMedia(开放媒体联盟)，于 17 年发布了 AV1。

### 革命性的 AV1
基于 VP9 开源免费的 AV1 按照 Mozilla 的[测试结果](https://blog.mozilla.org/en/mozilla/royalty-free-web-video-codecs/)：

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16453517920385.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


相较 AVC 同等清晰度下减少了 50% 的体积，甚至优于 HEVC。YouTube、Netflix 和 Bilibili 等视频网站也在转向 AV1。AV1 可以让这些内容提供商少缴纳 50% 的流量税。

** 推广阻力 **
Apple 公司 18 年加入了 AOMedia，但至今没有支持硬件解码，使得 AV1 的视频在 macOS 解码效率极低。

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16453521648785.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


如图，在 YouTube 播放 8K HDR 视频导致 CPU 占用暴增至 50%，通过 chrome://media-internals/ 发现

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16453521774231.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)

查阅资料了解到 dav1d 是视频播放软件 [VLC](https://www.videolan.org/projects/dav1d.html) 开发商发布的跨平台 AV1 解码器(软解)。相较于 H.264 的硬解，AV1 劣势非常大：

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16453523161851.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)

如图，从对这四种解码的 [性能统计数据](http://www.rtcbits.com/2021/02/webrtc-video-codecs-performance.html) 来看，在 MacBook Pro (i9) 上因为 H.264 支持硬解，CPU 占用变化不大。而 VP8、VP9 和 AV1 由于软解非常占用 CPU，AV1 更甚。

相对硬解，硬编要严峻得多，可能由于视频解码远大于编码次数，市面上的消费级 CPU/GPU 都不支持 AV1 编码，使得 AV1 编码的时间和硬件成本非常高 **。据说未来至强等企业级处理器会支持 **。

## WebCodecs
现代 Web 已提供 [Media Stream API](https://developer.mozilla.org/en-US/docs/Web/API/MediaStream_Recording_API)、[Media Recording API](https://developer.mozilla.org/en-US/docs/Web/API/MediaStream_Recording_API)、[Media Source API](https://developer.mozilla.org/en-US/docs/Web/API/Media_Source_Extensions_API)、[WebRTC API](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API) 支持音视频需求，但仍缺少帧级别的编解码 API。

许多 Web 音视频编辑器为了解决此问题，使用 WebAssembly 技术把音视频编解码带到浏览器(软解)；但浏览器已在底层内置了音视频的编解码器，还支持硬件加速。这样的情况既浪费开发精力，还增加了运行时计算开销。

因此诞生了 [WebCodecs API](https://wicg.github.io/web-codecs/)，把浏览器已有的能力暴露出来。从解码角度看，终于可以跳出 timeupdate 事件获取到每一帧画面；对编码需求而言，浏览器也能像客户端一样导出视频了。

### 兼容性
目前仅 Chromium 支持，从 [Google Chrome version history](https://en.wikipedia.org/wiki/Google_Chrome_version_history) 可知，11 年发布的 Chrome 10 已支持视频硬件加速。在 [Chrome Platform Status](https://chromestatus.com/feature/5669293909868544) 了解到从 86 版本已支持 [Origin Trial](https://developer.chrome.com/origintrials/#/trials/active) 手动启用此 API，94 开始默认 Enabled。

在 chrome://gpu 页面能看到视频硬件加速支持情况：

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16453525313934.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)

如图，视频编解码均允许硬件加速，搜 "video" 关键字还能找到具体支持的编码标准：

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16453525473348.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)

图中编码仅支持 h264([baseline/main/high](https://en.wikipedia.org/wiki/Advanced_Video_Coding#Profiles))，可能是因为消费者编码场景较少，大部分硬件都不会支持太多格式。

### 处理流程
规范里提供了 VideoEncoder(编码器) 和 VideoDecoder(解码器) 两个主要接口，前者可以传入 Canvas、ImageBitmap、HTMLVideoElement 等 [CanvasImageSource](https://html.spec.whatwg.org/multipage/canvas.html#canvasimagesource) 类型以及 Uint8Array 更底层的图像数据，导出编码后的数据块。后者反之。

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16453525935489.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)

如图，VideoFrame 用于整合各种类型的外部输入，并给到 VideoEncoder，经过编码后的 EncodedVideoChunk 可以拼接后组成视频码流。

### 代码实践

** 功能 **

把 canvas 画面编码成 H.264 流，使用 [ffmpeg.wasm](https://github.com/ffmpegwasm/ffmpeg.wasm) 封装成 mp4 文件，并导出到本地。

** 初始化 VideoEncoder**


```typescript
const init: VideoEncoderConfig = {
  output: handleChunk,
  error: (e) => {
    console.log(e.message);
  },
};
const { supported } = await VideoEncoder.isConfigSupported(config);
let encoder: VideoEncoder;
if (supported) {
  encoder = new VideoEncoder(init);
  encoder.configure(config);
}
```

config 指定了编码器，具体名称可在[编码参数 - MDN](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/codecs_parameter) 找到，还有码率、帧率以及分辨率等信息。
init 接收回调事件，错误消息和编码后的 chunk。

> 截至目前，Typescript 尚未在 [lib.dom.d.ts](https://github.com/microsoft/TypeScript/blob/main/lib/lib.dom.d.ts) 内增加 VideoEncoder。需手动安装 @types/dom-webcodecs。

**接收 chunk**


```typescript
function handleChunk(chunk) {
  const buffer = new ArrayBuffer(chunk.byteLength);
  chunk.copyTo(buffer);
  state.buffers.push(buffer);
}
```

拿到编码好的 chunk 转换成 ArrayBuffer 记录下来，待后续合成使用。

**编码**

```typescript
function createVideoFrame() {
  // ...
  const $canvas = $('#canvas') as HTMLCanvasElement;
  const frame = new VideoFrame($canvas, init);
  return frame;
}

const keyFrame = fc % 150 == 0; // 每隔150帧标记成keyframe
fc++; // 第几帧数
encoder.encode(frame, { keyFrame }); 
```

使用 requestAnimationFrame 或 setInterval 等 API 循环记录编码 canvas 内容。

> 循环可能会受主线程繁忙影响，建议使用 [OffscreenCanvas](https://developer.mozilla.org/en-US/docs/Web/API/OffscreenCanvas) 转到 Worker 线程处理。


**合成导出**


```typescript
async function exportVideo(ffmpeg: FFmpeg) {
  const blob = new Blob(state.buffers);
  const vb = await blob.arrayBuffer();
  const filename = 'test.264';
  const outfile = 'test.mp4';
  ffmpeg.FS('writeFile', filename, new Uint8Array(vb));
  await ffmpeg.run('-framerate', '30', '-i', filename, '-c', 'copy', '/' + outfile);
  const mp4Blob = new Blob([ffmpeg.FS('readFile', '/' + outfile)]);

  downloadBlob(mp4Blob, outfile);
}
```

使用 Blob 把多条 UInt8Array 流拼接成一条，再借助 ffmpeg 把 H.264 封装成 mp4，导出下载至本地。因为封装计算量很小，导出已编码好的帧会显得非常快。

### 其它

**接口设计**

WebCodec 在设计上分成了两层，上层同步接口层，下层异步执行层，再通过 handleChunk 协调两条处理线，很好的避免了异步污染问题。

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16453530189761.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


## 推荐阅读
- [网络视频编解码器指南](https://developer.mozilla.org/zh-CN/docs/Web/Media/Formats/Video_codecs#%E6%BA%90%E8%A7%86%E9%A2%91%E6%A0%BC%E5%BC%8F%E5%AF%B9%E7%BC%96%E7%A0%81%E8%BE%93%E5%87%BA%E7%9A%84%E5%BD%B1%E5%93%8D)
- [《在线视频技术精要》](https://book.douban.com/subject/34947377/)
- [WebCodecs - w3.org](https://www.w3.org/TR/webcodecs/#dictdef-videoencoderconfig)
- [【从零开始】理解视频编解码技术](https://zhuanlan.zhihu.com/p/93398878)