# JavaScript 事件循环

最近看了很多关于事件循环的资料，感觉有点晕，赶紧整理整理，过几天忘了再来看。

## 事件循环

代码运行时分为**同步任务**和**异步任务**两种。我个人对于同步和异步的理解就是：*能立即获得结果的任务就是同步任务，需要通过回调函数才能执行的是异步任务。*

如果程序中都是同步任务，就按照从上到下的先后顺序执行主线程代码，运行结束了程序也就结束了，此时没有事件循环。

只有代码执行过程中遇到了异步任务才会触发事件循环，当异步任务全部完成，异步任务队列清空时，事件循环结束。


### 异步任务类型

很多文章中都介绍了**宏任务队列(macrotask)**和**微任务队列(microtask)**，只需要理解为两个队列，用来存放不同类型的任务就行了。

_*规范中没有提到macrotask，只有task，可以理解为task是规范，而macrotask是具体的实现，其实是指的相同的东西*_

>(macro)task主要包含：script(整体代码)、setTimeout、setInterval、I/O、UI交互事件、setImmediate(node环境)
>microtask主要包含：Promise、MutaionObserver、process.nextTick(node环境)


### 事件循环的控制流程

规范定义的主要步骤：

1. 执行一个task
2. 检查microtask队列，有任务就全部执行，直到清空
3. render(渲染可能不是每个事件循环都执行，具体过程中会多次事件循环后将修改一次性渲染)

整体实现流程图如下：

![引用自资料](https://pic4.zhimg.com/80/v2-fdd9322a0cabafa7d3461e5d25718586_hd.jpg)

文字版：

* 执行同步任务，遇到异步任务根据类型分别插入Task Queue和Microtask Queue
* 同步任务执行完成，检查Microtask Queue，存在任务全部执行直到清空
* 下一轮循环开始，执行第一个Task，完成后检查Microtask Queue，存在任务全部执行至清空
* 重复上一步知道Task全部执行

## 实践一下

#### 情景1

```
console.log(1)

setTimeout(()=>{
    console.log(2)
})

new Promise(resolve=>{
    console.log(3)
    resolve()
}).then(()=>{
    console.log(4)
})

Promise.resolve().then(()=>{
    console.log(5)
})

console.log(7)
```

**分析过程**

1. 执行同步任务，打印1、3、7
2. 检查Microtask Queue，有两个Promise.then，按顺序执行，打印4、5
3. 第二轮循环，执行Task Queue的第一个任务，打印2

#### 情景2

```
console.log(1)

setTimeout(() => {
    console.log(2)
    new Promise(resolve => {
        console.log(4)
        resolve()
    }).then(() => {
        console.log(5)
    })
})

new Promise(resolve => {
    console.log(7)
    resolve()
}).then(() => {
    console.log(8)
})

setTimeout(() => {
    console.log(9)
    new Promise(resolve => {
        console.log(11)
        resolve()
    }).then(() => {
        console.log(12)
    })
})
```
**分析过程**

1. 执行同步任务，打印1、7
2. 清空Microtask，打印8
3. 执行第一个Task，打印2、4
4. 清空Microtask，打印5
5. 执行第二个Task，打印9、11
6. 清空Microtask，打印12

#### 情景3
```
console.log(1)

setTimeout(() => {
    console.log(2)
    new Promise(resolve => {
        console.log(4)
        resolve()
    }).then(() => {
        console.log(5)
    })
    process.nextTick(() => {
        console.log(3)
    })
})

new Promise(resolve => {
    console.log(7)
    resolve()
}).then(() => {
    console.log(8)
})

process.nextTick(() => {
    console.log(6)
})
```
按照之前的套路分析理论上应该输出顺序是：1、7、8、6、2、4、5、3。
实际在node环境下运行结果是：1、7、6、8、2、4、3、5。
原因是process.nextTick总是在Microtask的最前面，执行优先级比其他Microtask高。

#### 情景4

```
console.log(1)

setTimeout(() => {
    console.log(2)
    new Promise(resolve => {
        console.log(4)
        resolve()
    }).then(() => {
        console.log(5)
    })
    process.nextTick(() => {
        console.log(3)
    })
})

new Promise(resolve => {
    console.log(7)
    resolve()
}).then(() => {
    console.log(8)
})

process.nextTick(() => {
    console.log(6)
})

setTimeout(() => {
    console.log(9)
    process.nextTick(() => {
        console.log(10)
    })
    new Promise(resolve => {
        console.log(11)
        resolve()
    }).then(() => {
        console.log(12)
    })
})
```
按照套路分析，理论上：
第一轮输出：1、7、6、8
第二轮输出：2、4、3、5
第三轮输出：9、11、10、12

实际在node中运行情况是：1、7、6、8、2、4、9、11、3、10、5、12

**这是因为node的事件循环方式和规范的有些区别，在node中，Task是一次性全部执行的**

node的Event Loop实现：
![](http://voidcanvas.com/wp-content/uploads/2018/02/nodejs-event-loop-workflow.png)

这里引用一下阮一峰大大的翻译
>（1）timers

>这个是定时器阶段，处理setTimeout()和setInterval()的回调函数。进入这个阶段后，主线程会检查一下当前时间，是否满足定时器的条件。如果满足就执行回调函数，否则就离开这个阶段。

>（2）I/O callbacks

>除了以下操作的回调函数，其他的回调函数都在这个阶段执行。

> setTimeout()和setInterval()的回调函数
> setImmediate()的回调函数
> 用于关闭请求的回调函数，比如socket.on('close', ...)

>（3）idle, prepare

>该阶段只供 libuv 内部调用，这里可以忽略。

>（4）Poll

>这个阶段是轮询时间，用于等待还未返回的 I/O 事件，比如服务器>的回应、用户移动鼠标等等。

>这个阶段的时间会比较长。如果没有其他异步任务要处理（比如到期的定时器），会一直停留在这个阶段，等待 I/O 请求返回结果。

>（5）check

>该阶段执行setImmediate()的回调函数。

>（6）close callbacks

>该阶段执行关闭请求的回调函数，比如>socket.on('close', ...)。

所以在node中分析过程应该是：

1. 执行同步任务，打印1、7
2. 清空Microtask，Process.next优先，打印6、8
3. 执行所有Task，打印2、4、9、11
4. 清空Microtask，打印3、10、5、12

#### 情景5

```
setTimeout(()=>{
    console.log(1)
})
setImmediate(()=>{
    console.log(2)
})
```
在node中以上代码输出是不能保证先后顺序的，因为setTimeout虽然没有传第二个参数，但是实际上node中第二个参数的最小值是1，所以

```
setTimeout(()=>{ console.log(1) }, 0)
setTimeout(()=>{ console.log(1) }, 1)
```

两者是等价的，在运行过程中取决于系统状态，在事件循环开始时，如果到了1ms，先执行setTimeout，没到1ms则跳过timer阶段，执行setImmediate。

要保证setImmediate优先于setTimeout，只能在同一个I/O操作的回调中执行，就像如下代码：

```
const fs = require('fs')

fs.readFile('/path/to/file', () => {
    setTimeout(() => {
        console.log('timeout')
    })
    setImmediate(() => {
        console.log('immediate')
    })
})
```
这里事件循环已经开始，此时正处于I/O callbacks阶段，之后会优先进入check阶段再进入timer阶段，所以setImmediate一定比setTimeout优先。

#### 情景6
还没完。。。

```
new Promise(resolve => {
    resolve(1);
    Promise.resolve().then(() => console.log(2));
    console.log(4)
}).then(t => console.log(t));
console.log(3);
```
这里还有一道题，是阮老师在其推特上放的，根据上面这么多的经验累积，我很迅速的写出了自己的答案：4、3、1、2
然而呵呵。。。
无论是chrome还是node环境都给出了4、3、2、1的答案。
这部分我还没研究明白，只能暂时认为Promise定义阶段产生的Microtask总是比自己的then要优先级高。

## 参考

* [Event Loop的规范和实现](https://zhuanlan.zhihu.com/p/33087629)
* [这一次，彻底弄懂 JavaScript 执行机制](https://juejin.im/post/59e85eebf265da430d571f89)
* [Node 定时器详解](http://www.ruanyifeng.com/blog/2018/02/node-event-loop.html)
* [从一道题浅说 JavaScript 的事件循环](https://juejin.im/entry/5a8bc3215188257a856f4b2b?utm_source=gold_browser_extension)

