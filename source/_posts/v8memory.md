title: nodejs内存控制
date: 2019-02-20 16:32:00
categories: scliuyang
tags:
- node

---
主要给大家带来nodejs中v8引擎的内存控制以及泄漏排查
<!--more-->

# 为什么需要研究内存控制

过去很长一段时间内，JavaScript开发者很少会遇到对内存精确控制的场景，也缺乏控制手段。在浏览器里如果内存占用过多基本上用户都会等不及刷新网页，所以基本无需关注内存泄漏问题。
但是随着nodejs的发展，基于无阻塞、事件驱动建立的node服务具有内存消耗低的优点，非常适合处理海量的网络请求。在海量请求的前提下，开发者就需要关注一些平常不会形成影响的问题，比如一个简单字符串内存泄漏，在海量请求下就会导致内存大量溢出导致应用崩溃。
所以在服务端领域，内存控制正是在海量请求和长时间运行的前提下需要着重探讨的。

# v8的垃圾回收机制与内存限制

## v8的内存限制

v8的内存在32位系统下限制为0.7GB,64位下为1.4GB。为什么会有这个限制呢？首先V8设计之初就是为浏览器考虑的，在浏览器的开发场景下这套设计绰绰有余足以胜任前端页面中所有的需求。深层次的原因就是V8的垃圾回收机制的限制，以1.5GB的垃圾为例，V8做一次小的垃圾回收约50毫秒以上，做一次非增量式的垃圾回收甚至需要1秒以上。在垃圾回收中JavaScript线程暂停执行，在这样的时间花费下不仅仅后端无法接受，前端也无法接受。因此做出了内存限制这个决定。

当然这个限制也可以通过参数打开，在node启动时传递如下参数。注意只在启动时生效，一旦启动不能修改

```javascript
node --max-old-space-size=1700 test.js // 单位为MB
node --max-new-space-size=2014 test.js // 单位为kb
```

## V8的垃圾回收

### V8的内存分代
在V8中，主要讲内存分为新生代和老生代。新生代中的对象存活时间比较短，老生代中的对象存活时间较长或常驻内存对象。

![内存分代](https://p0.meituan.net/dpnewvc/382aa42eaeb4c6e54a13cd047f4fda1321204.png)

### 新生代内存回收机制Scavenge算法

`Scavenge`实际采用的复制方式实现的垃圾回收算法，他将内存空间一分为二，一半称之为from空间,另一半称为to空间。当我们分配内存时先在from空间分配。当开始内存回收时，将存活对象复制到to空间中，释放from空间内存。完成后两个空间角色互换。

`Scavenge`的缺点是只能利用一半的内存，优点是只复制存活对象。由于新生代内存空间大部分都是非存活对象所以`Scavenge`算法在时间效率上有优异的表现。是典型的空间换时间算法。
当一个对象在新生代空间中多次回收后依然存在，则被认定为生命周期较长的对象，迁移至老生代内存空间中。

### 老生代内存回收机制 Mark-Sweep&Mark-Compact

老生代中的对象就不适合使用`Scavenge`，因为会有两个问题：一个是存活对象太多，复制效率低。第二是浪费一半空间

#### Mark-Sweep标记清除

标记清除就是将内存回收分为标记和清除两个阶段，首先遍历所有的对象并标记活着的对象，接着清理所有的死亡对象。因为老生代内存空间只有小部分是死亡对象，所以标记清除算法将比`Scavenge`算法有着更高的效率。

标记清除算法最大的问题是在一次标记清除回收后，内存空间会出现不连续的状态，可能会造成后续分配大对象无可用内存的情况。

#### Mark-Compact标记整理

为了解决上面算法带来的内存碎片问题，`Mark-Compact`被提了出来。他最大的差别就是在标记出对象死亡后，在整理内存时将活着的对象往一段移动，移动完毕后直接清理掉边界外的内存。

| 回收算法     | Mark-Sweep | Mark-Compact | Scavenge         |
| ------------ | ---------- | ------------ | ---------------- |
| 速度         | 中等       | 最慢         | 最快             |
| 空间开销     | 少(有碎片) | 少(无碎片)   | 双倍空间(无碎片) |
| 是否移动对象 | 否         | 是           | 是               |

# 查看内存使用状况并打印内存快照

```javascript
process.memoryUsage()
{ rss: 22757376,
  heapTotal: 7684096,
  heapUsed: 5036952,
  external: 8734 }
```

heapTotal 和 heapUsed 代表V8的内存使用情况。 external代表V8管理的，绑定到Javascript的C++对象的内存使用情况。 rss, 驻留集大小, 是给这个进程分配了多少物理内存(占总分配内存的一部分) 这些物理内存中包含堆，栈，和代码段(比如Buffer分配的内存)。

对象，字符串，闭包等存于堆内存。 变量存于栈内存。

## 利用node-heapdump排查泄漏

```javascript
const http = require('http');
const heapdump = require('heapdump');
const url = require('url');
const path = require('path');

const leak = [];
http
  .createServer((req, res) => {
    const urlObj = url.parse(req.url);

    if (urlObj.path === '/leak') {
      for (let i = 0; i < 100000; i++) {
        leak.push('leak');
      }
    } else if (urlObj.path === '/heapdump') {
      
      heapdump.writeSnapshot(
        path.resolve(__dirname, `${Date.now()}.heapsnapshot`),
        function(err, filename) {
          console.log('dump written to', filename, err);
        },
      );
    }
    res.writeHead(200, 'success', { 'content-type': 'text/html;charset=utf-8;' });
    res.end('成功');
  })
  .listen(3000);

```
首先访问`/heapdump`然后多次访问`/leak`，接着再次访问`/heapdump`。这时候会生成两个内存快照。打开chrome dev tools。选择Memory，load生成的数据。得到比较图如下

![内存比较](https://p0.meituan.net/dpnewvc/0db2a4798109a46b44cac2e30ff0388461957.png)

可以明显的发现Array数组了leak字符串大量增加,由此查找内存泄漏点

## 生产环境和pm2配合

`pm2 --max-memory-restart 300M`，当内存使用超过300M时，pm2会发送一个`SIGINT`信号给进程这里注意默认的`kill_timeout`超时为1600ms。这时使用如下代码判断内存是否存在溢出情况并打印内存快照

```javascript
const heapdump = require('heapdump');
const path = require('path');

process.on('SIGINT', () => {
  const mem = process.memoryUsage();
  const usaged = mem.heapUsed / 1024 / 1024;
  if (usaged >= 300) {
    global.gc(); // 触发一次gc，减少干扰

    heapdump.writeSnapshot(path.resolve(__dirname, `${Date.now()}.heapsnapshot`), function(
      err,
      filename,
    ) {
      console.log('dump written to', filename, err);
      process.exit(0);
    });
  } else {
    process.exit(0);
  }
});


```

