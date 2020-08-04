## 0. Hello World

首先我们来打印一些视频的基本信息，例如它的格式（使用的容器），持续时间，分辨率，音频通道等。然后解码一些帧并保存为图片。

### FFmpeg libav 结构

首先看一下FFmpeg libav是如何运行的，并且不同组件之间如何进行协作。下面是一个视频解码过程的图表：

![视频解码](https://github.com/leandromoreira/ffmpeg-libav-tutorial/raw/master/img/decoding.png)

首先将视频文件加载到一个叫做`AVFormatContext`的组件中，但它并没有加载全部的文件，**只是读取了文件头部**。

加载完文件的头部后，我们就可以获得它的文件流，每个流都被包含在一个叫`AVStream`的组件中。加入我们的视频有两个流：一个是使用[AAC_CODEC](https://en.wikipedia.org/wiki/Advanced_Audio_Coding)编码的音频流，一个是使用[H264 (AVC) CODEC](https://en.wikipedia.org/wiki/H.264/MPEG-4_AVC)编码的视频流。

对于每个流都可以提取其中名为packets的**数据片（pices of data）**，并且加载到一个叫做`AVPacket`的组件中。但是在`AVPacket`组件内的数据仍然是编码过的，并且如果要解码这些`AVPacket`，需要传递一个`AVCodec`。

`AVCodec`会将数据解码为`AVFrame`，而`AVFrame`这个组件会给我们未压缩的帧（frame）。

> 上面的流程对于视频和音频来说都是一样的。

### 代码部分

首先分配内存给`AVFormatContext`，它会存储关于视频格式（容器）的信息：

```c
AVFormatContext *pFormatContext = avformat_alloc_context();
```

然后打开文件并读取里面的头部信息，填充到`AVFormatContext`。使用的函数是`avformat_open_input`。这个函数接收四个参数：`AVFormatContext`、`filename`和两个可选的参数`AVInputFormat`（如果传递NULL则FFMpeg会自动推断）、`AVDictionary`。

```c
avformat_open_input(&pFormatContext, filename, NULL, NULL);
```

读取完毕后，可以得到或打印视频的信息：

```c
printf("Format %s, duration %lld us", pFormatContext->iformat->long_name, pFormatContext->duration);
```

为了得到真正的信息流`streams`，需要从文件内读取数据。函数`avformat_find_stream_info`将读取的流数据田中到`AVFormatContext`内。执行完成后，就可以通过`pFormatContext->streams[i]`来访问每个流，流的数量则是`pFormatContext->nb_streams`。

```c
avformat_find_stream_info(pFormatContext,  NULL);
```

然后我们开始循环地对每个流进行处理，主要是得到对应的`AVCodecParameters`，它描述了每个流的编码格式：

```c
for (int i = 0; i < pFormatContext->nb_streams; i++)
{
  AVCodecParameters *pLocalCodecParameters = pFormatContext->streams[i]->codecpar;
}
```

得到了每个流的一些属性，就可以通过函数`avcodec_find_decoder`从注册的解码器中找到对应的解码器。最后返回一个`AVCodec`的组件，它能知道如何编码和解码这个流。

```c
AVCodec *pLocalCodec = avcodec_find_decoder(pLocalCodecParameters->codec_id);
```

这样我们就可以得到有关此解码器的信息：

```c
// 判断是视频还是音频
if (pLocalCodecParameters->codec_type == AVMEDIA_TYPE_VIDEO) {
  printf("Video Codec: resolution %d x %d", pLocalCodecParameters->width, pLocalCodecParameters->height);
} else if (pLocalCodecParameters->codec_type == AVMEDIA_TYPE_AUDIO) {
  printf("Audio Codec: %d channels, sample rate %d", pLocalCodecParameters->channels, pLocalCodecParameters->sample_rate);
}
// general
printf("\tCodec %s ID %d bit_rate %lld", pLocalCodec->long_name, pLocalCodec->id, pCodecParameters->bit_rate);
```

得到了解码器我们就可以分配内存给`AVCoderContext`，它会保存我们编码/解码的上下文。还需要用**CODEC**参数来填充它，使用的函数是`avcodec_parameters_to_context`。填充完毕后需要使用函数`avcodec_open2`打开此解码器，这样我们就可以使用它了。

```c
AVCodecContext *pCodecContext = avcodec_alloc_context3(pCodec);
avcodec_parameters_to_context(pCodecContext, pCodecParameters);
avcodec_open2(pCodecContext, pCodec, NULL);
```

然后从流中读取packets并将它们解码称为frames。但我们需要事先为两个组件分配内存：`AVPacket`和`AVFrame`。

```c
AVPacket *pPacket = av_packet_alloc();
AVFrame *pFrame = av_frame_alloc();
```

接下来使用函数`av_read_frame`将流提取到packets内。

```c
while (av_read_frame(pFormatContext, pPacket) >= 0) {
  //...
}
```

