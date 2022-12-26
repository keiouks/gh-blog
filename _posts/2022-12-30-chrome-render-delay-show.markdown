---
layout: posts
classes: wide
title:  "chrome 渲染延迟上屏分析"
date:   2022-12-30 23:25:32 +0800
categories: 技术分析
---

### 名词解释：

- 渲染：指chrome的blink引擎把html内容和样式渲染到dom树上，但不包括显示到屏幕上；
- 上屏：指blink引擎把渲染内容交给合成层，让其调度GPU显示到屏幕上；
- FCP：是一个性能指标，指第一次有实质性内容的渲染，更详细定义可以去网上查；
- blink：chrome的渲染引擎，每个浏览器都会有一个渲染引擎负责把html内容渲染到dom上，以及进一步调度GPU把内容显示到屏幕上；

### 背景

项目开发中发现，chrome渲染了html内容之后，并没有马上显示到屏幕上，而是等到下一次渲染后才显示内容。对于复杂项目，下一次渲染可能耗时很久，比如可能是一秒后，对于希望尽快显示内容的场景，这个效果十分影响用户体验。本文会介绍该特性的官方用意，还会从源码分析其底层逻辑，也会给出实验来复现该特性，以及给出加快显示上屏的方法。

### 官方说明

一开始遇到上屏慢的问题后，就开始通过各种手段做分析，并且通过分析源码，解读出上屏慢的底层逻辑，以及解决的手段。

后来在网上查找到了官方的相关说明，chrome从某个版本开始支持[**Paint Holding特性**](https://developer.chrome.com/blog/paint-holding/)。很多web应用切换页面时，采用url跳转，那么跳转时页面内容会先清空，显示一片空白，即使网络速度很快，也免不了一闪而过的白屏，这样的用户体验不好。而[**Paint Holding特性**](https://developer.chrome.com/blog/paint-holding/)针对这种情景，当发生同源页面跳转时，先不清空页面内容，也不急着渲染下一个页面的内容，而是先保持显示上一个网页，直到要跳转到的页面先触发一次FCP，然后再触发一次渲染才让它的内容上屏。这样做对于web应用来说，网站上各个页面内容框架类似，切换页面时不会出现先显示空白，给人感觉就像是主内容切换，而整体相似内容一直保持，这样体验更好。详细说明可以看官方介绍。

官方说明中的其中一段话的第一句描述了[**Paint Holding特性**](https://developer.chrome.com/blog/paint-holding/)主要通过推迟上屏，并且推迟到FCP或者超过一定时间之后。

[![chrome-paint-holding-describe](https://github.com/keiouks/paint_holding_analyze/blob/main/img/chrome-paint-holding-describe.png?raw=true)](https://github.com/keiouks/paint_holding_analyze/blob/main/img/chrome-paint-holding-describe.png?raw=true)

奇怪的是，实践中发现，第一次直接打开一个页面时，以及，非同源网页跳转时，也同样会有[**Paint Holding特性**](https://developer.chrome.com/blog/paint-holding/)。这就导致了页面打开时ssr内容未能尽快上屏，从源码上也发现，chrome加了一个非同源[**Paint Holding特性**](https://developer.chrome.com/blog/paint-holding/)来支持除了同源跳转以外的所有情况下都展示[**Paint Holding特性**](https://developer.chrome.com/blog/paint-holding/)，至于为什么要这样做，我不清楚。

### 表现

先构造demo看看[**Paint Holding特性**](https://developer.chrome.com/blog/paint-holding/)的实际效果。

用`node + koa`起一个简单的http服务，**注意**，需要通过请求服务器打开网页才能展示chrome正常打开页面时的所有特性，浏览器直接打开本地html文件会导致完全不一样的特性。

服务器代码放在另一个github库，这是[代码链接](https://github.com/keiouks/paint_holding_analyze/blob/main/index.js)，本文相关的所有代码和文件都放到这个库里面，方便后续的复现。

把库clone到本地后，先通过命令安装依赖：

```bash
npm i
```

然后可以启动服务器：

```bash
node index.js
```

服务会跑在3000端口。

页面都是html文件，这里要先解释一下代码中多次出现的某个片段：

```html
<script>
    let x = Date.now();
    while(Date.now() <= x + 17){
        ;
    }
</script>
<div style="display:none;">
    <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
    <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
    <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
    <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
    <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
    <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
    <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
    <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
    <b>b</b><b>b</b>b
</div>
```

这个代码片段的目的是为了让blink引擎停止继续往下执行js代码，并触发渲染。一个html文件会包含很多元素标签，blink不会解析到一个html标签就渲染，这样渲染的频率就会太高，也太消耗资源，所以blink触发渲染有自己的策略。这个代码片段，先让js阻塞17毫秒(16毫秒也行)，再继续解析一堆b标签，目的是迎合了blink触发渲染的底层逻辑，让它触发渲染。相关源码分析之后有时间再写，这是别人已经做过的分析，后续可以再完善一下再写一个总结。这个代码片段本文就称它为`渲染片段`，这里只要记住它会触发渲染就行。

第一个demo的页面代码[在这里](https://github.com/keiouks/paint_holding_analyze/blob/main/views/index1.html):

```html
<!DOCTYPE html>
<html>
<head>
    <title>渲染测试</title>
</head>
<body>
    <div>第1个内容</div>
    <script>
        let x = Date.now();
        while(Date.now() <= x + 17){
            ;
        }
    </script>
    <div style="display:none;">
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b>b
    </div>
    <script>
        let x1 = Date.now();
        while(Date.now() <= x1 + 2000) {
            ;
        }
    </script>
    <div>第2个内容</div>
    <script>
        let x2 = Date.now();
        while(Date.now() <= x2 + 17) {
            ;
        }
    </script>
    <div style="display:none;">
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b>b
    </div>
    <script>
        let x3 = Date.now();
        while(Date.now() <= x3 + 2000) {
            ;
        }
    </script>
    <div>第3个内容</div>
    <script>
        let x4 = Date.now();
        while(Date.now() <= x4 + 17) {
            ;
        }
    </script>
    <div style="display:none;">
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b>b
    </div>
    <script>
        let x5 = Date.now();
        while(Date.now() <= x5 + 2000) {
            ;
        }
    </script>
    <div>第4个内容</div>
</body>
</html>
```

看代码逻辑，页面表现应该是下面的顺序
1. 页面先解析`<div>第1个内容</div>`；
2. 通过`渲染片段`触发渲染，这时候应该在屏幕上看到`<div>第1个内容</div>`；
3. js阻塞2000毫秒(2秒)；
4. 页面解析`<div>第2个内容</div>`；
5. 通过`渲染片段`触发渲染，这时候应该在屏幕上看到`<div>第2个内容</div>`；
6. 之后应该是过2秒看到`<div>第3个内容</div>`，再过2秒看到`<div>第4个内容</div>`；

这里录屏看看实际效果是否跟推断一致：

[![demo1-gif](https://github.com/keiouks/paint_holding_analyze/blob/main/img/demo1-first-render-delay.gif?raw=true)](https://github.com/keiouks/paint_holding_analyze/blob/main/img/demo1-first-render-delay.gif?raw=true)

实际效果是，当我输入网址之后，页面没有马上显示任何内容，而是在2秒后`<div>第1个内容</div>`和`<div>第2个内容</div>`一起显示出来，之后每隔2秒显示`<div>第3个内容</div>`和`<div>第4个内容</div>`。

为什么`<div>第1个内容</div>`没有马上显示出来。

### 查看performance

录performance看看整个过程的情况：[demo1的performance文件链接](https://github.com/keiouks/paint_holding_analyze/blob/main/performance/demo1-first-render-delay.json)。可以下载这个performance文件然后在自己本地chrome加载看。

整体上可以看到三段JavaScript的执行，每段耗时2秒，其实就是页面逻辑中那三段阻塞2秒的js逻辑，选中第一段js，看到Script位置是`1:25:13`，最前面的数字1是页面路径，25和13是代码的行号和列号，25行13列恰好就是第一个阻塞2秒的script标签内侧位置。

[![demo1-performance-all](https://github.com/keiouks/paint_holding_analyze/blob/main/img/demo1-performance-all.png?raw=true)](https://github.com/keiouks/paint_holding_analyze/blob/main/img/demo1-performance-all.png?raw=true)

放大第一段2秒script前面的逻辑，可以看到，第一个红色箭头指向`<div>第1个内容</div>`之后的17毫秒js，第二个红色箭头指向第一段2秒js，中间蓝色箭头处发生了`Recalculate Style`,`Layout`以及`Paint`，表明触发了渲染，但`<div>第1个内容</div>`并没有在这时候显示出来，注意蓝色箭头指向第一次渲染之后还有虚线竖线，标记了`FCP`字样。

[![demo1-performance-before-first-2-second](https://github.com/keiouks/paint_holding_analyze/blob/main/img/demo1-performance-before-first-2-second.png?raw=true)](https://github.com/keiouks/paint_holding_analyze/blob/main/img/demo1-performance-before-first-2-second.png?raw=true)

放大第一段2秒之后，第二段2秒之前，这之间的逻辑看看。第一个红色箭头指向的是`<div>第2个内容</div>`之后的17毫秒js，第二个红色箭头指向的是第二段2秒js，蓝色箭头指向的地方同样是执行了渲染逻辑，比起上一次渲染不同的是，这次多执行了一个`Composite Layers`，这个调用意味着blink调用了合成层并把内容显示在屏幕上，这也是为什么第一次渲染没有显示任何东西，直到2秒后才显示前两个内容，之后`<div>第3个内容</div>`和`<div>第4个内容</div>`的渲染都有触发`Composite Layers`。

[![demo1-performance-between-first-2-and-second-2](https://github.com/keiouks/paint_holding_analyze/blob/main/img/demo1-performance-between-first-2-and-second-2.png?raw=true)](https://github.com/keiouks/paint_holding_analyze/blob/main/img/demo1-performance-between-first-2-and-second-2.png?raw=true)

为什么第一次渲染没有触发`Composite Layers`？

### 查看tracing

录tracing看看整个过程逻辑：[demo1的tracing文件链接](https://github.com/keiouks/paint_holding_analyze/blob/main/tracing/trace_demo1.json.gz)。可以下载该tracing文件并在本地chrome加载看。

整体上看，同样是有3段2秒耗时的js，每一段同样能看到执行的script开始处的行号和列号。

[![demo1-tracing-all](https://github.com/keiouks/paint_holding_analyze/blob/main/img/demo1-tracing-all.png?raw=true)](https://github.com/keiouks/paint_holding_analyze/blob/main/img/demo1-tracing-all.png?raw=true)

放大第一个2秒js前面的逻辑，第一个红色箭头指向`<div>第1个内容</div>`之后的17毫秒js，第二个红色箭头指向第一段2秒耗时js，中间红框就是第一次渲染的过程，蓝色箭头指向的浅蓝色竖线就是FCP触发位置。

[![demo1-tracing-before-first-2](https://github.com/keiouks/paint_holding_analyze/blob/main/img/demo1-tracing-before-first-2.png?raw=true)](https://github.com/keiouks/paint_holding_analyze/blob/main/img/demo1-tracing-before-first-2.png?raw=true)

放大渲染部分接近最后的位置，也就是浅蓝色竖线附近的逻辑，会看到红色箭头处有一个调用，耗时很短，该调用是`EarlyOut_DeferCommit_InsideBeginMainFrame`，表示推迟commit，commit就是blink把渲染后的内容提交给合成层，然后显示上屏，DeferCommit意味着没有commit，推迟了上屏。

[![demo1-tracing-first-render-defercommit](https://github.com/keiouks/paint_holding_analyze/blob/main/img/demo1-tracing-first-render-defercommit.png?raw=true)](https://github.com/keiouks/paint_holding_analyze/blob/main/img/demo1-tracing-first-render-defercommit.png?raw=true)

直接放大看第一段2秒后的第二次渲染的逻辑，第二次渲染最后调用了`ProxyMain::BeginMainFrame::commit`，因此这里才提交了渲染内容到合成层并显示出内容。

[![demo1-tracing-second-render-commit](https://github.com/keiouks/paint_holding_analyze/blob/main/img/demo1-tracing-second-render-commit.png?raw=true)](https://github.com/keiouks/paint_holding_analyze/blob/main/img/demo1-tracing-second-render-commit.png?raw=true)

为什么第一次渲染会推迟commit，而第二次渲染又会commit ？

### 源码分析

直接看chrome源码。看渲染的主要方法`ProxyMain::BeginMainFrame`，这里贴的是chrome 101版本的代码([代码链接](https://source.chromium.org/chromium/chromium/src/+/refs/tags/101.0.4951.74:cc/trees/proxy_main.cc;l=127;bpv=0;bpt=0))。主要关心`skip_commit`变量最终是true还是false，看贴出的代码，最终的if判断`skip_commit`是true时，就会跳过commit，导致这次渲染没有交给合成层，没有显示到屏幕上。最新版本的代码`skip_commit`变量的值可能加入了更多的判断条件，但一般都不属于初始化展示这个case。

```cpp
void ProxyMain::BeginMainFrame(......) {
    ......
    // If main frame updates and commits are deferred, skip the entire pipeline.
    if (defer_main_frame_update_) {
      TRACE_EVENT_INSTANT0("cc", "EarlyOut_DeferCommit",
                         TRACE_EVENT_SCOPE_THREAD);
      ......
      return;
    }
    ......
    bool commit_timeout = false;
    if (IsDeferringCommits() && base::TimeTicks::Now() > commits_restart_time_) {
        StopDeferringCommits(ReasonToTimeoutTrigger(*paint_holding_reason_));
        commit_timeout = true;
    }
    bool skip_commit = IsDeferringCommits();
    bool scroll_and_viewport_changes_synced = false;

    if (!skip_commit) {
        // Synchronizes scroll offsets and page scale deltas (for pinch zoom) from
        // the compositor thread to the main thread for both cc and and its client
        // (e.g. Blink). Do not do this if we explicitly plan to not commit the
        // layer tree, to prevent scroll offsets getting out of sync.
        layer_tree_host_->ApplyCompositorChanges(
            begin_main_frame_state->commit_data.get());
        scroll_and_viewport_changes_synced = true;
    }
    ......
    skip_commit |= defer_main_frame_update_ || IsDeferringCommits();
    skip_commit |= begin_main_frame_state->begin_frame_args.animate_only;
    if (skip_commit) {
        current_pipeline_stage_ = NO_PIPELINE_STAGE;
        layer_tree_host_->DidBeginMainFrame();
        TRACE_EVENT_INSTANT0("cc", "EarlyOut_DeferCommit_InsideBeginMainFrame",
                             TRACE_EVENT_SCOPE_THREAD);
        ......
        return;
    }
    ......
}
```

从源码可见，`skip_commit`从声明到最后的if判断，一共被3个条件赋值过，只要有一个条件是true，最终结果就是true，三个条件分别是：

- IsDeferringCommits()；
- defer_main_frame_update_；
- begin_main_frame_state->begin_frame_args.animate_only；

先看后面两个条件

**begin_main_frame_state->begin_frame_args.animate_only**

跟初始时推迟上屏无关，应该是`headless模式`下显示配置成true才会拿到true，正常初始化时总是false。

**defer_main_frame_update_**

看贴出的代码，该变量在`skip_commit`声明之前就有被if判断，若它是true，前面就会return，并触发`EarlyOut_DeferCommit`事件，但从tracing上可以看到，首次渲染`<div>第1个内容</div>`并推迟上屏时，没有触发`EarlyOut_DeferCommit`事件，而是触发了`EarlyOut_DeferCommit_InsideBeginMainFrame`事件，该事件前面tracing分析时提到过，从源码可见`EarlyOut_DeferCommit_InsideBeginMainFrame`事件在`skip_commit`是true时触发，从中可以推断出`defer_main_frame_update_`是false。

事实上，`EarlyOut_DeferCommit`事件目前只看到在html开始解析之前会触发，而html解析之前根本没有内容可以渲染。

### IsDeferringCommits()

初始化时的推迟上屏主要看该方法的返回值。而它的返回值是：

```cpp
return paint_holding_reason_.has_value();
```

这里表示`paint_holding_reason_`变量有值就会推迟，变量名其实就指明了chrome的[**Paint Holding特性**](https://developer.chrome.com/blog/paint-holding/)，这个变量是一个枚举值，初始化默认值是空。网页初始化时根据某个判断来确定是否要给这个变量赋值，[代码链接在这里](https://source.chromium.org/chromium/chromium/src/+/refs/tags/101.0.4951.74:third_party/blink/renderer/core/frame/local_frame_view.cc;l=4611;drc=b8524150039182faf7988e9478a9eff89728ac03)。

这里贴出该代码片段：

```cpp
  // Determine if we want to defer commits to the compositor once lifecycle
  // updates start. Doing so allows us to update the page lifecycle but not
  // present the results to screen until we see first contentful paint is
  // available or until a timer expires.
  // This is enabled only when the document loading is regular HTML served
  // over HTTP/HTTPs. And only defer commits once. This method gets called
  // multiple times, and we do not want to defer a second time if we have
  // already done so once and resumed commits already.
  if (WillDoPaintHoldingForFCP()) {
    have_deferred_main_frame_commits_ = true;
    chrome_client.StartDeferringCommits(
        GetFrame(), base::Milliseconds(kCommitDelayDefaultInMs),
        cc::PaintHoldingReason::kFirstContentfulPaint);
  }
```

if里面的`chrome_client.StartDeferringCommits`方法调用会给`paint_holding_reason_`变量赋值(`chrome_client.StartDeferringCommits`方法没有直接给`paint_holding_reason_`变量赋值，而是会调用另一个类的`StartDeferringCommits`方法，而另一个类的`StartDeferringCommits`方法又会继续调用别的类的`StartDeferringCommits`方法，最后会调用`ProxyMain::StartDeferringCommits`然后给`paint_holding_reason_`变量赋值)。

先看代码上的注释，大概意思是：当生命周期开始更新时，这里决定是否要推迟commit到合成线程，这样做会让页面继续更新，比如渲染时dom会有内容，但不会显示上屏，直到第一次看到FCP已经被触发，或者直到超出某个时间。推迟行为只会发生在http/https返回的普通html，并且只会推迟一次。

当这里的if判断是true时，后续的渲染会被推迟显示上屏，但触发FCP之后就不再推迟，或者超过某个时间之后不再推迟，这里的某个时间就是`chrome_client.StartDeferringCommits`调用时的第二个参数，它的值是写死的500毫秒。

再看上面贴的`void ProxyMain::BeginMainFrame`方法的源码，中间一个逻辑是这样：

```cpp
    bool commit_timeout = false;
    if (IsDeferringCommits() && base::TimeTicks::Now() > commits_restart_time_) {
        StopDeferringCommits(ReasonToTimeoutTrigger(*paint_holding_reason_));
        commit_timeout = true;
    }
```

这里判断从调用`chrome_client.StartDeferringCommits`方法设置推迟上屏开始，到某次渲染，是否已经超出了设定的500毫秒，如果是，就调用`StopDeferringCommits`方法清空`paint_holding_reason_`变量的值，从而导致`IsDeferringCommits()`返回false。

### WillDoPaintHoldingForFCP

现在看该方法，它的返回值决定是否需要执行[**Paint Holding特性**](https://developer.chrome.com/blog/paint-holding/)。但它的返回值又依赖另一个方法调用，为了减少篇幅，忽略不重要的信息，直接看关键代码。[代码链接在这里](https://source.chromium.org/chromium/chromium/src/+/refs/tags/101.0.4951.74:third_party/blink/renderer/core/loader/document_loader.cc;l=2634;drc=b8524150039182faf7988e9478a9eff89728ac03;bpv=1;bpt=1)

```cpp
  // The PaintHolding feature defers compositor commits until content has
  // been painted or 500ms have passed, whichever comes first. The additional
  // PaintHoldingCrossOrigin feature allows PaintHolding even for cross-origin
  // navigations, otherwise only same-origin navigations have deferred commits.
  // We also require that this be an html document served via http.
  if (base::FeatureList::IsEnabled(blink::features::kPaintHolding) &&
      IsA<HTMLDocument>(document) && Url().ProtocolIsInHTTPFamily() &&
      (is_same_origin_initiator ||
       base::FeatureList::IsEnabled(
           blink::features::kPaintHoldingCrossOrigin))) {
    document->SetDeferredCompositorCommitIsAllowed(true);
  } else {
    document->SetDeferredCompositorCommitIsAllowed(false);
  }
```

这里的代码注释又再说明了一次[**Paint Holding特性**](https://developer.chrome.com/blog/paint-holding/)的特点，但这里有一句话比较关键：

> The additional PaintHoldingCrossOrigin feature allows PaintHolding even for cross-origin navigations.

意思是另外的`PaintHoldingCrossOrigin`特性甚至允许[**Paint Holding特性**](https://developer.chrome.com/blog/paint-holding/)在跨域时生效。

这里if判断为true时，给关键变量设置成true，否则设置成false，这直接决定`WillDoPaintHoldingForFCP`方法的返回值。

列出if判断的条件：

- IsA<HTMLDocument>(document)：当前文档是否是一个html文档，当然是；
- Url().ProtocolIsInHTTPFamily()：是否通过http协议族请求访问，当然是；
- base::FeatureList::IsEnabled(blink::features::kPaintHolding)：是否支持`kPaintHolding`特性，较新版本的浏览器都支持，当然是；
- is_same_origin_initiator \|\| base::FeatureList::IsEnabled(blink::features::kPaintHoldingCrossOrigin)：是否同源跳转或者支持`kPaintHoldingCrossOrigin`特性，直接打开的不是同源跳转，但较新版本目前都支持`kPaintHoldingCrossOrigin`特性；

基本上，目前新版本chrome都会支持[**Paint Holding特性**](https://developer.chrome.com/blog/paint-holding/)，不管是否跨域打开网页，但不排除以后会有变化。

### 如何加快上屏速度

[**Paint Holding特性**](https://developer.chrome.com/blog/paint-holding/)会推迟上屏直到FCP或者500毫秒之后的渲染。500毫秒不太愿意等，看如何触发FCP。

FCP的意思是有实质内容的渲染，可以自己去查一下FCP的准确定义。

回想第一个demo，`<div>第1个内容</div>`渲染后没有上屏，但它的渲染触发了FCP，再贴一次当时第一次渲染的performance截图：

[![demo1-performance-before-first-2-second](https://github.com/keiouks/paint_holding_analyze/blob/main/img/demo1-performance-before-first-2-second.png?raw=true)](https://github.com/keiouks/paint_holding_analyze/blob/main/img/demo1-performance-before-first-2-second.png?raw=true)

蓝色箭头指向第一次选后的位置，渲染后接着触发了FCP，在虚线竖线中标明了位置。这里`<div>第1个内容</div>`的渲染就是一个实质内容的渲染。第一个`渲染片段`中的b标签因为看不见，所以不是实质性内容。

其实从源码上可以找到在FCP触发时，`paint_holding_reason_`变量的值被清空，从而导致`IsDeferringCommits()`返回false，为了减少文章篇幅这里没有展开看这段代码。

要想`<div>第1个内容</div>`快速显示，只需要在它渲染之前先触发一次FCP就行，我们可以在`<div>第1个内容</div>`之前放一个`<div>.</div>`和一个`渲染片段`，这样，`<div>.</div>`被渲染时就会触发FCP，然后到`<div>第1个内容</div>`渲染时就会显示上屏。

这里实现第二个demo，[页面代码链接在这](https://github.com/keiouks/paint_holding_analyze/blob/main/views/index2.html):

```html
<!DOCTYPE html>
<html>
<head>
    <title>渲染测试</title>
</head>
<body>
    <div>.</div>
    <script>
        let y = Date.now();
        while(Date.now() <= y + 17){
            ;
        }
    </script>
    <div style="display:none;">
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b>b
    </div>
    <div>第1个内容</div>
    <script>
        let x = Date.now();
        while(Date.now() <= x + 17){
            ;
        }
    </script>
    <div style="display:none;">
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b>b
    </div>
    <script>
        let x1 = Date.now();
        while(Date.now() <= x1 + 2000) {
            ;
        }
    </script>
    <div>第2个内容</div>
    <script>
        let x2 = Date.now();
        while(Date.now() <= x2 + 17) {
            ;
        }
    </script>
    <div style="display:none;">
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b>b
    </div>
    <script>
        let x3 = Date.now();
        while(Date.now() <= x3 + 2000) {
            ;
        }
    </script>
    <div>第3个内容</div>
</body>
</html>
```

去掉了`<div>第4个内容</div>`，减短了页面代码量，录屏看效果：

[![demo2-first-render-commit](https://github.com/keiouks/paint_holding_analyze/blob/main/img/demo2-first-render-commit.gif?raw=true)](https://github.com/keiouks/paint_holding_analyze/blob/main/img/demo2-first-render-commit.gif?raw=true)

可以看到请求网页后，`<div>第1个内容</div>`给人的感觉就是马上显示，但我们并不想看到那个`<div>.</div>`，实际开发中我们可以加一些逻辑让最开始的那个点消失，也可以用`<div>第1个内容</div>`(实际可能是一堆ssr内容)覆盖在那个点上面。

### 验证500毫秒超时上屏

源码逻辑上判断，超过了500毫秒就不再推迟上屏，那我们可以把`<div>第1个内容</div>`之后的阻塞17毫秒改成阻塞500毫秒试试。

我们实现第三个demo，[页面代码链接在这里](https://github.com/keiouks/paint_holding_analyze/blob/main/views/index3.html):

```html
<!DOCTYPE html>
<html>
<head>
    <title>渲染测试</title>
</head>
<body>
    <div>第1个内容</div>
    <script>
        let x = Date.now();
        while(Date.now() <= x + 500){
            ;
        }
    </script>
    <div style="display:none;">
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b>b
    </div>
    <script>
        let x1 = Date.now();
        while(Date.now() <= x1 + 2000) {
            ;
        }
    </script>
    <div>第2个内容</div>
    <script>
        let x2 = Date.now();
        while(Date.now() <= x2 + 17) {
            ;
        }
    </script>
    <div style="display:none;">
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b><b>b</b>
        <b>b</b><b>b</b>b
    </div>
    <script>
        let x3 = Date.now();
        while(Date.now() <= x3 + 2000) {
            ;
        }
    </script>
    <div>第3个内容</div>
</body>
</html>
```

直接录视屏看看效果：

[![demo3-after-500-commit](https://github.com/keiouks/paint_holding_analyze/blob/main/img/demo3-after-500-commit.gif?raw=true)](https://github.com/keiouks/paint_holding_analyze/blob/main/img/demo3-after-500-commit.gif?raw=true)

虽然等500毫秒后显示`<div>第1个内容</div>`感觉也没有很慢，但能明显感受到它是等了一会才显示的，而不像第二个demo那样立马显示。

可以录一个performance看看，500毫秒后，即使没有触发FCP，`<div>第1个内容</div>`渲染后也会触发`Composite Layers`上屏。

推迟上屏的分析结束。
