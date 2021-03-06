### 图像的显示原理
![](./Snip20190228_23.png)


**在iOS中是双缓冲机制，有前帧缓存、后帧缓存**

[iOS 视图，动画渲染机制探究](https://www.cnblogs.com/bugly/p/5056578.html)


#### CPU的工作


```mermaid
graph LR

A[Layout] --> B[Display]
B --> C[Prepare]
C --> D[Commit]

```

* Layout: UI布局、文本计算等
 此阶段主要是准备你的视图/图层的层级关系,设置 layer 的属性，如 frame，background color 等等 
 
* Display:绘制过程、drawRect:调用
 此阶段是图层的寄宿图片被绘制的阶段。创建 backing image,在这个阶段程序会创建 layer 的 backing image，无论是通过 setContents 将一个 image 传給 layer，还是通过 [drawRect:] 或 [drawLayer: inContext:] 来画出来的。所以 [drawRect:] 等函数是在这个阶段被调用的

* Prepare:图片的解码、动画过程中将要显示的图片的时间点
 此阶段Core Animation准备发送动画数据到渲染服务的阶段
 在这个阶段，Core Animation 框架准备要渲染的 layer 的各种属性数据，以及要做的动画的参数，准备传递給 render server。同时在这个阶段也会解压要渲染的 image。（除了用 imageNamed：方法从 bundle 加载的 image 会立刻解压之外，其他的比如直接从硬盘读入，或者从网络上下载的 image 不会立刻解压，只有在真正要渲染的时候才会解压）
 
 

* Commit:提交位图。
最后的阶段，`Core Animation`打包所有图层和动画属性,然后通过`IPC(内部处理通信)`发送到渲染服务进行显示。

#### GPU渲染管线
```mermaid
graph LR

A[顶点着色] --> B[图元装配]
B --> C[光栅化]
C --> D[片段着色]
D --> E[片段处理]
E --> F[FrameBuffer]
```
顶点着色主要是对位图进行处理。

在GPU进行绘制的时候, 首先由顶点着色器对传入的顶点数据进行运算。 

再通过图元装配,将顶点转换为图元。
然后进行光栅化，将图元这种矢量图形，转换为栅格化数据。
最后，将栅格化数据传入片段着色器中进行运算。

片段着色器会对光栅化数据中的每一个像素进行运算,并决定像素的颜色。


当上诉步骤处理完毕之后,会提交到帧缓存区[FrameBuffer]，然后由视频控制器有Vsync信号到来之前去帧缓存区提取将要显示到屏幕上的内容。



### UI卡顿、掉帧的原因
![](./Snip20190228_25.png)

在规定的16.7ms的时间内,cpu和gpu在这个时间内，没有完成下一帧画面的合成。

#### 滑动的优化方案
CPU:
* 在子线程中处理对象的创建、调整、销毁。
* 在子线程中提前处理预排版(布局计算、文本计算)
* 预渲染(文本的异步绘制,图片编解码等)

GPU:
* 纹理渲染 尽量避免离屏渲染、以及CPU预渲染减轻GPU的压力 避免重绘(主要是因为半透明图层)GPU的填充比率是有限的
* 视图混合 减轻图层的复杂度,避免太多的几何结构
* 避免处理过大的图片 如果图片绘制超出GPU支持的2048X2048 或者 4096X4096尺寸的纹理,则必须在显示之前进行预处理


#### 绘制原理

![](./Snip20190228_26.png)

当我们在调用 setNeedsDisplay的时候,并不会立即触发渲染而是会调用layer的同名方法给layer打上脏标记,然后在当前runloop结束的时候，调用 display.

##### 系统的绘制流程

![](./Snip20190228_27.png)
backing store 我们可以理解为合成了bitmap(位图)。

##### 异步绘制的流程

![](./Snip20190228_28.png)

实现的过程:

![](./Snip20190228_29.png)


#### 离屏渲染
触发离屏渲染会增加GPU的工作量,而增加了GPU的工作量可能会导致CPU和GPU的工作总耗时超过了16.7ms，有可能会导致UI的卡顿和掉帧。