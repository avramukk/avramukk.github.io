+++
author = "Mykola Avramuk"
title = "How to test videostream?"
date = "2023-09-01"
tags = [
    "streaming",
    "qa",
]
+++
Hey there! I'm Mykola Avramuk, a QA and Streaming Engineer at [Mixa.live](https://mixa.live/).

My mission is to deliver a quality product to users in a timely manner within available resources.

I want to share my experience with testing live video streaming and seek advice from the community. An experienced QA once told me, "_Write about what you would have wanted to read when you started_."

When we read articles about testing streaming video, we often consider PSNR, VMAF, and SSIM. It is best when the system is stable and working as expected, so we can start testing [objective metrics](https://jina-liu.medium.com/a-case-study-for-content-adaptive-encoding-579ee62c1634) such as VMAF.

Or when you are already are Netflix:

[Netflix Video Quality at Scale with Cosmos Microservices](https://netflixtechblog.com/netflix-video-quality-at-scale-with-cosmos-microservices-552be631c113)

[Toward a Better Quality Metric for the Video Community](https://netflixtechblog.com/toward-a-better-quality-metric-for-the-video-community-7ed94e752a30)

But no one tells what happens before this stage. Here, I will explain this stage from the perspective of a Quality Engineer.

_Disclaimer:_

_This material provides a brief overview without delving into specifics. Detailed information will be covered in subsequent articles._

---

# Strategy

## Architecture

Before testing anything, it is crucial to understand the product's purpose, structure, components, and their interactions. This information will guide your testing decisions.

Start planning QA as early as possible. _Shift left_ is an ideal approach.

Most streaming services have a transcoding server or a cluster of such servers at their core. Each server runs a program with encoders, decoders, multiplexers, filters, and other modules for specific tasks.

The streaming video process typically involves these stages:

1. **Encoding**: Convert audio and video sources into a digital format using an encoder. Analog or camera signals are transformed into efficient formats like H.264 or H.265 (AVC and HEVC), which store data at low rates.

2. **Containerization**: Package encoded video and audio data into a container format. MPEG-TS is suitable for traditional telecommunications, while MP4, FLV, WebM, and specialized containers are used for different streaming protocols.

3. **Adaptive or Single Bitrate Streaming**: Choose adaptive or single bitrate streaming. Adaptive strategy encodes video in various resolutions and bitrates, with the player selecting the most suitable stream based on network conditions and device information. Single bitrate streaming transmits a specific stream.

4. **Transport**: Transmit the encoded and packaged video data over the network. Different transport mechanisms are used based on the chosen protocol, such as RTMP, SRT, HLS, MPEG-DASH, and WebRTC.

5. **Network Transmission**: Transmit the video data to servers located in different regions or on content delivery network (CDN) servers, ensuring optimal speed and availability for viewers worldwide.

6. **Decoding and Playback**: Decode and playback the video data on the viewer's side. The player selects the appropriate decoder based on the container format.

Ready decoded stream we will try to receive, and verify these parameters.

First, determine if the program can be isolated for accurate verification. Isolating the program saves time in identifying issues. If isolation is not possible, use the available environment, but it's preferable for the test environment to mirror the production environment.

Identify the operating system on which the application runs to prepare logging tools and collect artifacts for faster issue reproduction and isolation.

## What to test?

When planning what you will test, you need to explore the requirements and ask [key questions](https://testing.googleblog.com/2016/06/the-inquiry-method-for-test-planning.html) about the system's features and what is expected from it.

Streaming video is transmitted using transmission protocol. These are a set of rules that define how video and audio are transmitted over the network. For example, the most common ones are RTMP, SRT, HLS, MPEG-DASH, and WebRTC.

Most systems support multiple [transmission protocols](https://www.wowza.com/blog/streaming-protocols), so each one needs to be tested separately.

Any transmission starts with establishing a connection. Therefore, it is necessary to test the correct establishment of connections based on the implementations of different protocols. All of them use a client-server connection. WebRTC can be client-client as well.

It is necessary to check negative scenarios when the connection should not be established and various cases related to access control.  
  
Вetailed analysis is only possible if part of a transport stream is recorded so that it can be picked apart later. This technique is known as deferred-time testing.

To test the stream, we need to receive, decode, and store it in a container. Let's take SRT as an example. To check the connection establishment, you can use Wireshark to capture all the traffic and analyze both the connection establishment and subsequent transmission. Here is a good explanation of how to do it:

We receive this stream with an available decoder and store it in a container. You can read a nice guide about containers [here](https://bitmovin.com/container-formats-fun-1/).

We can proceed to check the parameters of the transport stream. This includes video, audio, subtitles, and metadata.

To verify that your encoder is working according to the requirements, you will encode the stream, transmit it, and receive it as a user to check the stream parameters and compare them with what you specified on the encoder.

For example, let's review these parameters of the MPEG TS stream received using [SRT](https://youtu.be/VrE3dJej5IE):

### Video:

- **[PID](https://en.wikipedia.org/wiki/MPEG_transport_stream#:~:text=of%20the%20payload.-,Packet%20identifier%20(PID),-%5Bedit%5D)**: used in MPEG-2 transport streams, which are typically used for transmitting video and audio in networks and broadcasting systems. It is used to identify different types of data such as video, audio, subtitles, etc., and helps ensure their proper transmission and synchronization. I have encountered systems that expected video and audio on specific PID numbers, so at a minimum, you should check which numbers are assigned to your streams. Check several PIDs with different content.
    
- **[Codec](https://medium.com/@ceciliadigiarty/understanding-of-video-codec-and-video-container-format-16c6bd353c9d)**: Hardware or software that compresses and decompresses digital video, to make file sizes smaller and storage and distribution of the videos easier. Check only those supported by your encoder. The most popular ones currently are H.264, H.265, VP9, AV1, MPEG-4. Note that some codecs may require a lot of resources, so it is even worse if multithreading is not supported or not configured, and all the load falls on one core, which will be overloaded by more than 70-80%. In this case, the server may throttle the stream may become corrupted, and the tests will be unreliable.
    
- **[Profile](https://en.wikipedia.org/wiki/Advanced_Video_Coding#Profiles:~:text=%D0%B2%20%D0%BD%D0%B0%D1%81%D1%82%D1%83%D0%BF%D0%BD%D0%BE%D0%BC%D1%83%20%D1%80%D0%BE%D0%B7%D0%B4%D1%96%D0%BB%D1%96.-,%D0%9F%D1%80%D0%BE%D1%84%D1%96%D0%BB%D1%96,-%5B%5D)**: They are declared using a profile code (profile_idc), and sometimes a set of additional constraints applied in the encoder. The profile code and specified constraints allow the decoder to recognize the decoding requirements of this specific bitstream.
    
- **[Level](https://en.wikipedia.org/wiki/Advanced_Video_Coding#Levels:~:text=%D0%A2%D0%B0%D0%BA-,%D0%A0%D1%96%D0%B2%D0%BD%D1%96,-%5B%5D)**: It is a set of constraints indicating the required performance level of the decoder for the profile. For example, the support level in the profile determines the maximum resolution, frame rate, and bit rate that the decoder can use. A decoder that complies with a specific level must be able to decode all bitstreams encoded for that level and all lower levels.
    
- **[Chroma format](https://en.wikipedia.org/wiki/Chroma_subsampling#Types_of_sampling_and_subsampling:~:text=External%20links-,Chroma%20subsampling,-14%20languages)**: the practice of encoding images by implementing a lower resolution for chroma information than for luma information, taking advantage of the lower acuity of the human visual system for color differences than for brightness.
    
- **[Resolution](https://www.viewsonic.com/library/tech/monitor-resolution-aspect-ratio/)**: The frame sizes in your stream. One of the mainstream parameters. The quality of your video will be better with increasing resolution. But the higher it is, the higher the required bitrate. And don't forget to consider the system load for 2, 4, 8K. Recommended bitrates for different resolutions:
    
    - SD (Standard Definition): 480p: Around 500 Kbps to 1.5 Mbps
        
    - HD (High Definition):
        
        - 720p: Around 1.5 Mbps to 4 Mbps
            
        - 1080p: Around 3 Mbps to 6 Mbps
            
    - Full HD: 1080p: Around 3 Mbps to 6 Mbps
        
    - 2K: 1440p: Around 6 Mbps to 10 Mbps
        
    - 4K: 2160p: Around 12 Mbps to 25 Mbps
        
    - 8K: 4320p: Can vary significantly, but generally upwards of 30 Mbps
        
- **[Bitrate](https://videocompressionguru.medium.com/what-is-bitrate-what-is-the-difference-between-cbr-and-vbr-f89a3241012d)**: the amount of data transmitted per unit of time during video playback or transmission. It is measured in bits per second (bps). Play the received videos and assess the quality of each bitrate variant. This can be done using visual assessment or objective quality metrics such as PSNR (Peak Signal-to-Noise Ratio) or SSIM (Structural Similarity Index).
    

It can be **CBR (Constant Bit Rate)**: CBR means that the bit rate remains constant throughout the entire video. Regardless of the content complexity, the same amount of data is used at all encoding stages.



![.](https://substackcdn.com/image/fetch/w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F8cd33f46-27b2-432a-98d0-17143ba7668e_3226x1948.png)



[](https://substackcdn.com/image/fetch/f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F8cd33f46-27b2-432a-98d0-17143ba7668e_3226x1948.png)

**VBR (Variable Bit Rate)**: VBR means that the bit rate varies depending on the complexity of the content. The more complex the scene, the more data needs to be transmitted.


![](https://substackcdn.com/image/fetch/w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F773bda96-af99-4009-975d-82539a89371b_2812x1948.png)



[](https://substackcdn.com/image/fetch/f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F773bda96-af99-4009-975d-82539a89371b_2812x1948.png)

- **Framerate**: the rate at which video frames are transmitted. There are 24, 25, 30, 50, 60, and 120 frames per second.
    

You can check the framerate using tools such as ffprobe, mediainfo, or VLC (although I do not recommend VLC as it can sometimes be misleading). To verify the framerate, it is advisable to use high-quality input data so that you can see the number of each frame and ensure that no frames are lost by reviewing each frame.


![](https://substackcdn.com/image/fetch/w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Ffce17feb-9f6a-4c76-9fdc-8f7b55b50c80_800x430.gif)



[](https://substackcdn.com/image/fetch/f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Ffce17feb-9f6a-4c76-9fdc-8f7b55b50c80_800x430.gif)

Frame rate and bitrate can have either **CFR (Constant Frame Rate)**: In this mode, each frame is encoded at a consistent rate, and the entire video uses the same number of frames per second. **VFR (Variable Frame Rate)**: In this mode, the frame rate can vary depending on the complexity of the content. This can be useful for situations where certain parts of the video are less complex and can use fewer frames per second to save data, while more complex scenes can have more frames to ensure better quality.

- **[GOP](https://videocompressionguru.medium.com/group-of-pictures-and-its-structure-8d9c4ea20852)** [(Group of Pictures)](https://videocompressionguru.medium.com/group-of-pictures-and-its-structure-8d9c4ea20852): In video coding, GOP is a sequence of video frames that includes a key frame (I-frame), one or more predicted frames (P-frames), and bidirectional frames (B-frames) that use information from previous or future frames for data compression.
    
    
![](https://substackcdn.com/image/fetch/w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F59d6d04d-83ec-488a-876c-e7b3590f55c6_870x490.png)

- **I-frame (keyframe)**: This is a frame that is independent of other frames in the GOP. It contains complete information about the video scene and can be decoded on its own. Since I-frames are not dependent on other frames, they usually have a larger size than other frame types.
    
- **P-frame (predicted frame)**: These frames are predicted based on I-frames and other P-frames in previous GOPs. They only contain changes relative to previous frames, which allows them to be stored with less data.
    
- **B-frame (between I and P frames)**: These frames contain information based on two other frames - I and P. They can further reduce the data size.
    

B-frames are usually not used in low-latency video streaming for the following reasons:

1. Computational delay: Video codecs require more computations to decode B-frames since they need to be constructed based on other frames. This can result in increased overall video stream delay.

2. Transmission delay: Using B-frames can increase the amount of data that needs to be transmitted for video playback. This can affect network bandwidth and increase data transmission delay.

3. Interpolation quality: Using B-frames requires interpolation between reference frames (I or P) to construct B-frames. This can result in some level of additional artifacts or decreased playback quality.

4. Possible loss of synchronization: In case of a lost B-frame, there may be an issue with decoding subsequent frames that may depend on it. This can lead to degraded playback quality or delays in the video stream.

Test whether your encoder correctly sets the GOP structure. Prepare different test data variations with different GOP sizes and contents.

The GOP structure in the stream can be checked with special tools.

![](https://substackcdn.com/image/fetch/w_1456,c_limit,f_auto,q_auto:good,fl_lossy/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F521ebb18-5729-45ea-a141-32548731453c_800x520.gif)



- **Keyframe interval**: shows the distance between key I frames. It is specified either in a number of frames or in seconds. Different test data can be prepared, starting from a video where all frames are keyframes and up to 10 seconds between keyframes. We observe how an encoder that encodes such distances and a decoder that decodes it will behave. We evaluate the image quality at different keyframe intervals.
    
- **[Color space](https://trac.ffmpeg.org/wiki/colorspace)**: The color space describes how an array of pixel values should be displayed on a screen. It provides information such as how pixel values are stored in a file, as well as the range and values of these values. We check whether the decoder correctly understands the encoded video at different values, for example:
    

- [BT.601]("[Standard Definition](https://en.wikipedia.org/wiki/Rec._601)" or SD)

- [BT.709]("[High Definition](https://en.wikipedia.org/wiki/Rec._709)" or HD)

- [BT.2020]("[Ultra-High-Definition](https://en.wikipedia.org/wiki/Rec._2020)" or UHD)

- **Bit depth**: Bit depth in video refers to the number of bits used to represent the color information of each pixel in an image or frame. Common bit depths used in video are 8-bit, 10-bit, and 12-bit. We check each value.
    
![](https://substackcdn.com/image/fetch/w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fa52167c8-bb56-445f-9b83-ce7cbbae1886_1280x720.png)



- Scan type: Progressive and interlaced are two different approaches to displaying an image on a screen, particularly in video.
    

1. Progressive scanning: In this mode, each frame of the video is displayed in its entirety, and all lines of the image are processed at once. All pixels on the screen are updated immediately, making the image sharper and smoother. Videos in formats such as 720p, 1080p, 4K, etc. use progressive scanning.

2. Interlaced scanning: In this mode, the frame is divided into two halves - even and odd lines. First, all even lines are displayed, and then the odd lines. This was invented to reduce the amount of information that needs to be transmitted through limited communication channels.

![](https://substackcdn.com/image/fetch/w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F8a2f1b75-a671-464a-9dd0-8ffcb1750f6a_2445x1180.png)



- **Aspect ratio**: the ratio of the width to the height of a video frame. Your system should correctly understand each type and add black bars when necessary or pass only the useful payload according to your aspect ratio. We are checking popular values and those specified in the requirements.
    



![](https://substackcdn.com/image/fetch/w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F223aa500-d67d-4fba-9c91-bec3891dd2e0_656x360.png)


- **A/V synchronization**: synchronization of audio and video. Known as lipsync. We prepare corresponding content in which it is easy to understand the delay between sound and image.

![](https://substackcdn.com/image/fetch/w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F903a7c1e-e2b7-40c9-9c00-44bc4483702a_800x454.gif)



[](https://substackcdn.com/image/fetch/f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F903a7c1e-e2b7-40c9-9c00-44bc4483702a_800x454.gif)

- **NTP sync**: If your service supports timecode-based stream synchronization, you need to verify that it is working correctly. NTP works on synchronizing your streams through video encoders and decoders. NTP stands for Network Time Protocol, an internet standard that functions by synchronizing your servers and devices with Coordinated Universal Time (UTC). NTP works by synchronizing your devices (in this case, encoders and decoders) with an NTP server, which in turn synchronizes with a "grandmaster" clock (often an atomic clock or GPS clock). NTP is accurate to tens of milliseconds over the internet and can be accurate to sub-millisecond levels under ideal conditions. And this accuracy is from the grandmaster clock to your device. Therefore, your devices (all of which should be connected to the same NTP server) will be very close to each other in terms of time accuracy.
    
- **PTS/DTS**: **PTS (Presentation Time Stamp)** indicates the time when a specific frame or audio fragment should be displayed on the screen or played on the audio output. **DTS (Decoding Time Stamp)** indicates the time when a given frame or audio fragment should be decoded. Here is a good description of [how to check the correctness of these timestamps in a video stream](https://www.elecard.com/page/timestamp_validation_in_transport_stream).
    
- **Latency**: Latency is the delay between the moment the video signal is created at the source and the moment it is played back at the end user's side.
    



![](https://substackcdn.com/image/fetch/w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Ffc341aae-c50e-410e-a4de-b8c9883e8347_700x380.png)



[](https://substackcdn.com/image/fetch/f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Ffc341aae-c50e-410e-a4de-b8c9883e8347_700x380.png)

As in any other measurement methodologies, there are more or less accurate ways to do this. The usual method that many people use is to place a high-precision digital stopwatch in front of the camera and take a photograph while the stopwatch is already decoding the stream on the screen. But if this is not enough for you, there are [scientific papers](https://www.researchgate.net/publication/307516190_A_system_for_high_precision_glass-to-glass_delay_measurements_in_video_communication) where more accurate measurements can be made.



![](https://substackcdn.com/image/fetch/w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F79ac6c51-3f63-4832-9056-d3f34a3e8039_2326x510.png)

## Audio:

- **PID**: Check that PIDs have the expected identification numbers and that they are reproduced correctly.
    
- **Codec**: Like video, audio has its own codecs AAC, MP3, AC3, DTS, as well as uncompressed PCM audio, which is used for accurate reproduction of the audio signal without data compression.
    
- **Bitrate**: 96, 128, 192, 256 check that the encoder correctly encodes the values and that the sound passes without distortion.
    
- **Channel** **layout**: Check different sound channel schemes such as mono, stereo (2.0), 2.1, 5.1, 7.1, 7.1.2, and 9.1. Prepare a variety of combinations of test data to verify that the transcoder correctly processes them.
    
- **Sample rate**: The sample rate is a parameter that determines how many times per second data is collected from analog audio to convert it into a digital format. This is one of the key characteristics of digital audio. The sample rate is measured in hertz (Hz) and indicates the number of audio samples recorded per second. The standard sample rate is 44100 Hz or 48000 Hz.
    
- **Bit depth**: The bit depth is a parameter that determines the accuracy of storing or transmitting audio information in digital format. It is measured in bits and indicates how many different values can be represented for each audio sample. The higher the bit depth, the more possible values can be recorded for each sound sample, resulting in greater accuracy and sound detail. Check common values: 8, 16, 24, 32 bits.
    

Use test data of different content: human voice, music, noise, and different levels of audio tone. Also, check peak and clipping (>0dB) values.

### Metadata:  


![](https://substackcdn.com/image/fetch/w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F9a7e1c7a-ac2d-4fd6-b586-01aba53b2e6d_3670x2022.png)

You should check the metadata of the transport stream, and for this, you need to at least understand what it is:

1. **Program Association Table (PAT)**: This table provides information about the available programs in the MPEG-TS stream. It includes the Program Map Table (PMT) identifier for each program, allowing receivers to find the corresponding PMT for additional information.
    
2. **Program Map Table (PMT)**: PMT provides details about the components of a specific program, such as audio streams, video streams, subtitles, and other data. Each component is identified by a Packet Identifier (PID), and PMT helps receivers understand how to handle and decode the different components.
    
3. **Service Information (SI)**: SI metadata contains information about services, such as program names, descriptions, and information about service providers. This metadata helps users determine the content they want to view.
    
4. **Conditional Access Table (CAT)**: This table contains information related to conditional access systems that control the encryption and decryption of content for authorized viewers.
    
5. **Event Information Table (EIT)**: EIT metadata provides details about scheduled events, including start time, duration, and program names. They are particularly useful for Electronic Program Guides (EPG), which help users navigate through available content.
    
6. **Network Information Table (NIT)**: NIT metadata contains information about the network itself, such as configuration parameters, frequency, and modulation details. This is especially important for receivers to properly tune to desired channels.
    
7. **Time and Date Table (TDT)**: TDT metadata transmits current time and date information in the MPEG-TS stream. They help ensure synchronization between the transmitter and receiver.
    
8. **Descriptors**: Descriptors are small pieces of metadata that provide additional information about the content or components in the MPEG-TS stream. They can contain details such as audio and video encoding formats, language information, and more.
    

Some major streaming platforms, such as YouTube, will inform you that if you want certification from us, please include the correct metadata in each of your streams (such as the name of the company or encoder), so that we can easily identify who it is if any problems arise.

### Real-Time Statistic

Since we took the SRT streaming protocol as an example, it is necessary to note that SRT provides a powerful set of statistical data on a socket. This data can be used to keep an eye on a socket's health and track faulty behavior. Statistics are calculated independently on each side (receiver and sender) and are not exchanged between peers unless explicitly stated. Our developers have created a player for testing that can display these statistics in real-time and you can observe the values of many parameters. Soon, we plan to make it open source. But let's talk about testing SRT another time.


![](https://substackcdn.com/image/fetch/w_1456,c_limit,f_auto,q_auto:good,fl_lossy/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F90323a19-f1aa-4e4e-94b4-84df0803fe24_800x498.gif)


## Test Environment:

**Internet connection**: Before streaming testing, make sure you have a stable internet connection with high bandwidth capacity. It is best to use an Ethernet connection with 1 Gbps. And only use Wi-Fi for testing different bandwidth scenarios (to simulate a typical user) or tools that allow flexible configuration of bandwidth, latency, delay, lost packets, etc.

You can use tools like Netflix's [Fast](https://fast.com/) or [Speedtest](https://www.speedtest.net/) or [Twitch's tool](https://r1ch.net/projects/twitchtest) to check your network.

## Test Data:

Prepare a lot of test data. Here, you need to find a golden mean for all the parameters described above, depending on which ones are used more frequently. It is good to have recorded test videos, as well as live ones, such as a stream from your camera or a hardware encoder. Choose both complex and simple test content, as this will help simulate different testing conditions.


![](https://substackcdn.com/image/fetch/w_1456,c_limit,f_auto,q_auto:good,fl_lossy/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F9f6593a6-c960-4410-8c59-3a1fb849c79e_800x464.gif)


Here are some links to open test videos:

**streams**:

- https://ottverse.com/free-hls-m3u8-test-urls/
    
- https://github.com/bengarney/list-of-streams
    

-

**files**:

- [https://mango.blender.org/download/](https://mango.blender.org/download/)
    
- [http://download.tsi.telecom-paristech.fr/gpac/dataset/dash/uhd/](http://download.tsi.telecom-paristech.fr/gpac/dataset/dash/uhd/)
    
- [https://medialab.sjtu.edu.cn/tag/dataset/](https://medialab.sjtu.edu.cn/tag/dataset/)
    
- [https://ultravideo.fi/#testsequences](https://ultravideo.fi/#testsequences)
    
- [https://www.murideo.com/test-pattern-library.html](https://www.murideo.com/test-pattern-library.html)
    

## Types of testing:

### Unit testing

At the first stage, pray to God that the developer writes some unit tests and checks the smallest parts of the system.

### Component testing

This is the first level of testing, where individual components of your system are checked for proper functioning. It would be great if you could separately test how the encoder, decoder, multiplexers, filters, etc. work.

### Integration testing

**Component Integration Testing**: Here, you can start designing cases where your modules could interact with each other and simulate cases where someone fails to perform their task as expected.

**System Integration Testing**: At this level, you check the integration of the system as a whole. This includes checking the interaction between different components, servers, clients, and other parts of the system.

**Hardware Software Integration Testing**: This type of testing focuses on checking the interaction between hardware and software components of the system.

**Software Integration Testing**: Here, you focus on the integration of software components, such as programs, libraries, services, and other software modules used for streaming.

### **System Testing**

At this stage, you already have the entire system, and you check if it meets the specifications, works correctly, and meets user requirements.

After this, you can move on to:

- Stability
    
- Load
    
- Stress
    
- Failover and Recovery
    
- Scalability
    
- Volume
    
- Reliability
    

But let's talk about these types later, because I will never finish this article.

## Tools:

Make sure to prepare the tools you will use for the conversion, transmission, reception, and analysis of streams. Among them, there are open-source or trial versions that will be sufficient.

Here are the ones I use:

- ffmpeg
    
- ffprobe
    
- mediainfo
    
- ffplay
    
- ffmetrics
    
- ffbitrate
    
- Elecard:
    
    - Stream Eye
        
    - Quality Estimator
        
    - Stream Analyzer
        
    - Quality Gates
        
- HandBrake
    
- VidCoder
    
- VQProbe
    
- NetBalancer
    
- NetLimiter
    
- OBS
    
- vMix
    
- VLC
    
- [SRTStressTool](https://srtminiserver.com/help/stress_test_tool.html)
    
- [https://thumb.co.il](https://thumb.co.il/)
    
- [https://dvbsnoop.sourceforge.net](https://dvbsnoop.sourceforge.net/)
    
- [https://github.com/tsduck/tsduck](https://github.com/tsduck/tsduck)
    
- [https://www.digitalekabeltelevisie.nl/dvb_inspector](https://www.digitalekabeltelevisie.nl/dvb_inspector/)
    

### Reporting:

It is important to reinforce all the bugs you find with steps, logs, test data, screenshots, and videos. Then your developers will be happy and you won't be bombarded with requests for help.

## Conclusions:

1. To deliver a high-quality product to the user, you need to determine what exactly you are doing and create a well-thought-out testing strategy based on that, which can prevent and mitigate most problems.

2. Start testing before implementing the functionality (shift-left), as testing requires a lot of resources (especially time).

3. Learn and understand how your system works, what components it consists of, and how they interact.

4. Prioritize correctly depending on the usage of your product.

5. Be an attentive observer.