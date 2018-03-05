# JavaScript高级技巧

## 高级函数

### 一、安全的类型检测

```
Object.prototype.toString.call()
```
这个方法可以返回目标到底是什么类型：

```
var toString = Object.prototype.toString;

toString.call(new Date); // [object Date]
toString.call(new String); // [object String]
toString.call(Math); // [object Math]

//Since JavaScript 1.8.5
toString.call(undefined); // [object Undefined]
toString.call(null); // [object Null]
```

>以上代码摘自MDN

### 二、作用域安全的构造函数

一般来说构造函数我们都这样写：

```
function Person(name, age, job) {
    this.name = name;
    this.age = age;
    this.job = job;
}
```
看起来没什么问题，但是保不住有人乱用呀，要是有人这样用：

```
var person = Person('json', 29, 'Engineer');
```
这里没有使用new操作符，所以构造函数中this的指向是window对象，于是:

```
console.log(window.name) // 'json'
console.log(window.age) // 29
console.log(window.job) // 'Engineer'
```

为了确保即使没有使用new操作符依旧不会出现上述的this指向乱跑的问题，我们可以这样写：

```
function Person(name, age, job) {
    if (this instanceof Person) {
        this.name = name;
        this.age = age;
        this.job = job;
    } else {
        return new Person(name, age, job);
    }
}
```
这样就能确保即使在没有正确使用构造函数的情况下也不会污染全局环境了。

#### 但是！！！

这样写虽然能确保构造函数的正确使用，如果想要继承这个对象就需要注意了，看代码：

```
function PersonInfo (height, weight) {
    Person.call(this, 'James', 20, 'college student');
    this.height = height;
    this.weight = weight;
}
var JamesInfo = new PersonInfo(175, 130);
console.log(JamesInfo.height); // 175
console.log(JamesInfo.weight); // 130
console.log(JamesInfo.name); // undefined
console.log(JamesInfo.age); // undefined
console.log(JamesInfo.job); // undefined
```
如果采用这种寄生式的继承方式会出现问题，因为Person构造函数是锁定this指向的，此时由于PersonInfo的this不是Person的实例，所以Person构造函数会创建一个新对象并且返回，这里没有用到返回值，于是就无法继承。
如果想要采用寄生式继承，可以这样写

```
PersonInfo.prototype = new Person();
```
这样在调用Person.call()的时候就不会出现this不是Person实例的情况了。

### 三、惰性加载函数

很多时候我们会写一些判断代码执行环节的方法，例如：

```
var addEvent = function(el, type, fn, capture) {
    if (window.addEventListener) {
        el.addEventListener(type, function(e) {
            fn.call(el, e);
        }, capture);
    } else if (window.attachEvent) {
        el.attachEvent("on" + type, function(e) {
            fn.call(el, e);
        });
    } 
};
```
于是每次调用addEvent方法的时候都会进行一次ifelse判断，但是这其实是没有必要的，只要第一次判断后面的都不需要进行判断了，环境又不会变化。
所以可以进行改造：

```
var addEvent = (function () {
    if (window.addEventListener) {
        return function (el, type, fn, capture) {
            el.addEventListener(type, function(e) {
                fn.call(el, e);
            }, capture);
        }
        
    } else if (window.attachEvent) {
        return function (el, type, fn, capture) {
            el.attachEvent("on" + type, function(e) {
                fn.call(el, e);
            });
        }
    } 
})();
```
这样子就只会在第一次执行addEvent方法的时候进行一次ifelse判断了。
还有一种写法：

```
var addEvent = function(el, type, fn, capture) {
    if (window.addEventListener) {
        addEvent = function(el, type, fn, capture) {
            el.addEventListener(type, function(e) {
                fn.call(el, e);
            }, capture);
        }
        
    } else if (window.attachEvent) {
        addEvent = function(el, type, fn, capture) {
            el.attachEvent("on" + type, function(e) {
                fn.call(el, e);
            });
        }
    } 
    addEvent(arguements);
};
```
在判断完成后重新定义函数，这样下一次再执行这个函数就不会进行二次判断了。

### 四、函数绑定

函数绑定可以在指定this环境中以特定参数调用另一个函数。
举个例子：

```
let btn = document.createElement('button');
let handler = {
    count: 0,
    handlerClick: function clickHandler () {
        console.log(++this.count);
    }
}
btn.addEventListener('click', handler.handlerClick);
```
我们创建了一个按钮，给按钮添加了点击事件，但是当我们点击按钮的时候控制台打印的不会是1、2、3、4，因为在函数执行时this指向的是全局对象，也就是window，而window没有count属性。那么要是想达到这个效果就需要函数绑定来帮忙。

绑定函数写法：

```
function bindFun (fn, target) {
    return function () {
        fn.apply(target, arguements)
    }
}
```
用法：

```
btn.addEventListener('click', bindFun(handler.handlerClick, handler));
```
这样我们就能在点击事件中保持函数的this的对象了，并且由于我们将参数也一同传给了函数，函数还能获取到event对象。

```
let handler = {
    count: 0,
    handlerClick: function clickHandler (event) {
        console.log(++this.count + ':' + event.type);
    }
}
```
ES5已经实现了原生的bind方法，可以直接在函数上调用这个方法。

```
btn.addEventListener('click', handler.handlerClick.bind(handler));
```


### 五、函数柯理化

函数柯理化(function currying)，这个理解起来比较奇怪，我自己的理解是这样的：

>有一个函数，能接受多个参数，但是我只能确定他的前几个参数，随后的几个想在实际使用时再传入，这时候就需要用到柯理化的方法，将接受多个参数的函数变成一次只接受一个参数的函数。

举个例子：

```
function add (num1, num2) {
    return num1 + num2;
}
```
很普通的加法函数，现在我知道第一个数字是3，那么在实际调用的时候可能是这样的：

```
add(3, 4);
add(3, 5);
add(3, 133);
```
会发现我们写了很多遍第一个参数3，于是就会有人想到，那干脆新定义一个函数：

```
function newAdd(num) {
    return 3 + num;
}
```
这样不就只需要传一个参数了么。有道理，但是也有些人是也需要add函数的，因为不是每个地方都使用3相加，那聪明的人肯定也会想到方法：

```
function newAdd(num) {
    return add(3, num);
}
```
这不就行了。很好，其实在不知不觉你已经知道了为何要进行柯理化了。

柯理化的通用方法：

```
function curry(fn) {
    var arg = Array.prototype.slice.call(arguements, 1);
    return function () {
        var innerArg = Array.prototype.slice.call(arguements);
        var finalArg = arg.concat(innerArg);
        return fn.apply(null, finalArg);
    }
}
```
这个函数的主要作用就是将需要柯理化的函数和默认的参数进行绑定，然后返回一个接受剩下参数的函数。

柯理化结合函数绑定能实现更加复杂的bind函数：

```
function bind(fn, context) {
    var arg = Array.prototype.slice.call(arguements, 2);
    return function () {
        var innerArg = Array.prototype.slice.call(arguements);
        var finalArg = arg.concat(innerArg);
        return fn.apply(context, finalArg);
    }
}
```
这个负责的bind函数有什么用呢？除了能绑定执行环境，还能动态添加函数的入参，例如：

```
let handler = {
    count: 0,
    handlerClick: function clickHandler (name, event) {
        console.log(++this.count + ':' + name + ':' + event.type);
    }
}
btn.addEventListener('click', bind(handler.handlerClick, handler, 'my-btn'));
```
handler.handlerClick方法现在能接收name入参了。

**柯理化和函数绑定结合起来能提供强大的动态创建函数功能。**

