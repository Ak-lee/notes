## 原生JS实现一个双向绑定Object.defineProperty

使用 `Object.defineProperty` 可以实现数据的拦截。

`Object.defineProperty()` 方法会直接在一个对象上定义个新属性，或者修改一个对象的现有属性。这个方法执行完之后会返回修改后的这个对象。

### 语法

```javascript
Object.defineProperty(obj, prop, descriptor)
```

* obj: 要处理的目标对象
* prop： 要定义或修改的属性的名称
* descriptor: 将被定义或修改的属性描述符

### 使用方法

```javascript
var obj = {}
obj.name = 'Mike'
obj['age'] = 18
Object.defineProperty(obj, 'intro', {
    value: 'hello world',
})
console.log(obj)	// {name: 'Mike', age: 18, intro: 'hello world'}
```

以上三种方法都可以用来定义/修改一个属性，`Object.defineProperty` 的方法貌似看起来有些大题小做，没关系，且往下看它的复杂用法

```javascript
var obj = {}
Object.defineProperty(obj, 'intro', {
    configurable: false,
    value: 'hello world'
})
obj.intro = 'hello Michael'
console.log(obj.intro)	// “hello world”
delete obj.intro		// false，删除失败
console.log(obj.intro)	// “hello world”

Object.defineProperty(obj, 'name', {
    configurable: false,
    value: "Mike Jackson"
})
delete obj.name	// true 删除成功
```

通过上面的例子我们可以看出，数据描述对象中 configurable 的值设置为false后（默认

用 Object.defineProperty 定义的数据的 configurable 就是false），以后就不能再次通过 Object.defineProperty 修改属性，也无法删除该属性。

```javascript
var obj = {name: "xiaoming"}
Object.defineProperty(obj, 'age', {
    value: 3,
    enumerable: false,
})
for(var key in obj) {
    console.log(key)	// 只输出 'name', 不输出 'age'
}
```

设置 enumerable 属性为 false 后，遍历对象的时候会忽略当前属性（如果未设置，默认就是false，即不可遍历）

```javascript
var obj = {name: "xiaoming"}
Object.defineProperty(obj, 'age', {
    value: 3,
    writabel: false,
})
obj.age = 4
console.log(obj.age) // 3, writable 为 false 时，修改对象的当前属性值无效
```

value 和 writable 叫 **数据描述符**， 具有以下可选键值：

* value： 该属性对应的值。可以是任何有效的 Javascript 值，默认为 undefined
* writable： 当且仅当该属性的writable为true时，该属性该才能被赋值运算符改变。**默认为 false**。

`configurable: true` 和 `writable: true` 的区别就是，前者是设置属性能删除，后者是设置属性能修改。

```javascript
var obj = {}
var age
Object.defineProperty(obj, 'age', {
    get: function() {
        console.log('get age ...')
        return age	// 这个age， 不是上面那个带引号的age，而是第二行的变量 age
    },
    set: function(val) {
        console.log('set age ...')
        age = val
    }
    
})
obj.age = 100	// 'set age ...'
console.log(obj.age)	// 'get age ...', 100
```

在上面的例子中， `obj.age` 与外面的age指向同一块内存，修改其中的人一个，另一个也跟着一起变化。

不能这样写：

```javascript
var obj2 = {}
Object.defineProperty(obj2, 'age', {
    get: function() {
        console.log('get age ...')
        return obj2.age
    },
    set: function(val) {
        console.log('set age ...')
        obj2.age = val
    }
})
```

上面的这种写法会造成死循环。如打印 obj2.age 时，会触发`get`方法，`return` 方法中返回 `obj2.age` ，也是对 `obj2.age` 的读操作，又会触发 `get` 方法。故会死循环。对obj2.age 赋值的时候也是同理。

`get` 和 `set` 叫**存取描述符**，有以下可选键值：

* get: 一个给属性提供 `getter` 的方法，如果没有 getter 则为undefined。该方法返回值被用作属性值。默认为 undefined。
* set: 一个给属性提供 `setter` 的方法，如果没有 setter 则为undefined。改方法接受唯一参数，并将该参数的新值分配给该属性。默认为 undefined。

**数据描述符和存取符是不可能同时存在的**

```javascript
var obj = {}
var age 
Object.defineProperty(obj, 'age', {
    value: 100,
    get: function(){
        console.log('get age...')
        return age
    },
    set: function(val){
        console.log('set age...')
        age = val
    }
})
```

因为有 `value` ，又有 `get` ，故上述代码会报错。

## 数据的劫持

上面我们介绍了 `Object.defineProperty` 的用法，现在我们利用这个方法来动态监听数据。

```javascript
var data = {
    name: 'Michael',
    friends: [1, 2, 3]
}
observe(data)
console.log(data.name)
data.name = 'Jackson'
data.friends[0] = 4

function observe(data){
    if(!data || typeof data !== 'object') return
    for(var key in data) {
    	let val = data[key]	// val 是一个临时变量
        Object.defineProperty(data, key, {
			enumerable: true,
    		configurable: true,
            get: function() {
				console.log(`get ${val}`)
				return val
			},
            set: function(newVal) {
				console.log(`changes happen: ${val}=>${newVal}`)
				val = newVal
            }
        })
        if(typeof val === 'object') {
			observe(val)
        }
	}
}
```

上面的 observe 函数实现了一个数据监听，当监听某个对象后，我们可以在用户读取或者设置属性值得时候做个拦截，做我们想做的事情。

## 观察者模式

一个典型的观察者模式应用场景是用户在一个网站订阅主题

1. 多个用户（观察者，Observer） 都可以订阅某个主题（Subject）

2. 当主题内容更新时订阅该主题的用户都能收到通知。

以下是代码实现

Subject 是构造函数，new Subject() 创建一个主题对象，该对象内部维护订阅当前主题的观察者数组。主题对象上有一些方法，如添加观察者（addObserver），删除观察者（removeObserver）、通知观察者更新（notify）。当notify时，实际上调用全部观察者 observer 自身的 update 方法。

Observer 是构造函数，new Observer() 创建一个观察者对象，该对象有一个 update 方法。

```javascript
function Subject() {
    this.observers = []
}
Subject.prototype.addObserver = function(observer) {
    this.observers.push(observer)
}
Subject.prototype.removeObserver = function(observer) {
    var index = this.observers.indexOf(observer)
    if(index > -1) {
        this.observers.splice(index, 1)
    }
}
Subject.prototype.notify = function() {
    this.observers.forEach(function(observer) {
        observer.update()
    })
}

function Observer(name) {
    this.name = name
    this.update = function() {
        console.log(name + ' update...')
    }
}

// 创建主题
var subject = new Subject()

// 创建观察者1
var observer1 = new Observer('Michael')
// 主题添加观察者1
subject.addObserver(observer1)
// 创建观察者2
var observer2 = new Observer('Jackson')
// 主题添加观察者2
subject.addObserver(observer2)

// 主题通知所有的观察者更新
subject.notify()
```

换成ES6 的写法：

```javascript
class Subject {
    constructor() {
        this.observers = []
    }
    addObserver(observer) {
        this.observers.push(observer)
    }
    removeObserver(observer) {
        var index = this.observers.indexOf(observer)
        if(index > -1) {
            this.observers.splice(index, 1)
        }
    }
    notify() {
        this.observers.forEach(observer => {
            observer.update()
        })
    }
}

class Observer() {
    constructor() {
        this.update = function() {}
    }
}
```

上面的代码中， 主题被观察者订阅的写法是 subject.addObserver(observer), 不是很直观，下面我们给观察者添加订阅方法。

```javascript
class Observer() {
    constructor() {
        this.update = function() {}
    }
    subscribeTo(subject){
        subject.addObserver(this)
    }
}

let subject = new Subject()
let observer = new Observer()
observer.update = function() {
    console.log('observer update')
}
observer.subscribeTo(subject)	// 观察者订阅主题

subject.notify()
```

## MVVM 单向绑定的实现

### 概念

MVVM（Model-View-ViewModel）是一种用于把数据和 UI 分离的设计模式

MVVM中的 Model 表示应用程序使用的数据，比如一个用户账户信息（名字、头像、电子邮箱等），Model 保存信息，但通常不会处理行为，不会对信息进行再次加工。数据的格式化是由 View 处理的。行为一般认为是业务逻辑。封装在 ViewModel 中。

View 是与用户进行交互的桥梁。

ViewModel 充当数据转换器。将 Model 信息转换为 View 的信息。将命令从 View 传递到 Model 中。

## 思考

假如有如下代码， data 里面的 `name` 会和视图中的 `{{name}}` 一一映射，修改 data 里面的值，会直接引起视图中对应数据的变化。

```html
<body>
    <div id="app">{{name}}</div>
    <script>
        function mvvm() {
            // todo ...
        }
        var vm = new mvvm() {
            el: '#app', 
            data: {
				name: 'xiaoming'
            }
        }
    </script>
</body>
```

如何实现上述 mvvm 呢？

一起回想之前讲的观察者模式和数据监听。

1. 主题（subject）是什么
2. 观察者（observer）是什么
3. 观察者何时订阅主题？
4. 主题何时通知更新？

上面的例子中，主题应该是 `data` 的 name 属性，观察者应该是视图里面的 `{{name}}` ，当一开始执行mvvm初始化（根据 `el` 解析发现 `{{name}}`的时候订阅主题，当 `data.name` 发生改变的时候，通知观察者更新内容。我们在一开始监控 `data.name`(Object.defineProperty(data, 'name', {...}))）, 当用户修改`data.name` 的时候对用主题的 `subject.notify`.

## 单向绑定的实现

```
!DOCTYPE html>
<html>
    <head>
        <meta charset="utf-8"/>
        <title>MVVM 单向绑定</title>
    </head>
    <body>
        <div id="app">
            <h1>
                {{name}}'s age is {{age}}
            </h1>
        </div>
    </body>
</html>
<script src='./mvvm.js'></script>
```

下面是 `mvvm.js`

`data` 中的每一个属性都是一个 `subject`. 即发布者

```javascript
function observe(data) {
    // 监控数据的变化
    if(!data || typeof data !== 'object') return
    for(var key in data) {
        let val = data[key]
        let mykey = key
        // 为 data 中的每个字段创建一个 subject（发布者）
        let subject = new Subject()
        Object.defineProperty(data, key, {
            enumerable: true,
            configurable: true,
            get: function() {
                console.log(`get ${mykey}: ${val}`)
                if(currentObserver) {
                    // currentObserver 当前的观察者
                    // 如果有当前的观察者，就把他加入到当前字段的订阅列表中
                    console.log('has currentObserver')
                    currentObserver.subscribeTo(subject)
                }
                return val
            },
            set: function(newVal) {
                val = newVal
                console.log('start notify...')
                subject.notify()
            }
        })
        if(typeof val === 'object') {
            observe(val)
        }
    }
}

let id = 0
// currentObserver 即当前的观察者
let currentObserver = null

class Subject {
    constructor() {
        // 每个Subject实例都有个id
        this.id = id++
        this.observers = []
    }
    addObserver(observer) {
        this.observers.push(observer)
    }
    removeObserver(observer) {
        var index = this.observers.indexOf(observer)
        if(index > -1) {
            this.observers.splice(index, 1)
        }
    }
    notify() {
        this.observers.forEach(observer => {
            observer.update()
        })
    }
}

class Observer {
    constructor(vm, key, cb) {
        // vm 即 let vm = new mvvm({}) 的那个vm
        // key 字段名，被监听的那个属性
        // cb，即callback，回调函数
        this.subjects = {}
        this.vm = vm
        this.key = key
        this.cb = cb
        this.value = this.getValue()
    }
    update() {
        let oldVal = this.value
        let value = this.getValue()
        if(value !== oldVal){
            this.value = value
            this.cb.bind(this.vm)(value, oldVal)
        }
    }
    subscribeTo(subject) {
        if(!this.subjects[subject.id]) {
            console.log('subscribeTo ...', subject)
            subject.addObserver(this)
            this.subjects[subject.id] = subject
        }
    }
    getValue() {
        currentObserver = this
        let value = this.vm.$data[this.key]
        currentObserver = null
        return value
    }
}

class mvvm {
    constructor(opts) {
        this.init(opts)
        observe(this.$data)
        this.compile()
    }
    init(opts){
        this.$el = document.querySelector(opts.el)
        this.$data = opts.data
        this.observers = []
    }
    compile() {
        this.traverse(this.$el)
    }
    traverse(node) {
        if(node.nodeType === 1) {
            node.childNodes.forEach(childNode => {
                this.traverse(childNode)
            })
        } else if(node.noeType === 3) {
            // 文本
            this.renderText(node)
        }
    }
    renderText(node) {
        let reg = /{{(.+?)}}/g
        let match
        while(match = reg.exec(node.nodeValue)) {
            let raw = match[0]
            let key = match[1].trim()
            node.nodeValue = node.nodeValue.replace(raw, this.$data[key])
            new Observer(this, key, function(val, oldVal) {
                node.nodeValue = node.nodeValue.replace(oldVal, val)
            })
        }
    }
}

let vm = new mvvm({
    el: '#app',
    data: {
        name: 'mike',
        age: 3
    }
})

setInterval(function() {
    vm.$data.age ++
}, 1000)
```

解析：

* mvvm() 介绍一个配置参数的对象。
  * 做初始化 `init`。把 `el` 对应的dom节点挂载到 `$el` 上，把  `data` 赋值为 `$data` 。创建一个空的 observer 的数组observers（**这一步目前还看不懂**）
  * 监听 `$data` 的变化。会为每个 `$data` 中的属性创建一个 `subject`（发布者）。当读取 `$data` 中的数据时，`subject`(发布者) 中会记录有哪些（observer）订阅者，同时 observer （订阅者）中也会记录自己订阅了哪些 `subject`(发布者)。当修改 `$data` 中的数据时，会调用 `subject ` 保存的所有订阅者的 `update` 方法。
  * 启动编译`compile` , 对dom元素进行递归，指导递归到 文本级（node.nodeType === 3）, 对文本内容调用 `renderText(node)` 方法。
  * `renderText(node)` 方法，首先使用正则表达式匹配出待注入数据的项。然后使用 `$data` 中的数据替换掉原来的占位字符。**并且新建一个 Observer（观察者）， 这个观察者用于观察对应对象的对应属性，一旦对应的属性发生变化，则调用回调函数，即重新更新当前的dom元素内容**。 即模板html中有几个 `{{}}` 标记，才会新建几个观察者（observer）

* Observer 观察者

  * 生成 Observer 实例 `new Observer(vm, key, cb)` vm 即要观察的是哪个对象。key，即观察这个对象的哪个属性。`cb`, 为 callback, 回调函数。回调函数在 `Observer ` 实例被调用 `update` 方法是会调用。

  * Observer 实例上会挂载 `vm`(即当前监听的是哪个对象)， `key` 当前监听的是哪个对象的哪个属性。`cb` 即回调函数，即属性更新时要做什么。`value` 监听对象的监听属性的当前值。
  * `getValue` 方法，即获取当前监听对象的监听属性的当前值。当获取当前监听对象的监听属性的当前值的时候，会把全局变量 `currentObserver` 设置为当前 observer。方便`observe` 函数那边记录当前observer有没有进入订阅者数组。**这里的 getValue 是观察者去获取被监听对象的值**。
  * `update` 方法，比较当前保存的 `value` 与现在最新值有无变化。如果有变化，则调用 `cb` 回调函数。回调函数会被一次传入 `val` 和 `oldVal` (新值和旧值)。

**总结一下思路**： 根据模板html字符串中需要注入的数据来分别新建观察者（observer）。读取被监听对象的被监听属性时，分为两种情况，一种是直接读，`renderText` 函数中有出现。另一种是由观察者去读。观察者去读有点特殊，观察者去读之前，会把 `currentObserver` 置为当前的观察者，方便 subject（发布者）那边把观察者放入观察者数组中。每次新建一个观察者，都会触发这个观察者以观察者的身份去取对应的属性。对方的subject就会把这个观察者添加到观察者数组中去。每次数据发生变化后，都会调用观察者数组中每个观察者的`update` 的方法，即告诉各个观察者：**我这边的数据更新了，你们重新来获取一下吧。** 观察者每次重新获取`value` 后，会判断 `value` 有无变化（毕竟可能出现把一个值修改为一模一样的新值，同样触发了 setter），如果有变化就执行回调函数更新页面。

**专门设计了观察者（Observer）会保存观察对象的观察属性的当前值**。故观察者在初始化时（constructor函数阶段）一定会调用 getValue, 方便在第一次时，subject 那边把观察者放进观察者数组中。

理解上面的代码，关键是要弄清楚一个问题，**我们需要去订阅一个属性的变动，但什么时候去订阅呢？**。答：在解析模板的时候创建观察者，在观察者初始化的时候就去订阅。创建了观察者就马上会进入观察者初始化工作。

## 实现双向绑定

```
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>MVVM 双向绑定</title>
</head>
<body>

<div id="app" >
  <input v-model="name" type="text">
  <h1>{{name}} 's age is {{age}}</h1>
</div>

<script>

function observe(data) {
  if(!data || typeof data !== 'object') return
  for(var key in data) {
    let val = data[key]
    let subject = new Subject()
    Object.defineProperty(data, key, {
      enumerable: true,
      configurable: true,
      get: function() {
        console.log(`get ${key}: ${val}`)
        if(currentObserver){
          console.log('has currentObserver')
          currentObserver.subscribeTo(subject)
        }
        return val
      },
      set: function(newVal) {
        val = newVal
        console.log('start notify...')
        subject.notify()
      }
    })
    if(typeof val === 'object'){
      observe(val)
    }
  }
}

let id = 0
let currentObserver = null

class Subject {
  constructor() {
    this.id = id++
    this.observers = []
  }
  addObserver(observer) {
    this.observers.push(observer)
  }
  removeObserver(observer) {
    var index = this.observers.indexOf(observer)
    if(index > -1){
      this.observers.splice(index, 1)
    }
  }
  notify() {
    this.observers.forEach(observer=> {
      observer.update()
    })
  }
}

class Observer{
  constructor(vm, key, cb) {
    this.subjects = {}
    this.vm = vm
    this.key = key
    this.cb = cb
    this.value = this.getValue()
  }
  update(){
    let oldVal = this.value
    let value = this.getValue()
    if(value !== oldVal) {
      this.value = value
      this.cb.bind(this.vm)(value, oldVal)
    }
  }
  subscribeTo(subject) {
    if(!this.subjects[subject.id]){
      console.log('subscribeTo.. ', subject)
       subject.addObserver(this)
       this.subjects[subject.id] = subject
    }
  }
  getValue(){
    currentObserver = this
    let value = this.vm.$data[this.key]
    currentObserver = null
    return value
  }
} 


class Compile {
  constructor(vm){
    this.vm = vm
    this.node = vm.$el
    this.compile()
  }
  compile(){
    this.traverse(this.node)
  }
  traverse(node){
    if(node.nodeType === 1){
      this.compileNode(node)   //解析节点上的v-bind 属性
      node.childNodes.forEach(childNode=>{
        this.traverse(childNode)
      })
    }else if(node.nodeType === 3){ //处理文本
      this.compileText(node)
    }
  }
  compileText(node){
    let reg = /{{(.+?)}}/g
    let match
    console.log(node)
    while(match = reg.exec(node.nodeValue)){
      let raw = match[0]
      let key = match[1].trim()
      node.nodeValue = node.nodeValue.replace(raw, this.vm.$data[key])
      new Observer(this.vm, key, function(val, oldVal){
        node.nodeValue = node.nodeValue.replace(oldVal, val)
      })
    }    
  }

  //处理指令
  compileNode(node){
    let attrs = [...node.attributes] //类数组对象转换成数组，也可用其他方法
    attrs.forEach(attr=>{
      //attr 是个对象，attr.name 是属性的名字如 v-model， attr.value 是对应的值，如 name
      if(this.isDirective(attr.name)){
        let key = attr.value       //attr.value === 'name'
        node.value = this.vm.$data[key]  
        new Observer(this.vm, key, function(newVal){
          node.value = newVal
        })
        node.oninput = (e)=>{
          this.vm.$data[key] = e.target.value  //因为是箭头函数，所以这里的 this 是 compile 对象
        }
      }
    })
  }
  //判断属性名是否是指令
  isDirective(attrName){
     return attrName === 'v-model'
  }

}


class mvvm {
  constructor(opts) {
    this.init(opts)
    observe(this.$data)
    new Compile(this)
  }
  init(opts){
    this.$el = document.querySelector(opts.el)
    this.$data = opts.data
  }

}

let vm = new mvvm({
  el: '#app',
  data: { 
    name: 'jirengu',
    age: 3
  }
})

// setInterval(function(){
//   vm.$data.age++
// }, 1000)


</script>
</body>
</html>
```

## 增加事件指令 、完善代码

```
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>事件与代码优化</title>
</head>
<body>

<div id="app" >
  <input v-model="name" v-on:click="sayHi" type="text">
  <h1>{{name}} 's age is {{age}}</h1>
</div>

<script>

function observe(data) {
  if(!data || typeof data !== 'object') return
  for(var key in data) {
    let val = data[key]
    let subject = new Subject()
    Object.defineProperty(data, key, {
      enumerable: true,
      configurable: true,
      get: function() {
        console.log(`get ${key}: ${val}`)
        if(currentObserver){
          console.log('has currentObserver')
          currentObserver.subscribeTo(subject)
        }
        return val
      },
      set: function(newVal) {
        val = newVal
        console.log('start notify...')
        subject.notify()
      }
    })
    if(typeof val === 'object'){
      observe(val)
    }
  }
}

let id = 0
let currentObserver = null

class Subject {
  constructor() {
    this.id = id++
    this.observers = []
  }
  addObserver(observer) {
    this.observers.push(observer)
  }
  removeObserver(observer) {
    var index = this.observers.indexOf(observer)
    if(index > -1){
      this.observers.splice(index, 1)
    }
  }
  notify() {
    this.observers.forEach(observer=> {
      observer.update()
    })
  }
}

class Observer{
  constructor(vm, key, cb) {
    this.subjects = {}
    this.vm = vm
    this.key = key
    this.cb = cb
    this.value = this.getValue()
  }
  update(){
    let oldVal = this.value
    let value = this.getValue()
    if(value !== oldVal) {
      this.value = value
      this.cb.bind(this.vm)(value, oldVal)
    }
  }
  subscribeTo(subject) {
    if(!this.subjects[subject.id]){
      console.log('subscribeTo.. ', subject)
       subject.addObserver(this)
       this.subjects[subject.id] = subject
    }
  }
  getValue(){
    currentObserver = this
    let value = this.vm[this.key]   //等同于 this.vm.$data[this.key]
    currentObserver = null
    return value
  }
} 


class Compile {
  constructor(vm){
    this.vm = vm
    this.node = vm.$el
    this.compile()
  }
  compile(){
    this.traverse(this.node)
  }
  traverse(node){
    if(node.nodeType === 1){
      this.compileNode(node)   //解析节点上的v-bind 属性
      node.childNodes.forEach(childNode=>{
        this.traverse(childNode)
      })
    }else if(node.nodeType === 3){ //处理文本
      this.compileText(node)
    }
  }
  compileText(node){
    let reg = /{{(.+?)}}/g
    let match
    console.log(node)
    while(match = reg.exec(node.nodeValue)){
      let raw = match[0]
      let key = match[1].trim()
      node.nodeValue = node.nodeValue.replace(raw, this.vm[key])
      new Observer(this.vm, key, function(val, oldVal){
        node.nodeValue = node.nodeValue.replace(oldVal, val)
      })
    }    
  }

  //处理指令
  compileNode(node){
    let attrs = [...node.attributes] //类数组对象转换成数组，也可用其他方法
    attrs.forEach(attr=>{
      //attr 是个对象，attr.name 是属性的名字如 v-model， attr.value 是对应的值，如 name
      if(this.isModelDirective(attr.name)){
        this.bindModel(node, attr)
      }else if(this.isEventDirective(attr.name)){
        this.bindEventHander(node, attr)
      }
    })
  }
  bindModel(node, attr){
    let key = attr.value       //attr.value === 'name'
    node.value = this.vm[key]  
    new Observer(this.vm, key, function(newVal){
      node.value = newVal
    })
    node.oninput = (e)=>{
      this.vm[key] = e.target.value  //因为是箭头函数，所以这里的 this 是 compile 对象
    }
  }
  bindEventHander(node, attr){       //attr.name === 'v-on:click', attr.value === 'sayHi'
    let eventType = attr.name.substr(5)       // click
    let methodName = attr.value
    node.addEventListener(eventType, this.vm.$methods[methodName]) 
  }

  //判断属性名是否是指令
  isModelDirective(attrName){
     return attrName === 'v-model'
  }

  isEventDirective(attrName){
    return attrName.indexOf('v-on') === 0
  }

}


class mvvm {
  constructor(opts) {
    this.init(opts)
    observe(this.$data)
    new Compile(this)
  }
  init(opts){
    this.$el = document.querySelector(opts.el)
    this.$data = opts.data || {}
    this.$methods = opts.methods || {}

    //把$data 中的数据直接代理到当前 vm 对象
    for(let key in this.$data) {
      Object.defineProperty(this, key, {
        enumerable: true,
        configurable: true,
        get: ()=> {  //这里用了箭头函数，所有里面的 this 就指代外面的 this 也就是 vm
          return this.$data[key]
        },
        set: newVal=> {
          this.$data[key] = newVal
        }        
      })
    }

    //让 this.$methods 里面的函数中的 this，都指向当前的 this，也就是 vm
    for(let key in this.$methods) {
      this.$methods[key] = this.$methods[key].bind(this)
    }
  }

}

let vm = new mvvm({
  el: '#app',
  data: { 
    name: 'jirengu',
    age: 3
  },
  methods: {
    sayHi(){
      alert(`hi ${this.name}` )
    }
  }
})

let clock = setInterval(function(){
  vm.age++   //等同于 vm.$data.age， 见 mvvm init 方法内的数据劫持

  if(vm.age === 10) clearInterval(clock)
}, 1000)


</script>
</body>
</html>
```

