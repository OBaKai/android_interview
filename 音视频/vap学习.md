## Vap

### vap特色

#### vap与svga相比优缺点

```java
svga：序列帧来实现礼物动画效果
  优点：
  	① 小动画的播放效率比较高（播放原理就是在canvas上对svga图像进行对应的位移、形变等操作）
    ② 兼容性好（不涉及到硬件解码）
  缺点：在大动画上：文件体积大（无法很好地压缩数据）；占用内存大（解析之后是大量的帧信息以及svg图像bitmap）

vap：
  优点：mp4文件体积小（h264编码）；解码效率高（硬解码，有专门的硬件做这个事情）;
  缺点：由于是硬解码，依赖硬件可能存在兼容性问题（有些设备不支持硬编码器）。
```



#### vap如果让mp4格式支持透明通道

```java
https://zhuanlan.zhihu.com/p/356719736?ivk_sa=1024320u

mp4是不支持透明的：
图像透明使用Alpha通道表示，即ARGB里的A，该通道是一个8位灰度通道，由256级灰度来记录图像中的透明信息。
由于mp4是用h264编码的，颜色空间使用的是YUV，而YUV只能转为RGB，无法转ARGB，就导致了mp4是不支持透明的。
  
但是礼物动效大部分都是透明背景的，所有如何用mp4，就必须支持透明。
  
vap的做法：
使用两个视频合二为一。其中一个视频在RGB通道里存储Alpha的值，另外一个视频保留原始的RGB信息。
解码的时候，利用OpenGL将两个视频数据提取，将这些数据合成为ARGB图像（带透明通道的图像）
```



### 基本概念

```java
//使用：
animView.setFetchResource(object : IFetchResource { //填充融合图片或文字
            override fun fetchImage(resource: Resource, result: (Bitmap?) -> Unit) {
                val srcTag = resource.tag
                if (srcTag == "[sImg1]") {
                    val drawableId = if (head1Img) R.drawable.head1 else R.drawable.head2
                    head1Img = !head1Img
                    val options = BitmapFactory.Options()
                    options.inScaled = false
                    result(BitmapFactory.decodeResource(resources, drawableId, options))
                } else {
                    result(null)
                }
            }
            override fun fetchText(resource: Resource, result: (String?) -> Unit) {  ... }
  
            override fun releaseResource(resources: List<Resource>) { ... }
        })
animView.startPlay(file) //设置vap文件，并开始播放
  
//加载流程概要：
AnimView#startPlay 
  -> AnimPlayer#startPlay
  	 ① 初始化HardDecoder、AudioPlayer。HardDecoder用于视频解码，AudioPlayer用于音频解码
     ② 初始化Decoder里边的两个线程（都是HandlerThread，一个用来渲染，一个用来解码）
     ③ 读取vap文件头信息，校验vap文件合法性 以及 提取写在文件里边的配置信息（json格式）
     ④ 开始解码（AnimPlayer#innerStartPlay）
  -> AnimPlayer#innerStartPlay
  	 decoder?.start(fileContainer) //视频解码
     audioPlayer?.start(fileContainer) //音频解码 
```



#### 注意点：解析vap配置信息过程，可能存在OOM风险。

```java
//AnimConfigManager#parse 函数中，作者也有提醒
val vapcBuf = ByteArray(head.length - 8) // ps: OOM exception
    
如果vap动效涉及的融合元素较多，并且融合元素的参与帧较长，会导致该json配置信息非常的大。
原因是vap json配置信息除了保存视频相关的信息外，如果有融合元素，会记录融合元素的key、类型等信息，并且还会记录该元素在每个关键帧中的位置信息，导致json会非常的大。
```



### 视频解码流程 - HardDecoder.kt

```java
//HardDecoder#startPlay
① 创建MediaExtractor（提取器），并绑定提取的目标文件
② 提取器寻找视频轨道下标，选中轨道，提取该视频轨道的格式信息
③ 创建一个名叫 glTexture 的 SurfaceTexture，并且设置监听 setOnFrameAvailableListener
④ 创建MediaCodec（解码器），给解码器绑定上 Surface(glTexture)，启动解码器
⑤ 开始解码视频
  while(true){
    //提取器 -> input队列
    ① 从input队列中申请有效的缓冲区下标 index = decoder.dequeueInputBuffer(TIMEOUT_USEC)
    ② 通过下标拿到对应的缓冲区 ByteBuffer buffer = decoder.getInputBuffers()[index]
    ③ 提取器读取样本数据到缓存区 size = extractor.readSampleData(buffer, 0)
    ④ 将填满数据的缓冲区加入到input队列 decoder.queueInputBuffer(index, 0, size, extractor.getSampleTime(), 0);

    //解码过程由MediaCodec完成，它就是从input队列拿到数据，解码之后，放到output队列
    ⑤ 处理output队列，提取output缓冲区信息 decoderStatus = decoder.dequeueOutputBuffer(bufferInfo, TIMEOUT_USEC)
    ⑥ 更新当前帧下标，frameIndex 应该是记录i帧的index
    ⑦ 检查是否解码完成，如何解码完成，判断是否loop（有可能这个vap不止播放一次）
    ...
	}
⑥ 解码后的数据都会去到glTexture，当有新的图像buffer到来，SurfaceTexture就会回调 OnFrameAvailableListener#onFrameAvailable 通知业务层。
⑦ 在 onFrameAvailable 中做了三个操作：
   ① 调用 updateTexImage 将新的图像buffer更新到纹理。
   ② PluginManager#onRendering 通知动画插件（MaskAnimPlugin、MixAnimPlugin），让插件们处理图像。
   ③ Render#swapBuffers() 最后将处理好的图像，展示到 TextureView 的 SurfaceTexture。
```



### 音频解码流程 - AudioPlayer.kt

```java
AudioTrack是播放pcm音频数据的api。所以它只能播放解码后的数据（MediaPlayer是播放编码格式的音频文件的，因为MediaPlayer包含了解码逻辑）
vap框架就是使用AudioTrack播放解码后的pcm数据

//AudioPlayer#startPlay
① 创建MediaExtractor（提取器），并绑定提取的目标文件
② 提取器寻找音频轨道下标，选中轨道，提取该音频轨道的格式信息（如果这个vap带音频轨道的话，就会继续往下执行）
③ 创建MediaCodec（解码器）
④ 开始解码音频
  while(true){
    //获取input缓冲区，将要解码的数据填入缓冲区，再将缓冲区加入到input队列里边
    //获取output缓冲区，将MediaCodec完成解码后的pcm数据，写入到AudioTrack进行播放
    ...
	}
```



### 支持透明通道流程 - MaskAnimPlugin

```java
将rgb视频和透明图层视频分离，两个视频的像素数据合并重新计算（rgb + a），生成argb像素数据，让视频支持透明通道
```



### 元素融合流程 - MixAnimPlugin

```java
将元素（bitmap、text）转成纹理，在帧率更新的时候读取json配置，查看元素对应信息，看看是否需要在当前帧融入该元素。

视频内容无法直接实现属性的插入，只能曲线救国，通过对属性图片进行修剪，欺骗用户的眼睛，让其看起来像是在视频内容里，实现最终的融合效果（效果如文章开头展示）。

为实现属性图片处理，需要引入“遮罩”素材，利用遮罩与属性图片进行Porter-Duff操作，就能得到需要的形状
再将结果贴到视频对应坐标位置，就能实现最后的融合效果。
“遮罩”素材保存在每一帧视频内容里，之前通过缩小Aplha区域，空出来的区域得到利用。
```





## OpenGL ES

```java
着色器 Shader（运行在gpu上的小程序）：执行 先画顶点再进行上色，所有是先执行vsh再执行fsh
vsh：vertex shader（顶点着色器）：作用是画形状
fsh：fragment shader（片着色器）：作用是上色、贴图
  
正常色器代码是写成.vsh、.fsh文件，放到raw文件夹里边的
也可以直接将着色器代码写到代码里边
  
  
opengl世界坐标（中心点0,0）
-1, 1   0, 1   1, 1
-1, 0   0, 0   1, 0
-1,-1   0,-1   1, -1
  
纹理坐标（第一象限）
0,1  1,1
0,0  1,0
  
  
着色器代码语法：
//attribute：定义的变量内容可以从java传递进来
//varying：定义的变量可以在任意地方使用，也就是vsh定义，fsh也能使用
  
//====== 一个vsh
//定义变量vPosition float[4]，这里需要用它来表示一个顶点(一个顶点有xyzw的坐标)
attribute vec4 vPosition;
attribute vec2 vCoor; //代表纹理坐标
varying vec2 aCoor; 
void main(){
  gl_Position = vPosition; //gl_Position是内置变量，这里是把坐标点赋值给gl_Position
  aCoor = vCoor; //vCoor赋值给aCoor
}

//====== 一个fsh
#extension GL_OES_EGL_image_external : require //扩展纹理
precision mediump float; //数据精度
varying vec2 aCoor; //跟vsh定义的一样，vsh的aCoor会传递过来
uniform samplerExternalOES vTexture; //samplerExternalOES：采样器（可以看做图片，接收java的每一帧图片）
void main(){
  //texture2D函数：从vTexture采样器里边采样一个RGBA值的像素值，aCoor这个像素点的RGBA值 
  gl_FragColor = texture2D(vTexture, aCoor);
}

```



## Android硬编解码API - MediaCodec

```java
MediaCodec - 编解码器（编码、解码）
  
MediaMuxer - 混合器（生成音频、视频或音视频混合文件）
  
MediaExtracto - 提取器（从容器中提取出不同轨道的数据，例如从mp4中提取视频轨道和音频轨道）
 
MediaCodec 生产者（input队列）- 消费者（output队列）
编码：相机采集数据 -> input队列 -> 编码 -> output队列 -> 混合器（生成mp4文件）
解码：mp4文件 -> 提取器（提取轨道）-> input队列 -> 解码 -> output队列 -> surface（显示）

解码：
//绑定surface
decoder.configure(format, surface, null, 0);
decoder.start();
while(true){
  //提取器 -> input队列
  ① 从input队列中申请有效的缓冲区下标 index = decoder.dequeueInputBuffer(TIMEOUT_USEC)
  ② 通过下标拿到对应的缓冲区 ByteBuffer buffer = decoder.getInputBuffers()[index]
  ③ 提取器读取样本数据到缓存区 size = extractor.readSampleData(buffer, 0)
  ④ 将填满数据的缓冲区加入到input队列 decoder.queueInputBuffer(index, 0, size, extractor.getSampleTime(), 0);
  
  //解码过程由MediaCodec完成，它就是从input队列拿到数据，解码之后，放到output队列
  //由于MediaCodec绑定了surface，所以会自动将解码后的数据，输出到surface中进行播放
  ...
}
```