### TTAvplayerCache

我们平时看到的视频文件有许多格式，比如 avi， mkv， rmvb， mov， mp4等等，这些被称为[容器](http://en.wikipedia.org/wiki/Digital_container_format)（[Container](http://wiki.multimedia.cx/index.php?title=Category:Container_Formats)）， 不同的容器格式规定了其中音视频数据的组织方式（也包括其他数据，比如字幕等）。容器中一般会封装有视频和音频轨，也称为视频流（stream）和音频流，播放视频文件的第一步就是根据视频文件的格式，解析出其中封装的视频流、音频流以及字幕（如果有的话），解析的数据读到包 （packet）中，每个包里保存的是视频帧（frame）或音频帧，然后分别对视频帧和音频帧调用相应的解码器（decoder）进行解码，比如使用 H.264编码的视频和MP3编码的音频，会相应的调用H.264解码器和MP3解码器，解码之后得到的就是原始的图像(YUV or RGB)和声音(PCM)数据，然后根据同步好的时间将图像显示到屏幕上，将声音输出到声卡，最终就是我们看到的视频。



### 1.视频播放的几种选择

#### 1.1. 对于简单的视频播放可以使用**UIImagePickerController**。它既可以拍照也能录制视频。

```objective-c
//使用UIImagePickerController视频录制
UIImagePickerController *picker = [[UIImagePickerController alloc] init];
picker.delegate = self;
if ([UIImagePickerController isSourceTypeAvailable:UIImagePickerControllerSourceTypeCamera]) {
picker.sourceType = UIImagePickerControllerSourceTypeCamera;
}

//mediaTypes设置摄影还是拍照
//kUTTypeImage 对应拍照
//kUTTypeMovie  对应摄像
//    NSString *requiredMediaType = ( NSString *)kUTTypeImage;
NSString *requiredMediaType1 = ( NSString *)kUTTypeMovie;
NSArray *arrMediaTypes=[NSArray arrayWithObjects:requiredMediaType1,nil];
picker.mediaTypes = arrMediaTypes;
//    picker.videoQuality = UIImagePickerControllerQualityTypeHigh;默认是中等
picker.videoMaximumDuration = 60.; //60秒
[self presentViewController:picker animated:YES completion:^{

}];
```



#### 1.2. 使用AVPlayerViewController

```objective-c
AVPlayer *player = [AVPlayer playerWithURL:self.finalURL];
AVPlayerViewController *playerVC = [[AVPlayerViewController alloc] init];
playerVC.player = player;
playerVC.view.frame = self.view.frame;
[self.view addSubview:playerVC.view];
[playerVC.player play];
```



#### 1.3. 复杂的业务需求，需要自定义视频界面时，一般分为： 

（1）使用`AVFoundation`拍照和录制视频，自定义界面

（2）使用`AVAssetExportSession`压缩和转码

（3）使用`AFNetWorking`上传到服务器

（4）网络请求，使用`AVFoundation`框架的`AVPlayer`来自定义播放界面，在线播放视频流。播放又分为先下后播和边下边播。



### 2. AVPlayer

AVPlayer是用于管理媒体资产的播放和定时的控制器对象，它提供了控制播放器的有运输行为的接口，如它可以在媒体的时限内播放，暂停，和改变播放的速度，并有定位各个动态点的能力。我们可以使用AVPlayer来播放本地和远程的视频媒体文件。

- AVPlayer：播放器，将数据解码处理成为图像和声音。

- AVAsset：主要用于获取多媒体信息，是一个抽象类，不能直接使用。

- AVURLAsset：AVAsset的子类，可以根据一个URL路径创建一个包含媒体信息的AVURLAsset对象。负责网络连接，请求数据。

- AVPlayerItem：一个媒体资源管理对象，管理者视频的一些基本信息和状态，负责数据的获取与分发；一个AVPlayerItem对应着一个视频资源。

![relationship](/Users/meitu/Desktop/avplayer缓存方案/relationship.png)



创建一个视频播放器的思路:

- 创建一个view用来放置AVPlayerLayer

- 设置AVPlayer AVPlayerItem 并将 AVPlayer放到AVPlayerLayer中，再将AVPlayerLayer添加到view.layer中
- 添加观察者，观察播放源的状态，如果状态是AVPlayerItemStatusReadyToPlay就开始播放
- 其他功能添加




### 3. 缓存

如果直接设置AVplayer通过AVPlayerItem播放，系统并不会做缓存处理。

#### 3.1 HTTPServer

在本地开启一个 http 服务器，把需要缓存的请求地址指向本地服务器，并带上真正的 url 地址。



#### 3.2  AVAssetResourceLoaderDelegate

```objective-c
AVURLAsset *urlAsset = ...
[urlAsset.resourceLoader setDelegate:<AVAssetResourceLoaderDelegate> queue:dispatch_get_main_queue()];
```

```objective-c
- (BOOL)resourceLoader:(AVAssetResourceLoader *)resourceLoader shouldWaitForLoadingOfRequestedResource:(AVAssetResourceLoadingRequest *)loadingRequest
```

```objective-c
- (void)resourceLoader:(AVAssetResourceLoader *)resourceLoader didCancelLoadingRequest:(AVAssetResourceLoadingRequest *)loadingRequest
```

在 `AVAssetResourceLoadingRequest`里面，`request`代表原始的请求，由于 AVPlayer 是会触发分片下载的策略，还需要从`dataRequest`中得到请求范围的信息。有了请求地址和请求范围，我们就可以重新创建一个设置了请求 Range 头的 NSURLRequest 对象，让下载器去下载这个文件的 Range 范围内的数据。

而对应的下载我们可以使用NSURLSession 实现断点下载。



代理对象需要实现的功能

- 1.接收视频播放器的请求，并根据请求的range向服务器请求本地没有获得的数据
- 2.缓存向服务器请求回的数据到本地
- 3.如果向服务器的请求出现错误，需要通知给视频播放器，以便视频播放器对用户进行提示 



代理对象处理流程

- 1.当视频播放器向代理请求dataRequest时，判断代理是否已经向服务器发起了请求，如果没有，则发起下载整个视频文件的请求
- 2.如果代理已经和服务器建立链接，则判断当前的dataRequest请求的offset是否大于当前已经缓存的文件的offset，如果大于则取消当前与服务器的请求，并从offset开始到文件尾向服务器发起请求（此时应该是由于播放器向后拖拽，并且超过了已缓存的数据时才会出现）
- 3.如果当前的dataRequest请求的offset小于已经缓存的文件的offset，同时大于代理向服务器请求的range的offset，说明有一部分已经缓存的数据可以传给播放器，则将这部分数据返回给播放器（此时应该是由于播放器向前拖拽，请求的数据已经缓存过才会出现）
- 4.如果当前的dataRequest请求的offset小于代理向服务器请求的range的offset，则取消当前与服务器的请求，并从offset开始到文件尾向服务器发起请求（此时应该是由于播放器向前拖拽，并且超过了已缓存的数据时才会出现）
- 5.只要代理重新向服务器发起请求，就会导致缓存的数据不连续，则加载结束后不用将缓存的数据放入本地cache
- 6.如果服务器返回其他错误，则代理通知播放器网络错误











