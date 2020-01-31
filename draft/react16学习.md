## react 为什么需要引入fiber 架构？
#### 首先理解一下react的UI更新方式。
react 的一个关键特性是Virtual Dom，通过 diff 新旧Virtual Dom来计算出要做的修改，最终根据不同的环境通过不同的方式渲染更新到视图上。

<img src="https://raw.githubusercontent.com/HolyZheng/holyZheng-blog/master/images/virtualDom.png" width="80%"/>

###  Stack reconciler 
在先前的react中，采用的是stack reconciler的协调方式。react 从根组件或setState()的组件开始自顶向下递归，进行diff，并更新UI，整个过程不可打断。由于js线程和ui线程时互斥的，当遇到频繁的更新场景时，js就会持续占据主线程，页面就会出现卡顿的现象。

<img src="https://raw.githubusercontent.com/HolyZheng/holyZheng-blog/master/images/stackreconclier.jpeg" width="80%"/>

#### Fiber reconciler
为了解决这个问题，react16源码进行了一次重写，将reconciler过程进行时间分片，减轻卡顿的问题。这次重写将vDom tree 由 **树状结构** 改造为 **链表结构**，将基于递归的自顶向下遍历方式改造为循环的自顶向下遍历方式。遍历的上下文从原来存储于系统栈中变为存放在Fiber tree的节点中，更容易打断与恢复。

<img src="https://raw.githubusercontent.com/HolyZheng/holyZheng-blog/master/images/fiberLink.jpeg" width="80%"/>

把渲染/更新过程分成了 reconciliation 和 commit 两个阶段，reconciliation主要负责fiber tree的构建和diff运算，这个过程是可以拆分的。commit阶段主要是讲执行收集到的effect list 更新dom，这个阶段是不可以拆分的，因为涉及到dom操作，做拆分的话可能造成DOM实际状态与维护的内部状态不一致，另外还会影响体验

### 利用每帧的空闲时间

![](https://github.com/HolyZheng/holyZheng-blog/blob/master/images/framework?raw=true)

浏览器的每一帧就对应着浏览器dom视图的每一次更新。在每一帧中，浏览器要完成以下事情：
- 脚本执行
- 样式计算
- 布局
- 重绘
- 合成等

![](https://github.com/HolyZheng/holyZheng-blog/blob/master/images/framework2?raw=true)

详细可看图示。如果整个过程花的时间少的话，每一帧就会有空闲时间，比如我们浏览器的主流刷新频率是60Hz左右，那么每一帧的时间就是16ms，如果浏览器在16ms内完成了以上的工作，那么我们就可以利用这一帧中空闲出来的时间，这样就不会阻塞到浏览器视图的更新工作。

经过fiber改造后，reconciliation阶段是可以中断和恢复的。基于这个基础，可以将reconciliation阶段分片到浏览器每一帧的空闲时间当中去。我们浏览器中有个叫 **requestIdleCallback** 的api，可以在浏览器空闲时间去执行对应的回调函数。同时还可以设置timeout，避免回调一直得不到执行。
```js
    requestIdelCallback(myNonEssentialWork);
    
    function myNonEssentialWork (deadline) {
      // deadline.timeRemaining()  当前帧剩余时间
      while (deadline.timeRemaining() > 0 && tasks.length > 0) {
        doWorkIfNeeded();
      }
      if (tasks.length > 0){
        requestIdleCallback(myNonEssentialWork);
      }
    }
```
当然requestIdleCallback 的兼容性并不好，所以react自己实现了一个，一边将reconciliation阶段的工作放置于每一帧的空闲时间来执行。

### reconciliation 阶段如何运作

`ps: 以下两段引用自 黯羽轻扬的 完全理解React Fiber`
```
// 原react运行时3种实例。
DOM 真实DOM节点
-------
Instances React维护的vDOM tree node
-------
Elements 描述UI长什么样子（type, props）
```
```
// Fiber 架构下react运行时
DOM
    真实DOM节点
-------
effect
    每个workInProgress tree节点上都有一个effect list
    用来存放diff结果
    当前节点更新完毕会向上merge effect list（queue收集diff结果）
- - - -
workInProgress
    workInProgress tree是reconcile过程中从fiber tree建立的当前进度快照，用于断点恢复
- - - -
fiber
    fiber tree与vDOM tree类似，用来描述增量更新所需的上下文信息
-------
Elements
    描述UI长什么样子（type, props）
```
reconciliation 阶段的拆分，以每一个fiber tree 节点为单位，每处理一个节点（每一次循环）后就查看是否还有剩余时间，有就继续下一个单元，没有就等下一次空闲时间来执行。

在reconciliation过程中，react会同时维护两颗fiber tree，当前的current tree和一个正在构建的workInProgress tree。对fiber tree进行深度优先遍历，同时去构建workInProgress tree，可以将current tree理解为旧的vDom tree，workInProgress tree为新的vDom tree。如果当前节点有变化的话，就会在workInProgress tree这个节点中搭上effect tag记录当前节点要做的修改。（双缓冲）最终在commit阶段执行effect list 中所有的任务。



<img src="https://raw.githubusercontent.com/HolyZheng/holyZheng-blog/master/images/fiberCarton.jpeg" width="80%"/>

补充：
1. fiber 结构是怎样的？
2. 调度过程是怎样的？
3. 对应的源码在哪？

### react hooks最佳实践和带来影响？