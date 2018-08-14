#视频播放

### 1.播放功能综述

#### 1.1 AVPlayer

AVFoundation 的播放都围绕`AVPlayer`类展开,`AVPlayer`是一个用来播放基于时间的视听媒体的控制器对象.支持播放的流媒体如下:

- 本地流媒体文件
- 分步下载的流媒体文件
- 通过 HTTP Live Streaming 协议得到的流媒体文件.

上面说的`AVPlayer是一个控制器对象`,这里说的`控制器`它不是一个视图或窗口控制器,而是一个对播放和资源时间相关信息进行管理的对象.

**AVPlayer 是一个不可见组件**.

- 如果播放 MP3 或 AAC 音频文件,那么没有可视化的用户界面也不会有什么问题.
- 不过如果要播放一个 QuickTime 电影或一个 MPEG-4 视频,会导致非常不好的用户体验.
- 要将视频资源导出到用户界面的目标位置,需要使用`AVPlayerLayer`类.
- AVPlayer 只管理一个单独资源的播放,不过框架还提供了 AVPlayer 的一个子类 AVQueuePlayer, 可以用来管理一个资源队列.当需要在一个序列中播放多个条目或者为音频、视频资源设置播放循环时可使用该子类.

#### 1.2 AVPlayerLayer

`AVPlayerLayer`构建于 Core Animation 之上,是 AVFoundation 中能找到的为数不多的可见组件.

`AVPlayerLayer`扩展了 Core Animation 的 CALayer 类,并通过框架在屏幕上显示视频内容.这一图层并不提供任何可视化控件或其他根据开发者需求要搭建的控件,但是它用作视频内容的渲染面.

创建 AVPlayerLayer 需要一个指向 AVPlayer 实例的指针,这就将图层和播放器紧密绑定在一起,保证了当播放器基于时间的方法出现时使二者保持同步.

AVPlayerLayer 与其他 CALayer 一样,可以设置为 UIView 或 NSView 的备用层,或者可以手动添加到一个已有的层继承关系中.

在这一层中开发者可以自定义的领域只有 `video gravity`.总共有3个不同的 gravity 值,用来确定在承载层的范围内视频可以拉伸或缩放的成都.

- AVLayerVideoGravityResizeAspect
    - 在承载层的范围内缩放视频大小来保持视频的原始宽高比 
- AVLayerVideoGravityResizeAspectFill
    - 保留视频的宽高比,并使其通过缩放填满层的范围区域. 
- AVLayerVideoGravityResize
    - 将视频内容拉伸来匹配承载层的范围.不常用,会使视频扭曲.

    
#### 1.3 AVPlayerItem

最终目的: 使用 AVPlayer 来播放 AVAsset.

因为 AVAsset 模型只包含媒体资源的`静态信息`,这些不变的属性用来描述对象的静态状态.

所以这意味着`仅仅使用 AVAsset 对象是无法实现播放功能的`.如它不能获取当前时间.

所以当我们需要对一个资源及其相关曲目进行播放时,首先要通过`AVPlayerItem`和`AVPlayerItemTrack`类构建相应的动态内容.

`AVPlayerItem`会建立媒体资源动态视角的数据模型并保存 AVPlayer 在播放资源时的呈现状态.

`AVPlayerItem`由一个或多个媒体曲目组成,由`AVPlayerItemTrack`类建立模型.`AVPlayerItemTrack`实例用来表示播放器条目中的类型统一的媒体流. 比如音频或视频.`AVPlayerItem`中的曲目直接与基础`AVAsset`中的`AVAssetTrack`实例相对应.
    
### 2.播放秘籍

播放的基础架构设置.

```objc
- (void)playbackVideo {
    NSURL *assetURL = [[NSBundle mainBundle] URLForResource:@"刺激战场" withExtension:@".mp4"];
    AVAsset *asset = [AVAsset assetWithURL:assetURL];
    AVPlayerItem *playerItem = [AVPlayerItem playerItemWithAsset:asset];
    //对 AVPLayerItem 的 status 属性添加观察者
    [playerItem addObserver:self forKeyPath:@"status" options:NSKeyValueObservingOptionNew context:&PlayerItemStatusContext];
    self.player = [AVPlayer playerWithPlayerItem:playerItem];
    AVPlayerLayer *playerLayer = [AVPlayerLayer playerLayerWithPlayer:self.player];
    [self.view.layer addSublayer:playerLayer];
}

#pragma mark - Observing Method
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context {
    if (context == &PlayerItemStatusContext) {
        AVPlayerItem *playerItem = (AVPlayerItem *)object;
        if (playerItem.status == AVPlayerStatusReadyToPlay) {
            // 播放
            [self.player play];
        }
    }
}
```

- AVPlayerItem 具有一个名为`status`的`AVPlayerItemStatus`类型的属性.
- 在对象创建之初,播放条目由`AVPlayerItemStatusUnknown`状态开始.
- `AVPlayerItemStatusUnknown`状态表示:
    - 表示当前媒体还未被载入并且不在播放队列中.
- 将`AVPlayerItem`与一个`AVPlayer`对象进行关联就开始将媒体放入队列中,但是在具体内容可以播放前,需要等待对象的状态由`AVPlayerItemStatusUnknown`变为`AVPlayerItemStatusReadyToPlay`
- 开发者可以通过`KVO`机制监听`status`属性的改变.当`status`属性的值变为`AVPlayerItemStatusUnknown`时,可以播放了.

### 3.处理时间

回顾之前的`AVAudioPlayer`,可以看到时间使用`NSTimeInterval`展示的,其实就是简单的对`double`值进行了`typedef`定义.

不过使用浮点型数据类型表示时间存在一定的问题,因为浮点型数据的运算会导致不精确的情况.

当进行多时间计算累加时这些不精确的情况就会特别严重,经常导致时间的明显偏移,使得媒体的多个数据流几乎无法实现同步.

此外,以浮点型数据呈现时间信息无法做到自我描述,这就导致在使用不同时间轴进行比较和运算时比较困难.

AVFoundation 使用一种可靠性更高的方法来展示时间信息,这就是基于 `CMTime` 数据结构.

**CMTime**

AVFoundation 是基于 `Core Media`的高层封装.`Core Media`是基于`C`的底层框架,提供了许多处理`Mac 和 iOS`媒体栈的关键功能.

虽然这个框架通常都在后台工作,不过其中一个我们经常能够接触的部分就是他的数据结构`CMTime`.

`CMTime`为时间的正确表示给出了一种结构,即`分数值`的方式.

```c
typedef struct {
    // 64位整数值
    CMTimeValue value;
    
    // 32位整数值
    CMTimeScale timescale;
    
    CMTimeFlags flags;
    CMTimeEpoch epoch;
} CMTime;
```   

- `CMTimeValue` 和 `CMTimeScale` 在时间呈现样式中分别作为分子和分母.

**CMTimeMake 函数创建时间**

```objc
// 0.5 秒
CMTime halfSecond = CMTimeMake(1, 2);

// 5 秒
CMTime fiveSecond = CMTimeMake(5, 1);

// One sample from a 44.1 kHz audio file
CMTime oneSample = CMTimeMake(1, 44100);

// 时间值为 0
CMTime zeroTime = KCMTimeZero;
```

### 4.创建视频播放器


