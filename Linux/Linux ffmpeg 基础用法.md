### 简介

`FFmpeg` 是一个强大的开源多媒体框架，用于处理视频、音频和其他多媒体文件和流。它允许转换、录制、编辑、流媒体等等。

### 安装

* `Debian/Ubuntu`

```shell
sudo apt update
sudo apt install ffmpeg
```

* `Red Hat/CentOS`

```shell
sudo dnf install ffmpeg
```

* `macOS (via Homebrew)`

```shell
brew install ffmpeg
```

* 从源码构建

```shell
# Install dependencies
sudo apt update
sudo apt install -y build-essential yasm pkg-config libx264-dev libx265-dev libvpx-dev

# Clone FFmpeg repo and compile
git clone https://git.ffmpeg.org/ffmpeg.git ffmpeg
cd ffmpeg
./configure
make
sudo make install
```

### 常用选项

* `-i`：指定输入文件

* `-f`：指定输出的格式

* `-c:v`：指定视频编解码器

* `-c:a`：指定音频编解码器

* `-b:v`：指定视频比特率

* `-b:a`：指定音频比特率

* `-t`：持续时间 (`hh:mm:ss`)

* `-ss`：开始时间

* `-vn`：禁用视频流

* `-an`：禁用音频流

* `-map`：选择指定的流

* `-y`：无需询问即可覆盖输出文件

### 示例用法

#### 查看 `ffmpeg` 版本

```shell
ffmpeg -version
```

#### 转换视频格式

> 要将视频从一种格式转换为另一种格式（例如，将 `.avi` 转换为 `.mp4`）

```shell
ffmpeg -i input.avi output.mp4
```

#### 从视频中提取音频

> 提取音频并保存为 `mp3` 格式

```shell
ffmpeg -i input.mp4 -q:a 0 -map a output.mp3
```

* `-q:a 0`：设置音频质量（0 为最佳）

* `-map a`：选择音频流

#### 转换音频格式

> 要转换音频文件（例如，将 `.wav` 转换为 `.mp3`）

```shell
ffmpeg -i input.wav output.mp3
```

#### 调整视频大小

> 要将视频调整为特定分辨率（例如 `1280x720`）

```shell
ffmpeg -i input.mp4 -vf "scale=1280:720" output.mp4
```

#### 更改视频编解码器

> 要使用特定编解码器（例如 `H.264` 编解码器）转换视频

```shell
ffmpeg -i input.mp4 -c:v libx264 output.mp4
```

#### 从视频中提取特定时间范围

> 要从视频中提取特定片段（例如从 1 分钟开始的 30 秒）

```shell
ffmpeg -i input.mp4 -ss 00:01:00 -t 00:00:30 -c:v copy -c:a copy output.mp4
```

* `-ss 00:01:00`：开始时间（1分钟）

* `-t 00:00:30`：时长（30 秒）

#### 合并多个视频

> 将多个视频文件合并为一个

1. 先创建一个文本文件，把文件的名称写进去，如下：

```shell
file 'input1.mp4'
file 'input2.mp4'
```

2. 运行命令

```shell
ffmpeg -f concat -safe 0 -i filelist.txt -c copy output.mp4
```

#### 视频添加水印

```shell
ffmpeg -i input.mp4 -i watermark.png -filter_complex "overlay=10:10" output.mp4
```

* `overlay=10:10`：水印的位置（距左上角 `10px` ）

#### 调整视频速度

* 减速（50% 速度）

```shell
ffmpeg -i input.mp4 -filter:v "setpts=2.0*PTS" output.mp4
```

* 加速（200% 速度）

```shell
ffmpeg -i input.mp4 -filter:v "setpts=0.5*PTS" output.mp4
```

#### 从视频中创建 `GIF`

```shell
ffmpeg -i input.mp4 -vf "fps=10,scale=320:-1:flags=lanczos" -c:v gif output.gif
```

#### 将音频转换为单声道

```shell
ffmpeg -i input.mp3 -ac 1 output.mp3
```

#### 将音频转换为立体声

```shell
ffmpeg -i input.mp3 -ac 2 output.mp3
```

#### 将音频的音量增加 2 倍

```shell
ffmpeg -i input.mp3 -filter:a "volume=2.0" output.mp3
```

#### 将音频标准化为标准音量级别

```shell
ffmpeg -i input.mp3 -filter:a "loudnorm" output.mp3
```

#### 通过 `RTMP` 流式传输音频/视频

```shell
ffmpeg -re -i input.mp4 -f flv rtmp://live.twitch.tv/app/stream_key
```

#### 通过 `RTP` 传输音频

```shell
ffmpeg -i input.mp3 -f rtp rtp://192.168.0.100:1234
```

#### 应用视频滤镜

> `FFmpeg` 包含许多过滤器来调整视频属性，如亮度、对比度、饱和度等。

**增加亮度和对比度的示例**

```shell
ffmpeg -i input.mp4 -vf "eq=brightness=0.05:contrast=1.5" output.mp4
```

#### 应用音频滤镜

**降低音频噪音的示例**

```shell
ffmpeg -i input.wav -af "afftdn" output.wav
```

### 总结

`FFmpeg` 是一个非常强大和灵活的多媒体工具，用于转换、编辑和处理音频和视频。它支持广泛的格式和编解码器，使其成为任何媒体相关任务的必备工具。使用`FFmpeg`，可以执行简单的任务，如转换文件和提取音频，以及更高级的任务，如流式传输、创建 `gif` 和编辑视频。广泛的过滤器和选项使其适用于几乎任何媒体相关的需求。