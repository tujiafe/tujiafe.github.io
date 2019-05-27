# 途家无痕埋点实现之路 - 实战

## 背景
目前移动互联网行业都是依附了着大数据平台来分析用户行为来做出更适用用于用户的产品，那么客户端就需要提供精准的数据采样的机制.

公司原有的埋点是采用手动代码埋点方案，使用起来虽然灵活，但由于开发成本较高（每次新的需求都意味着迎来不同的埋点需求）。随之我们开启了对无痕埋点的探索和实践，目标能将App可能发生的采样点做到通过一整套框架来Hook，达到真正的AOP来大大降低开发人员的开发成本.

## 为什么要实现无埋点

> **手动埋点和无痕埋点两套技术方案对比**

![](https://github.com/dengluoy/PicassoUtils/blob/master/excel.jpg?raw=true)

#### 从前的手动埋点流程分为几个步骤：
* PM列出所需要的数据点 (ps: 列出本期需求所需要看的数据点。如：某个按钮的点击量、统计热区是哪？)
* BI部门归纳产品需求采样列表.
* 开发基于BI部门列出的数据采样点，做每个采样点做对应的开发.


![流程1](https://github.com/dengluoy/PicassoUtils/blob/master/flow4.jpg?raw=true)

#### 实现无痕埋点之后的步骤:
* PM列出本期需求所想看的数据 (ps: 列出本期需求所需要看的数据点。如：某个按钮的点击量、统计热区是哪？)
* BI基于PM需求，列出对应的数据点。
* RD基于BI给的需求只需要对每个页面添加一行代码。


![流程12](https://github.com/dengluoy/PicassoUtils/blob/master/flow5.jpg?raw=true)

> ***可以看到。无痕埋点在每次需求开发时，大大减少了开发时间、开发风险***


## 无痕埋点技术架构
### 1. Gradle Plugin开发，实现AOP开发
### 2. 提供数据采用工具类开发，主要处理对应数据格式上报

#### 我们选择在编译期间做到AOP开发，做到全局数据采样并同时开发人员无感知。

> **框架整体运行架构图**

![](https://github.com/dengluoy/PicassoUtils/blob/master/framework3.jpg?raw=true)


> **SDK架构图**

![架构1](https://github.com/dengluoy/PicassoUtils/blob/master/framework2.jpg?raw=true)

### Gradle Plugin开发关键技术点

#### Hook Click事件、Dialog弹窗事件等
1. 采用Transform API 拦截.class转变.dex生命周期
2. 采用ASM做字节码注入。

***Transform工作流程***
> Gradle Transform API 是Gradle1.5.0-beta1 引入的。可以用于在Android打包过程中设置Hook点

![](https://upload-images.jianshu.io/upload_images/2965551-f289ae9bb2767f59?imageMogr2/auto-orient/strip%7CimageView2/2/w/980/format/webp)



既然我们想用字节码注入的方式来实现无埋点，那么我们需要对整体的Android打包机制有一定的了解。咱们先来看下Android的整体的构建流程.

![](https://user-gold-cdn.xitu.io/2018/3/8/16204d3ceedb2328?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

Android使用Gradle打包的本质其实是将上图的基本每个节点拆解成一个一个单独Task来完成。如: aapt、JavaCompiler、apkBuilder等等.
那么根据我们的需求其实可以发现。其中JavaCompiler之后的这个时间节点是完全可以满足我们的需求的。

![](https://github.com/dengluoy/PicassoUtils/blob/master/class.jpg?raw=true)

#### ASM注入流程
我们在生成.class文件后，拦截掉对应需要注入的Class文件。随后扫描该类的Method列表。找到对应的代码块注入数据采样代码
> 具体流程如下:

![](https://github.com/dengluoy/PicassoUtils/blob/master/flow3.jpg?raw=true)


### 数据采样SDK核心代码实现
> 分为以下几个模块

#### 1. 构建View唯一标识
#### 2. 页面划分 -区分Fragment页面
#### 3. Dialog弹窗代理类实现 (在代理类中实现数据采样)
#### 4. 统计页面走向、前后台切换

#### 构建View唯一标识
1.1. 利用TargetView向上遍历ViewTree找到根节点，同时过滤掉DecorView已达到最小、唯一的ViewPath路径。来确定点击id

1.2. 同时根据采集过来的Fragment集合中寻找对应的FragmentRootView，添加Fragment节点
最后的数据展示

```
ContentView/FrameLayout[0]/LinearLayout[0]/ViewPager[0]/FragmentA[0]/AppCompatButton[0]
```

#### 页面划分

2.1. 基于Fragment的onResume、onPause、onHiddenChanged、setUserVisibleHint四个生命周期的数据采样

2.2. 维护一个Fragment集合。在每次的点击中。根据FragmentRootView节点判断，当前点击发生的页面名称

#### Dialog弹窗代理类实现

3.1. 通过实现一个Dialog类别的代理对象，复写Dialog的show方法。同时在编译期间将Dialog类的对象替换成代理对象。来达到无痕注入效果


***如下代码:***
```
@Override
public void show() {
    super.show();
    //添加数据采样逻辑
}
```

#### 统计页面走向、前后台切换
4.1. 添加ActivityLifecycleCallabck类。

4.2. 统计页面流动走向

4.3. 统计前后台切换时机 [参考文章](https://blog.csdn.net/bzlj2912009596/article/details/80073396)

## 结语
途家目前已经实现了实现App整体的无痕埋点SDK。目前已经使用的公司有途家、蚂蚁短租、安伴智能等企业。

### 总结:
1. 无痕埋点不是真正完全不加任何代码，但可以大大降低项目带来成本和风险
2. 随着现在的数据时代快速发展，为了满足公司各个业务日益复杂的数据需求，我们仍然需要接续优化完善采样框架
3. 后续工作需要继续优化及完善。快速搭建可视化平台、达到完全脱离开发，PM独立能够获取一切数据信息
