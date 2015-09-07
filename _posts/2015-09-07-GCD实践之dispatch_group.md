---
layout: post
title: GCD实践之dispatch_group 
---


## 发一条动态

在很多UGC产品中（如微博、赤兔、朋友圈），发一条图文并茂的动态总是群众喜闻乐见的。如下图：

<img src="https://raw.githubusercontent.com/liaojinxing/liaojinxing.github.io/master/ScreenShot/post_feed.png" alt="发布动态" width="200px" hspace="10"/>

实现这样的发布器，我见过多种方案。

方案一：因为发图可能是比较耗时的，而且有多张图片，所以先不管图片，将文字post给server返回发布成功的动态，成功以后再把每一张图片附加到这条动态。但是，从用户的角度来说，发表动态应该是一个原子操作，这种方案很容易引起数据不完整，用户很困惑，后果很严重。弃。

方案二：先将这多张图上传到server返回图片对应的url，然后再把这些url和文字作为动态的属性post server。这就比较科学了。今天要讨论的就是如何提高效率。

## 小明的代码
耗时的UI操作使用多线程可以有效提高性能，在iOS开发中，多线程的实现主要靠gcd和NSOperation，各有优劣和使用场景，这不是此文的重点。

分析上述方案，可以得知：

有一个约束：必须所有的图片都上传成功以后再发布动态；

小明可能这样写：

```
sendPhoto(image1, success:^(NSString *url){
    sendPhoto(image2, success:^(NSString *url) {
       	sendPhoto(image3, success:^(NSString *url) {
            .......
            postFeed(imageURLs, text);
        });
    });
});
```

这样完全就是串行，发个动态要半天，老板要哭了！

## 小红的代码

在这个例子中，其实多图的发布可以并发操作，dispatch_group大显身手的时候到了。

![Smaller icon](https://raw.githubusercontent.com/liaojinxing/liaojinxing.github.io/master/ScreenShot/dev_apple_dispatch_group.png "dispatch_group")

dispatch_group提供了多个block之间的同步机制，可以在多个block都执行完成之后，再执行有依赖的任务。

于是，小红改了代码：

```
NSMutableArray *imageURLs= [NSMutableArray array];
dispatch_group_t group = dispatch_group_create();                    // 1
for (UIImage *image in images) {
    dispatch_group_enter(group);                                    // 2
    sendPhoto(image, success:^(NSString *url) {
        [imageURLs addObject:url];
        dispatch_group_leave(group);                                 // 3
    });
}
dispatch_group_notify(group, dispatch_get_global_queue(), ^{         // 4
    postFeed(imageURLs, text);
});
```

1. 首先创建一个dispatch_group
2. `dispatch_group_enter`通知group说你的任务要开始了，`dispatch_group_leave`则表示我做完了。需要注意的是，enter和leave必须配对。
3. 当group中所有的任务都执行完，`dispatch_group_notify`收到通知，现在终于可以发布动态了。

对比小明和小红的代码，小明的模型是这样的：

<img src="https://raw.githubusercontent.com/liaojinxing/liaojinxing.github.io/master/ScreenShot/xiaoming_dispatch_group_model.png" alt="小明的代码" width="4jjjjjjjj00px" hspace="10"/>

小红的是这样的：

<img src="https://raw.githubusercontent.com/liaojinxing/liaojinxing.github.io/master/ScreenShot/xiaohong_dispatch_group_model.png" alt="小红的代码" width="150px" hspace="10"/>

嗯，大家都喜欢小红。
