# Systrace的使用和分析
## 概述

Systrace是分析Android性能问题的神器，Google IO 2017上更是对其各种强推。Systrace其实和TraceView很像，都是统计一些方法（或者是一个执行阶段）的耗时，然后在一个有时间轴的图表上展示出来。不同的是，TraceView是收集所有方法的耗时信息和嵌套关系，这使得TraceView本身的性能消耗很大，反而影响了实际的运行环境。Systrace则采用了不同的思路，通过有限的Label先粗略统计出一个阶段的耗时，定位到问题以后，在慢慢细化和进一步测算分析。

摘抄一段Systrace的原理描述：
> 在介绍使用之前，先简单说明一下Systrace的原理：它的思想很朴素，在系统的一些关键链路（比如System Service，虚拟机，Binder驱动）插入一些信息（我这里称之为Label），通过Label的开始和结束来确定某个核心过程的执行时间，然后把这些Label信息收集起来得到系统关键路径的运行时间信息，进而得到整个系统的运行性能信息。Android Framework里面一些重要的模块都插入了Label信息（Java层的通过android.os.Trace类完成，native层通过ATrace宏完成），用户App中可以添加自定义的Label，这样就组成了一个完成的性能分析系统。

另一方面，系统的渲染的关键步骤都有framework预置的label，Systrace又提供了非常适合观察系统的UI性能/流畅度的功能，所以Systrace也是分析UI性能的神器。

## 使用

#### 环境条件

- Android SDK Tools 20
- 安装的python并添加了环境变量
- Android 4.1以上的手机

#### 启动trace并记录

可以使用DDMS进行记录：

- Tools -> Android -> Android Device Monitor
- 选择app，点击Systrace按钮
- 设置保存的文件路径、监控的时间、监控的内容等，开始记录

也可以使用命令行进行记录，参考：[Systrace Command Reference](https://developer.android.com/studio/profile/systrace-commandline.html)

#### 结果的展现

不论是DDMS还是命令行，最终都会保存一个html文件，用chrome浏览器打开，即可进行分析。

#### 常用标签

不论是DDMS还是命令行，都可以在启动trace前指定采集哪些系统预置的标签。下面把一些常用的标签类别列举如下：

- `Graphics`: Graphic系统的相关信息，包括SerfaceFlinger，VSYNC消息，Texture，RenderThread等；分析卡顿非常有效。
- `Input`:
- `View System`: View绘制系统的相关信息，比如onMeasure，onLayout等；对分析卡顿比较有帮助。
- `Window Manager`:
- `Activity Manager`: ActivityManager调用的相关信息；用来分析Activity的启动过程比较有效。
- `Application`:
- `Resource Loading`:
- `Dalvik VM`:  虚拟机相关信息，比如GC停顿等。
- `CPU Scheduling`: CPU调度的信息；你能看到CPU在每个时间段在运行什么线程；线程调度情况，比如锁信息。

## Systrace的分析

### 分析帧率

基本操作：

- 'w'按键：放大
- 's'按键：缩小
- 'd'按键：向右平移
- 'a'按键：向左平移

首先在左边纵向的pid列表中，定位到当前应用的包名所在的进程项，展开后包含`deliverInputEvent`, `UI Thread`, `RenderThread`三个子项。我们重点分析这部分内容。

右侧有一行写有F的圈圈，表示每个Frame的综合评价。如果是绿色，表示这一帧的渲染是ok的。如果是红色或者黄色，表示渲染超时了。
点击这个F圆圈，在下部的窗口中会展示一些详细信息。如果这一帧的渲染没有任何问题，下面的内容是空的；如果这一帧有问题，会展示Alert信息（我们稍后详细讨论Alert）。

![Systrace的界面概览和F标记](/assets/systrace1.png)
> 图：Systrace的界面概览和F标记

![观察有性能问题的Frame的提示信息](/assets/systrace2.png)
> 图：观察有性能问题的Frame的提示信息 

点击其中的一个方块，可以使这一段耗时高亮，在下面会展示出详细的耗时信息。

![详细](/assets/systrace3.png)
> 图：选中的Slice的详细耗时信息

### 举例：一个Frame的耗时信息详解

一帧的绘制大概需要经过这样几个区间（纵向的灰色竖条表示的是Vsync的时间）。

- Choreographer#doFrame （UI Thread）
    - traversal
        - measure（如果measure了多次，会在trace中有体现）
        - layout
        - draw
- DrawFrame (RenderThread)
    - syncFrameState
        - uploadTexture
    - flushDrawingCommands
    - eglSwapBuffersWithDamageKHR

![一帧的耗时](/assets/systrace4.png)
> 图：一帧的耗时

## 分析Alert的信息

Systrace会对trace中的一些事件执行自动分析，然后将一些性能问题以Alert形式标出，并给出一定的建议。前面已经提到，在有性能问题的Frame上点击，会在下方的详情中展示Alert，点击Alert项，会给出详细的耗时信息、相关描述，有的还会给出链接，链到指定的trace块中。

另外，右侧有一个Alert的tab页，会把此次trace中所有的Alert信息列举出来。点击其中一个Alert，对应的F圈会高亮，从而可以定位到有问题性能的帧。

![Alerts tab](/assets/systrace5.png)

发生Alert之后，可能我们已有的信息还不足以解决这些性能问题。此时，我们可以考虑先使用TraceView来定位耗时较长的方法，然后针对这些有问题的方法，在代码中添加Mark后，再执行Systrace进行确认和进一步的分析。（注意：TraceView本身有一定的性能损耗，会导致得出的结果和实际有较大出入，所以不建议使用TraceView取代Systrace，而是使用它作为查找问题的一个辅助工具）。

下一节我们进一步介绍，如何在代码中添加Systrace的Mark。

### 举例：一些Alert信息

这里列举一些常见的Alert信息：

- Scheduling delay
- Expensive measure/layout pass
- Expensive Bitmap uploads
- Inefficient ListView recycling/rebinding
- Inflation during ListView recycling
- Long View#draw()

## 从代码中追踪性能问题

从Android 4.3（API 18）开始，使用`Trace`类，可以在代码中添加更加详细的标记。使用`Trace.beginSection(String)`开始一段追踪，使用`Trace.endSection()`结束最近的一次追踪。

下面是官方说明里面摘抄的一段示例代码：

```
public class MyAdapter extends RecyclerView.Adapter<MyViewHolder> {

    @Override
    public MyViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        Trace.beginSection("MyAdapter.onCreateViewHolder");
        MyViewHolder myViewHolder;
        try {
            myViewHolder = MyViewHolder.newInstance(parent);
        } finally {
            Trace.endSection();
        }
        return myViewHolder;
    }

   @Override
    public void onBindViewHolder(MyViewHolder holder, int position) {
        Trace.beginSection("MyAdapter.onBindViewHolder");
        try {
            try {
                Trace.beginSection("MyAdapter.queryDatabase");
                RowItem rowItem = queryDatabase(position);
                mDataset.add(rowItem);
            } finally {
                Trace.endSection();
            }
            holder.bind(mDataset.get(position));
        } finally {
            Trace.endSection();
        }
    }
}
```


添加之后，我们就可以在抓到的trace中看到对应的耗时，非常方便。

![添加自定义的mark](/assets/systrace6.png)




## References

- [x] [性能工具Systrace的使用](http://gityuan.com/2016/01/17/systrace/)
- [x] [Analyzing UI Performance with Systrace](https://developer.android.com/studio/profile/systrace.html#app-trace)
- [x] [手把手教你使用Systrace（一）](https://zhuanlan.zhihu.com/p/27331842)
- [ ] [手把手教你使用Systrace（二）——锁优化](https://zhuanlan.zhihu.com/p/27535205?group_id=861682866420072448)