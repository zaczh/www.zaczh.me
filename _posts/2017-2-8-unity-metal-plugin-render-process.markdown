---
layout: post
title:  "有关Unity iOS Metal插件渲染过程"
date:   2017-02-08 15:44:01 +0800
categories: jekyll update
---
最近在做一个Unity的iOS原生渲染插件，目标是将主摄像头的texture做简单处理之后传给CPU，参考了Unity官方提供的Demo（代码在[这里](https://bitbucket.org/Unity-Technologies/iosnativecodesamples/src/0bcacd4605720fb8525eade3b3ab867d3bc558aa/Graphics/MetalNativeRenderingPlugin/?at=5.4-stable))。这个Demo的运行过程如下：
1. 主Camera下面挂载了一个脚本TestMetalPlugin，在其OnPreRender方法中，将屏幕的RenderBuffer传给native端，在其OnPostRender方法中，将渲染事件传递给native端，然后native端再将接下来第二步中获取到的g_TextureCopy最终渲染到屏幕上；
2. 另外创建了一个TestCamera，在其OnPreRender方法中，将该摄像头的RenderBuffer传给native端，在其OnPostRender方法中，将渲染事件传递给native端，然后native端再将此时获取到的RenderBuffer（TestCamera的视野）Blit到另一个g_TextureCopy中，以供第一步使用。

这个Demo很完善，模拟了native端对Unity texture处理的过程，以及native端绘制自定义texture的方法。但是，我自己仅仅需要获取到主Camera的纹理，而无须再将处理后的texture再传回，所以我将两个脚本合成了一个。但是，问题来了，我发现通过这种方式获取到帧数据，有一些并没有包含那些在Unity脚本中onGUI方法创建的UI控件。奇怪的是，我如果使用两个脚本的话，就没有这个问题。
Unity里面脚本方法有执行先后顺序之分，OnPreRender和OnPostRender方法执行的时间都比onGUI要早，也许这是问题的一部分原因，但是根源在于Unity里面渲染过程和脚本执行过程[不在同一个线程里](https://docs.unity3d.com/ScriptReference/GL.IssuePluginEvent.html)：

`
Rendering in Unity can be multithreaded if the platform and number of available CPUs will allow for it. When multithreaded rendering is used, the rendering API commands happen on a thread which is completely separate from the one that runs the scripts. Consequently, it is not possible for your plugin to start rendering immediately, since it might interfere with what the render thread is doing at the time.
`

脚本的GL.IssuePluginEvent()方法会dispatch到渲染线程来执行，这样就有可能有时序问题，因为这个时候渲染过程并没有真正结束，还有OnGUI方法没有执行。那为什么挂载两个脚本就解决问题了呢？那是因为Unity对每个摄像头都是单独绘制的。也就是说，主摄像头绘制完毕了之后才会启动TestCamera的绘制过程，两者互不干扰，由于渲染线程只有一个，因此，如果在一个摄像头的脚本里获取另一个摄像头的数据，这个时候那个摄像头必然已经绘制完毕，就不会有时序的问题了。
