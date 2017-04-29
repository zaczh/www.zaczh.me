---
layout: post
title:  "录制cocos2d-x游戏画面（iOS）"
date:   2017-04-29 10:28:01 +0800
categories: jekyll update
---
最近要实现一个功能：录制cocos2d-x游戏的画面，然后通过RTMP协议推流上传。
(cocos2d-x是一个基于OpenGL ES 2.0的开源2D图像引擎)

因为之前基于Metal做过类似的工作，所以首先想到的是OpenGL是否有类似Metal里面的“共享缓存”机制？。
首先查看官方的OpenGL编程指引：Apple:  OpenGL ES Programming Guide: Map Buffers into Client Memory for Fast Updates
文档指出了如何高效地在GPU和CPU之间共享buffer。但是遗憾的是，受限于OpenGL ES 2.x版本，这个方案仅仅能用来做vertex buffer，与Metal里的共享buffer相去甚远：
1. Buffer的长度最多仅为GL_BUFFER_SIZE，在iOS上，GL_BUFFER_SIZE=0x8764，根本没法用来放置纹理缓存；
2. shader不能修改buffer内容。
似乎比较鸡肋。
OpenGL ES 4.x提供了另一种用来在GPU和CPU之间共享buffer的方法：glMapBufferRange。
通过glGenBuffers方法创建，然后glBindBuffer绑定，最后通过glMapBufferRange（映射参数需要指定GL_MAP_COHERENT_BIT和GL_MAP_PERSISTENT_BIT）来更新或者获取GPU处理后的结果。
但是OpenGL ES 2不能使用，虽然Apple在OpenGL ES 2的extension里加入了glMapBufferRangeExt方法，但是却并不能指定映射参数，实际上效果还是和前面文档说到的方案一样。

实际上与Metal里对应的应该是Shader Storage Buffer Object（SSBO）https://www.khronos.org/opengl/wiki/Shader_Storage_Buffer_Object
这种buffer的size比较大，而且支持shader读写操作，但是仅支持OpenGL ES 4，而iOS最高仅支持OpenGL ES 3，所以用不了这个。（这样看来，Metal大致与OpenGL ES 4.x平级啊）

一开始想到的是readpixel，但是这个方法会阻塞CPU，导致CPU占用非常高，不适用。

Apple有一套自己的基于纹理的GPU CPU数据共享机制，Metal的是CVMetalTextureCacheRef, OpenGL的是CVOpenGLESTextureCacheRef。
Unity3D引擎充分利用了这两种cache，封装了一个对上层透明的统一texture cache层，但是cocos2d-x没有用到。
对OpenGL ES，Apple官方的说明指出其用途是用来做输入的，即传送给GPU的数据，在CPU端修改之后，可以快速地传递到GPU端；而实际上，[这个Cache也是可以用作输出的！](http://stackoverflow.com/a/9704392)如果输出不能cache，那么每次都要做一次memcpy操作，很浪费CPU时间；另一方面，如果这个Cache只能用做输入，那岂不是很鸡肋？纹理数据一般变动比较小，基本上传递给GPU之后就可以丢弃了，后续很少更新。

## 前期方案：
1. 采用类似Metal的接口的blit操作，但是因为接口仅支持OpenGL ES 3.x 放弃
2. readpixel方案：占用CPU过多
3. 采用texture cache 将fbo缓存到texture中 不可行
4. eagle context绘制两份 一份到fbo 一份到texture cache，因为cocos2d之前的绘制操作都绑定到一个fbo上面 绘制两份会导致texture cache的数据为空
5. 可否将fbo复制？不可行
6. 最终方案：先绘制到texture cache，再将texture绘制上屏

## 遇到的问题

1. shader不能在cocos初始化的时候就编译链接 必须重新创建一个program 幸好program之间的fbo texture等数据是共享的
2. 如何将Cocos原有的绘制操作最终绘到创建的纹理缓存上而不是屏幕上面呢？关键点：context以及renderbuffer。在启动的时候动态替换到Cocos的初始化方法
3. 如何将纹理绘制到屏幕上 需要绑定renderbuffer 类似2
4. 如何做纹理的缩放？目前仅支持对纹理缩小。

## 最终方案
最终的方案如下图所示：

<img src="/assets/image/cocos2d-x-texture-streaming-1.png">

与最初的设想相比，我做了如下几点改动：
1. 用于过渡的renderbuffer没有采用cache，而是一个标准的OpenGL texture；
2. 使用VBO(vetex buffer object)。最开始的时候，我是按照Apple的文档Best Practicles for Working with Vetex Data上面的示例来绘制纹理的，并没有使用VBO，在Cocos2d-x 3.x版本上工作良好，但是在2.x版本上会触发内存读取错误，程序挂在了cocos2d-x内部一个glVertexAttribPointer方法上，后来我将程序完全使用VBO来实现，能同时适配两个版本。至于原因，有待进一步的探查。

部分代码如下：

1. 绘制缓存纹理:
```Objective-C
- (void)renderCachedTextureScale {
    CHECK_GL_ERROR;
    
    if (colorRenderbuffer_ == 0) {
        NSLog(@"offscreen frame buffer not prepared");
        return;
    }
    
    CGSize resolution = [[QGRTMPClient sharedClient] videResolution];
    CGFloat videoWidth = resolution.width;
    CGFloat videoHeight = resolution.height;
    
    glUseProgram(self.textureProgramScale);
    glBindFramebuffer(GL_FRAMEBUFFER, self.textureFramebufferScale);
    glBindRenderbuffer(GL_RENDERBUFFER, self.textureRenderbufferScale);
    CHECK_GL_ERROR;
    
    glViewport(0, 0, videoWidth, videoHeight);
    
    glBindTexture(GL_TEXTURE_2D, colorRenderbuffer_);
    CHECK_GL_ERROR;
    
    glBindBuffer(GL_ARRAY_BUFFER, self.textureVBScale);
    CHECK_GL_ERROR;

    glVertexAttribPointer(self.textureCoordsSlotScale,
                          2,
                          GL_FLOAT,
                          GL_FALSE,
                          sizeof(VertexBuffer),
                          (void *)offsetof(VertexBuffer, Coords));
    glEnableVertexAttribArray(self.textureCoordsSlotScale);
    CHECK_GL_ERROR;

    glVertexAttribPointer(self.positionSlotScale,
                          2,
                          GL_FLOAT,
                          GL_FALSE,
                          sizeof(VertexBuffer),
                          (void *)offsetof(VertexBuffer, Position));
    glEnableVertexAttribArray(self.positionSlotScale);
    CHECK_GL_ERROR;

    glDrawArrays(GL_TRIANGLE_STRIP, 0, 4);
    CHECK_GL_ERROR;

    glBindTexture(GL_TEXTURE_2D, 0); // 使用完之后解绑
    glBindBuffer(GL_ARRAY_BUFFER, 0);
    glDisableVertexAttribArray(self.textureCoordsSlotScale);
    glDisableVertexAttribArray(self.positionSlotScale);
    
    CHECK_GL_ERROR;
}
```

2. 用于缩放操作的program初始化:
```Objective-C
- (void)prepareTextureProgramScale {
    if (self.textureProgramScale > 0) {
        return;
    }
    
    // 1
    GLuint fragmentShader = [self compileShader: @"\
                             precision mediump float;\
                             uniform sampler2D Texture;\
                             varying vec2 TextureCoordsOut;\
                             void main(void)\
                             {\
                                vec4 mask = texture2D(Texture, TextureCoordsOut);\
                                gl_FragColor = vec4(mask.rgb, 1.0);\
                             }"
                                     withType:GL_FRAGMENT_SHADER];
    
    GLuint vertexShader = [self compileShader:@"\
                           attribute vec2 Position;\
                           attribute vec2 TextureCoords;\
                           varying vec2 TextureCoordsOut;\
                           void main(void)\
                           {\
                              gl_Position = vec4(Position, 0, 1);\
                              TextureCoordsOut = vec2(TextureCoords.x, 1.0 - TextureCoords.y);\
                           }"
                                       withType:GL_VERTEX_SHADER];
    
    // 2
    GLuint programHandle = glCreateProgram();
    glAttachShader(programHandle, fragmentShader);
    glAttachShader(programHandle, vertexShader);
    glLinkProgram(programHandle);
    
    // 3
    GLint linkSuccess;
    glGetProgramiv(programHandle, GL_LINK_STATUS, &linkSuccess);
    if (linkSuccess == GL_FALSE) {
        GLchar messages[256];
        glGetProgramInfoLog(programHandle, sizeof(messages), 0, &messages[0]);
        NSString *messageString = [NSString stringWithUTF8String:messages];
        NSLog(@"[Error]%@", messageString);
    }
    CHECK_GL_ERROR;
    
    // 4
    glUseProgram(programHandle);
    
    // 5
    self.positionSlotScale = glGetAttribLocation(programHandle, "Position");
    self.textureSlotScale = glGetUniformLocation(programHandle, "Texture");
    self.textureCoordsSlotScale = glGetAttribLocation(programHandle, "TextureCoords");

    glEnableVertexAttribArray(self.positionSlot);
    CHECK_GL_ERROR;
    self.textureProgramScale = programHandle;
    
    GLuint framebuffer = 0;
    glGenFramebuffers(1, &framebuffer);
    NSAssert(framebuffer, @"Can't create texture frame buffer");
    self.textureFramebufferScale = framebuffer;
    
    glBindFramebuffer(GL_FRAMEBUFFER, self.textureFramebufferScale);
    //create scaled texture cache and store it
    CVPixelBufferRef buf = 0;
    self.textureRenderbufferScale = [self createTextureCacheStore:&buf];
    self.renderTargetPixelBuffer = buf;
    CHECK_GL_ERROR;

    glBindRenderbuffer(GL_RENDERBUFFER, self.textureRenderbufferScale);
    glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, self.textureRenderbufferScale, 0);
    CHECK_GL_ERROR;

    CGSize resolution = [[QGRTMPClient sharedClient] videResolution];
    int cacheTextureWidth = resolution.width;
    int cacheTextureHeight = resolution.height;
    
    NSLog(@"cocos2d: scaled surface size: %dx%d", (int)cacheTextureWidth, (int)cacheTextureHeight);
    
    CHECK_GL_ERROR;
    if(self.textureDepthbufferScale == 0) {
        GLuint textureDepthbuffer = 0;
        glGenRenderbuffers(1, &textureDepthbuffer);
        self.textureDepthbufferScale = textureDepthbuffer;
        NSLog(@"[Error]Can't create texture depth buffer");
    }
    
    glBindRenderbuffer(GL_RENDERBUFFER, self.textureDepthbufferScale);
    CHECK_GL_ERROR;

    glRenderbufferStorage(GL_RENDERBUFFER, GL_DEPTH_COMPONENT16, cacheTextureWidth, cacheTextureHeight);
    CHECK_GL_ERROR;

    glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, GL_RENDERBUFFER, self.textureDepthbufferScale);
    CHECK_GL_ERROR;

    [self setupVBScale];
    CHECK_GL_ERROR;

    GLenum error;
    if((error = glCheckFramebufferStatus(GL_FRAMEBUFFER)) != GL_FRAMEBUFFER_COMPLETE) {
        NSLog(@"[Error]Failed to make complete framebuffer object 0x%X", error);
    }
}
```
