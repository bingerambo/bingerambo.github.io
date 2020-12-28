# JavaScript 模式设计-01



>"The key is to acknowledge from the start that you have no idea how this will grow. When you accept that you don't know everything, you begin to design the system defensively. You identify the key areas that may change, which often is very easy when you put a little bit of time into it. For instance, you should expect that any part of the app that communicates with another system will likely change, so you need to abstract that away." ——一切皆可变，所以要抽象。

<!--more-->

<br>

<h2>模块论</h2>
<p>
    大家可能都或多或少地使用了模块化的代码，模块是一个完整的强健程序架构的一部分，每个模块都是为了单独的目的为创建的，回到Gmail，我们来个例子，chat聊天模块看起来是个单独的一部分，其实它是有很多单独的子模块来构成，例如里面的表情模块其实就是单独的子模块，也被用到了发送邮件的窗口上。
    <br />
    <br />
    另外一个是模块可以动态加载，删除和替换。
    <br />
    <br />
    在JavaScript里，我们有几种方式来实现模块，大家熟知的是module模式和对象字面量,如果你已经熟悉这些，请忽略此小节，直接跳到CommonJS部分。
    <br />
    <br /> <strong>Module模式</strong>
    <br />
    module模式是一个比较流行的设计模式，它可以通过大括号封装私有的变量，方法，状态的，通过包装这些内容，一般全局的对象不能直接访问，在这个设计模式里，只返回一个API，其它的内容全部被封装成私有的了。
    <br />
    <br />
    另外，这个模式和自执行的函数表达式比较相似，唯一的不同是module返回的是对象，而自执行函数表达式返回的是function。
    <br />
    <br />
    众所周知， JavaScript不想其它语言一样有访问修饰符，不能为每个字段或者方法声明private,public修饰符，那这个模式我们是如何实现的呢？那就是return一个对象，里面包括一些公开的方法，这些方法有能力去调用内部的对象。
    <br />
    <br />
    看一下，下面的代码，这段代码是一个自执行代码，声明里包括了一个全局的对象basketModule， basket数组是一个私有的，所以你的整个程序是不能访问这个私有数组的，同时我们return了一个对象，其内包含了3个方法（例如addItem,getItemCount,getTotal)，这3个方法可以访问私有的basket数组。
</p>
<div class="cnblogs_code">
    <pre><span style="color: #0000ff;">var</span> basketModule = (<span style="color: #0000ff;">function</span>() {<br /><span style="color: #0000ff;">var</span> basket = []; <span style="color: #008000;">//</span><span style="color: #008000;">private</span><span style="color: #008000;"><br /></span><span style="color: #0000ff;">return</span> { <span style="color: #008000;">//</span><span style="color: #008000;">exposed to public</span><span style="color: #008000;"><br /></span>　　addItem: <span style="color: #0000ff;">function</span>(values) {<br />　　　　basket.push(values);<br />　　},<br />　　getItemCount: <span style="color: #0000ff;">function</span>() {<br /><span style="color: #0000ff;">return</span> basket.length;<br />　　},<br />　　getTotal: <span style="color: #0000ff;">function</span>(){<br /><span style="color: #0000ff;">var</span> q = <span style="color: #0000ff;">this</span>.getItemCount(),p=0;<br /><span style="color: #0000ff;">while</span>(q--){<br />　　　　p+= basket[q].price;<br />　　　　}<br /><span style="color: #0000ff;">return</span> p;<br />　　}<br />　}<br />}());</pre>
</div>
<p>
    <br />
    同时注意，我们return的对象直接赋值给了basketModule，所以我们可以像下面一样使用：
</p>
<div class="cnblogs_code">
    <pre><span style="color: #008000;">//</span><span style="color: #008000;">basketModule is an object with properties which can also be methods</span><span style="color: #008000;"><br /></span>basketModule.addItem({item:'bread',price:0.5});<br />basketModule.addItem({item:'butter',price:0.3});<br /> <br />console.log(basketModule.getItemCount());<br />console.log(basketModule.getTotal());<br /> <br /><span style="color: #008000;">//</span><span style="color: #008000;">however, the following will not work:</span><span style="color: #008000;"><br /></span>console.log(basketModule.basket);<span style="color: #008000;">//</span><span style="color: #008000;">(undefined as not inside the returned object)</span><span style="color: #008000;"><br /></span>console.log(basket); <span style="color: #008000;">//</span><span style="color: #008000;">(only exists within the scope of the closure)</span></pre>
</div>
<p>
    <br />
    那在各个流行的类库（如Dojo, jQuery)里是如何来做呢？
</p>
<p>
    <span style="color: #99cc00;"> <strong>Dojo</strong>
    </span>
</p>
<p>
    Dojo试图使用dojo.declare来提供class风格的声明方式，我们可以利用它来实现Module模式，例如如果你想再store命名空间下声明basket对象，那么可以这么做：
</p>
<div class="cnblogs_code">
    <pre>    <span style="color: #008000;">//</span><span style="color: #008000;">traditional way</span><span style="color: #008000;"><br /></span>    <span style="color: #0000ff;">var</span> store = window.store || {};<br />    store.basket = store.basket || {};<br />     <br />    <span style="color: #008000;">//</span><span style="color: #008000;">using dojo.setObject</span><span style="color: #008000;"><br /></span>    dojo.setObject("store.basket.object", (<span style="color: #0000ff;">function</span>() {<br />    <span style="color: #0000ff;">var</span> basket = [];<br />    <span style="color: #0000ff;">function</span> privateMethod() {<br />    　　　　console.log(basket);<br />    　　}<br />    　　<span style="color: #0000ff;">return</span> {<br />    　　　　publicMethod: <span style="color: #0000ff;">function</span>(){<br />    　　　　　　privateMethod();<br />    　　　　}<br />    　　　};<br />    }()));</pre>
</div>
<p>结合dojo.provide一起来使用，非常强大。</p>
<p>
    <strong><span style="color: #99cc00;">YUI</span></strong> 
</p>
<p>下面的代码是YUI原始的实现方式：</p>
<div class="cnblogs_code">
    <pre>YAHOO.store.basket = <span style="color: #0000ff;">function</span> () {<br /><br />    <span style="color: #008000;">//</span><span style="color: #008000;">"private" variables:</span><span style="color: #008000;"><br /></span>    <span style="color: #0000ff;">var</span> myPrivateVar = "I can be accessed only within YAHOO.store.basket .";<br /><br />    <span style="color: #008000;">//</span><span style="color: #008000;">"private" method:</span><span style="color: #008000;"><br /></span>    <span style="color: #0000ff;">var</span> myPrivateMethod = <span style="color: #0000ff;">function</span> () {<br />        YAHOO.log("I can be accessed only from within YAHOO.store.basket");<br />    }<br /><br />    <span style="color: #0000ff;">return</span> {<br />        myPublicProperty: "I'm a public property.",<br />        myPublicMethod: <span style="color: #0000ff;">function</span> () {<br />            YAHOO.log("I'm a public method.");<br /><br />            <span style="color: #008000;">//</span><span style="color: #008000;">Within basket, I can access "private" vars and methods:</span><span style="color: #008000;"><br /></span>            YAHOO.log(myPrivateVar);<br />            YAHOO.log(myPrivateMethod());<br /><br />            <span style="color: #008000;">//</span><span style="color: #008000;">The native scope of myPublicMethod is store so we can</span><span style="color: #008000;"><br /></span>            <span style="color: #008000;">//</span><span style="color: #008000;">access public members using "this":</span><span style="color: #008000;"><br /></span>            YAHOO.log(<span style="color: #0000ff;">this</span>.myPublicProperty);<br />        }<br />    };<br /><br />} ();</pre>
</div>
<p>&nbsp;</p>
<p>
    <strong><span style="color: #99cc00;">jQuery</span></strong> 
</p>
<p>
    jQuery里有很多Module模式的实现，我们来看一个不同的例子，一个library函数声明了一个新的library，然后创建该library的时候，在document.ready里自动执行init方法。
</p>
<div class="cnblogs_code">
    <pre><span style="color: #0000ff;">function</span> library(module) {<br />　　$(<span style="color: #0000ff;">function</span>() {<br /><span style="color: #0000ff;">if</span> (module.init) {<br />　　　　　　module.init();<br />　　　　}<br />　　});<br /><span style="color: #0000ff;">return</span> module;<br />}<br /> <br /><span style="color: #0000ff;">var</span> myLibrary = library(<span style="color: #0000ff;">function</span>() {<br /><span style="color: #0000ff;">return</span> {<br />　　　　init: <span style="color: #0000ff;">function</span>() {<br /><span style="color: #008000;">/*</span><span style="color: #008000;">implementation</span><span style="color: #008000;">*/</span><br />　　　　　　}<br />　　};<br />}());</pre>
</div>

<p>
    <br />
    <strong>对象自面量</strong>
    <br />
    对象自面量使用大括号声明，并且使用的时候不需要使用new关键字，如果对一个模块里的属性字段的publice/private不是很在意的话，可以使用这种方式，不过请注意这种方式和JSON的不同。对象自面量：
    <span style="color: #ff9900;">
        <strong>var item={name: "tom", value:123}</strong>
    </span>
    JSON:
    <span style="color: #ff9900;">var item={"name":"tom", "value":123}</span>
    。
    <br />
    <br />
</p>
<div class="cnblogs_code">
    <pre><span style="color: #0000ff;">var</span> myModule = {<br />    myProperty: 'someValue',<br />    <span style="color: #008000;">//</span><span style="color: #008000;">object literals can contain properties and methods.</span><span style="color: #008000;"><br /></span>    <span style="color: #008000;">//</span><span style="color: #008000;">here, another object is defined for configuration</span><span style="color: #008000;"><br /></span>    <span style="color: #008000;">//</span><span style="color: #008000;">purposes:</span><span style="color: #008000;"><br /></span>    myConfig: {<br />        useCaching: <span style="color: #0000ff;">true</span>,<br />        language: 'en'<br />    },<br />    <span style="color: #008000;">//</span><span style="color: #008000;">a very basic method</span><span style="color: #008000;"><br /></span>    myMethod: <span style="color: #0000ff;">function</span> () {<br />        console.log('I can haz functionality?');<br />    },<br />    <span style="color: #008000;">//</span><span style="color: #008000;">output a value based on current configuration</span><span style="color: #008000;"><br /></span>    myMethod2: <span style="color: #0000ff;">function</span> () {<br />        console.log('Caching is:' + (<span style="color: #0000ff;">this</span>.myConfig.useCaching) ? 'enabled' : 'disabled');<br />    },<br />    <span style="color: #008000;">//</span><span style="color: #008000;">override the current configuration</span><span style="color: #008000;"><br /></span>    myMethod3: <span style="color: #0000ff;">function</span> (newConfig) {<br />        <span style="color: #0000ff;">if</span> (<span style="color: #0000ff;">typeof</span> newConfig == 'object') {<br />            <span style="color: #0000ff;">this</span>.myConfig = newConfig;<br />            console.log(<span style="color: #0000ff;">this</span>.myConfig.language);<br />        }<br />    }<br />};<br /><br />myModule.myMethod(); <span style="color: #008000;">//</span><span style="color: #008000;">I can haz functionality</span><span style="color: #008000;"><br /></span>myModule.myMethod2(); <span style="color: #008000;">//</span><span style="color: #008000;">outputs enabled</span><span style="color: #008000;"><br /></span>myModule.myMethod3({ language: 'fr', useCaching: <span style="color: #0000ff;">false</span> }); <span style="color: #008000;">//</span><span style="color: #008000;">fr</span></pre>
</div>
<p>
    <br />
    <strong>CommonJS</strong>
    <br />
    关于 CommonJS的介绍，这里就不多说了，博客园有很多帖子都有介绍，我们这里要提一下的是CommonJS标准里里有2个重要的参数exports和require，exports是代表要加载的模块，require是代表这些加载的模块需要依赖其它的模块，也需要将它加载进来。
</p>
<div class="cnblogs_code">
    <pre>    <span style="color: #008000;">/*</span><span style="color: #008000;"><br />    Example of achieving compatibility with AMD and standard CommonJS by putting boilerplate around the standard CommonJS module format:<br />    </span><span style="color: #008000;">*/</span><br />     <br />    (<span style="color: #0000ff;">function</span>(define){<br />    　　define(<span style="color: #0000ff;">function</span>(require,exports){<br />    <span style="color: #008000;">//</span><span style="color: #008000;">module contents</span><span style="color: #008000;"><br /></span>    <span style="color: #0000ff;">var</span> dep1 = require("dep1");<br />    　　　　exports.someExportedFunction = <span style="color: #0000ff;">function</span>(){...};<br />    　　　　<span style="color: #008000;">//</span><span style="color: #008000;">...</span><span style="color: #008000;"><br /></span>    　　});<br />    })(<span style="color: #0000ff;">typeof</span> define=="function"?define:<span style="color: #0000ff;">function</span>(factory){factory(require,exports)});</pre>
</div>
<p>
    有很多CommonJS标准的模块加载实现，我比较喜欢的是RequireJS，它能否非常好的加载模块以及相关的依赖模块，来一个简单的例子，例如需要将图片转化成ASCII码，我们先加载encoder模块，然后获取他的encodeToASCII方法，理论上代码应该是如下：
</p>
<div class="cnblogs_code">
    <pre>    <span style="color: #0000ff;">var</span> encodeToASCII = require("encoder").encodeToASCII;<br />    exports.encodeSomeSource = <span style="color: #0000ff;">function</span>(){<br />    　　<span style="color: #008000;">//</span><span style="color: #008000;">其它操作以后，然后调用encodeToASCII</span><span style="color: #008000;"><br /></span>    }</pre>
</div>
<p>
    但是上述代码并没用工作，因为encodeToASCII函数并没用附加到window对象上，所以不能使用，改进以后的代码需要这样才行：
</p>
<div class="cnblogs_code">
    <pre>    define(<span style="color: #0000ff;">function</span>(require, exports, module) {<br />    <span style="color: #0000ff;">var</span> encodeToASCII = require("encoder").encodeToASCII;<br />    　　　　exports.encodeSomeSource = <span style="color: #0000ff;">function</span>(){<br />    　　　　<span style="color: #008000;">//</span><span style="color: #008000;">process then call encodeToASCII</span><span style="color: #008000;"><br /></span>    　　}<br />    });</pre>
</div>
<p>CommonJS 潜力很大，但是由于大叔不太熟，所以就不过多地介绍了。</p>
<h2>Facade模式</h2>
<p>
    Facade模式在本文架构里占有重要角色，关于这个模式很多JavaScript类库或者框架里都有体现，其中最大的作用，就是包括High level的API，以此来隐藏具体的实现，这就是说，我们只暴露接口，内部的实现我们可以自己做主，也意味着内部实现的代码可以很容易的修改和更新，比如今天你是用jQuery来实现的，明天又想换YUI了，这就非常方便了。
</p>
<p>下面这个例子了，可以看到我们提供了很多私有的方法，然后通过暴露一个简单的 API来让外界执行调用内部的方法：</p>
<div class="cnblogs_code">
    <pre><span style="color: #0000ff;">var</span> module = (<span style="color: #0000ff;">function</span> () {<br />    <span style="color: #0000ff;">var</span> _private = {<br />        i: 5,<br />        get: <span style="color: #0000ff;">function</span> () {<br />            console.log('current value:' + <span style="color: #0000ff;">this</span>.i);<br />        },<br />        set: <span style="color: #0000ff;">function</span> (val) {<br />            <span style="color: #0000ff;">this</span>.i = val;<br />        },<br />        run: <span style="color: #0000ff;">function</span> () {<br />            console.log('running');<br />        },<br />        jump: <span style="color: #0000ff;">function</span> () {<br />            console.log('jumping');<br />        }<br />    };<br />    <span style="color: #0000ff;">return</span> {<br />        facade: <span style="color: #0000ff;">function</span> (args) {<br />            _private.set(args.val);<br />            _private.get();<br />            <span style="color: #0000ff;">if</span> (args.run) {<br />                _private.run();<br />            }<br />        }<br />    }<br />} ());<br /><span class="kwd"><br />module</span><span class="pun">.</span><span class="pln">facade</span><span class="pun">({</span><span class="pln">run</span><span class="pun">:</span><span class="kwd">true</span><span class="pun">,</span><span class="pln">val</span><span class="pun">:</span><span class="lit">10</span><span class="pun">});</span><span class="com"><br /><span style="color: #339966;">//outputs current value: 10, running</span></span></pre>
</div>
<p>
    <br />
    Facade和下面我们所说的mediator的区别是，facade只提供现有存在的功能，而mediator可以增加新功能。
</p>
<p>&nbsp;</p>
<h2>Mediator模式</h2>
<p>
    讲modiator之前，我们先来举个例子，机场飞行控制系统，也就是传说中的塔台，具有绝对的权利，他可以控制任何一架飞机的起飞和降落时间以及地方，而飞机和飞机之前不允许通信，也就是说塔台是机场的核心，mediator就相当于这个塔台。
</p>
<p>
    mediator就是用在程序里有多个模块，而你又不想让各个模块有依赖的话，那通过mediator模式可以达到集中控制的目的。实际场景中也是，mediator封装了很多不想干的模块，让他们通过mediator联系在一起，同时也松耦合他们，使得他们之间必须通过mediator才能通信。
</p>
<p>
    那mediator模式的优点是什么？那就是解耦，如果你之前对观察者模式比较了解的话，那理解下面的mediator图就相对简单多了，下图是一个high level的mediator模式图：
</p>
<p>
    <img src="http://pic002.cnblogs.com/images/2011/349491/2011121316495096.jpg" alt="" />
</p>
<p>想想一下，各模块是发布者，mediator既是发布者又是订阅者。</p>
<ol>
    <li>Module 1向Mediator广播一个实际，说需要做某事</li>
    <li>
        Mediator捕获消息以后，立即启动处理该消息需要使用的Module 2，Module 2处理结束以后返回信息给Mediator
    </li>
    <li>
        与此同时，Mediator也启动了Module 3，当接受Module 2 返回消息的时候自动记录日志到Module 3里
    </li>
</ol>
<p>
    可以看到，各模块之间并没有通信，另外Mediator也可以实现监控各模块状态的功能，例如如果Module 3出错了，Mediator可以暂时只想其它模块，然后重启Module 3，然后继续执行。
</p>
<p>
    回顾一下，可以看到，Mediator的优点是：松耦合的模块由同一的Mediator来控制，模块只需要广播和监听事件就可以了，而模块之间不需要直接联系，另外，一次信息的处理可以使用多个模块，也方便我们以后统一的添加新的模块到现有的控制逻辑里。
</p>
<p>确定是：由于所有的模块直接都不能直接通信，所有相对来说，性能方面可能会有少许下降，但是我认为这是值得的。</p>
<p>&nbsp;</p>
<p>我们根据上面的讲解来一个简单的Demo：</p>
<div class="cnblogs_code">
    <pre>    <span style="color: #0000ff;">var</span> mediator = (<span style="color: #0000ff;">function</span>(){<br />        <span style="color: #0000ff;">var</span> subscribe = <span style="color: #0000ff;">function</span>(channel, fn){<br />            <span style="color: #0000ff;">if</span> (!mediator.channels[channel]) mediator.channels[channel] = [];<br />            mediator.channels[channel].push({ context: <span style="color: #0000ff;">this</span>, callback: fn });<br />            <span style="color: #0000ff;">return</span> <span style="color: #0000ff;">this</span>;<br />        },<br />     <br />        publish = <span style="color: #0000ff;">function</span>(channel){<br />            <span style="color: #0000ff;">if</span> (!mediator.channels[channel]) <span style="color: #0000ff;">return</span> <span style="color: #0000ff;">false</span>;<br />            <span style="color: #0000ff;">var</span> args = Array.prototype.slice.call(arguments, 1);<br />            <span style="color: #0000ff;">for</span> (<span style="color: #0000ff;">var</span> i = 0, l = mediator.channels[channel].length; i &lt; l; i++) {<br />                <span style="color: #0000ff;">var</span> subscription = mediator.channels[channel][i];<br />                subscription.callback.apply(subscription.context, args);<br />            }<br />            <span style="color: #0000ff;">return</span> <span style="color: #0000ff;">this</span>;<br />        };<br />     <br />        <span style="color: #0000ff;">return</span> {<br />            channels: {},<br />            publish: publish,<br />            subscribe: subscribe,<br />            installTo: <span style="color: #0000ff;">function</span>(obj){<br />                obj.subscribe = subscribe;<br />                obj.publish = publish;<br />            }<br />        };<br />     <br />    }());</pre>
</div>
<p>然后有2个模块分别调用：</p>
<div class="cnblogs_code">
    <pre>    <span style="color: #008000;">//</span><span style="color: #008000;">Pub/sub on a centralized mediator</span><span style="color: #008000;"><br /></span>     <br />    mediator.name = "tim";<br />    mediator.subscribe('nameChange', <span style="color: #0000ff;">function</span>(arg){<br />            console.log(<span style="color: #0000ff;">this</span>.name);<br />            <span style="color: #0000ff;">this</span>.name = arg;<br />            console.log(<span style="color: #0000ff;">this</span>.name);<br />    });<br />     <br />    mediator.publish('nameChange', 'david'); <span style="color: #008000;">//</span><span style="color: #008000;">tim, david</span><span style="color: #008000;"><br /></span>     <br />     <br />    <span style="color: #008000;">//</span><span style="color: #008000;">Pub/sub via third party mediator</span><span style="color: #008000;"><br /></span>     <br />    <span style="color: #0000ff;">var</span> obj = { name: 'sam' };<br />    mediator.installTo(obj);<br />    obj.subscribe('nameChange', <span style="color: #0000ff;">function</span>(arg){<br />            console.log(<span style="color: #0000ff;">this</span>.name);<br />            <span style="color: #0000ff;">this</span>.name = arg;<br />            console.log(<span style="color: #0000ff;">this</span>.name);<br />    });<br />     <br />    obj.publish('nameChange', 'john'); <span style="color: #008000;">//</span><span style="color: #008000;">sam, john</span></pre>
</div>
<h2>
    <br />
    应用Facade: 应用程序核心的抽象
</h2>
<p>
    一个facade是作为应用程序核心的一个抽象来工作的，在mediator和模块之间负责通信，各个模块只能通过这个facade来和程序核心进行通信。作为抽象的职责是确保任何时候都能为这些模块提供一个始终如一的接口（consistent interface），和sendbox controller的角色比较类似。所有的模块组件通过它和mediator通信，所以facade需要是可靠的，可信赖的，同时作为为模块提供接口的功能，facade还需要扮演另外一个角色，那就是安全控制，也就是决定程序的哪个部分可以被一个模块访问，模块组件只能调用他们自己的方法，并且不能访问任何未授权的内容。例如，一个模块可能广播dataValidationCompletedWriteToDB，这里的安全检查需要确保该模块拥有数据库的写权限。
</p>
<p>
    <br />
    总之，mediator只有在facade授权检测以后才能进行信息处理。
    <br />
    <br />
</p>
<h2>应用Mediator：应用程序的核心</h2>
<p>
    Mediator是作为应用程序核心的角色来工作的，我们简单地来说一下他的职责。最核心的工作就是管理模块的生命周期（lifecycle），当这个核心扑捉到任何信息进来的时候，他需要判断程序如何来处理&mdash;&mdash;也就是说决定启动或停止哪一个或者一些模块。当一个模块开始启动的时候，它应该能否自动执行，而不需要应用程序核心来决定是否该执行（比如，是否要在DOM ready的时候才能执行），所以说需要模块自身需要去判定。
    <br />
    <br />
    你可能还有问题，就是一个模块在什么情况下才会停止。当程序探测到一个模块失败了，或者是出错了，程序需要做决定来防止继续执行该模块里的方法，以便这个组件可以重新启动，目的主要是提高用户体验。
    <br />
    <br />
    <br />
    另外，该核心应该可以动态添加或者删除模块，而不影响其他任何功能。常见的例子是，一个模块在页面加载初期是不可用，但是用户操作以后，需要动态加载这个模块然后执行，就像Gmail里的chat聊天功能一样，从性能优化的目的来看，应该是很好理解的吧。
    <br />
    <br />
    <br />
    异常错误处理，也是由应用程序核心来处理的，另外各模块在广播信息的时候，也广播任何错误到该核心里，以便程序核心可以根据情况去停止/重启这些模块。这也是松耦合架构一个很重要的部分，我们不需要手工改变任何模块，通过mediator使用发布/订阅就可以来做到这个。
    <br />
    <br />
    <br />
</p>
<h2>组装起来</h2>
<p>
    <strong>各模块</strong>
    包含了程序里各种各样的功能，他们有信息需要处理的时候，发布信息通知程序（这是他们的主要职责），下面的QA小节里提到了，模块可以依赖一些DOM工具操作方法，但是不应该和系统的其它模块有依赖，一个模块不应该关注如下内容：
</p>
<ol>
    <li>哪个对象或者模块订阅了这个模块发布的信息</li>
    <li>这些对象是客户端对象还是服务器端对象</li>
    <li>多少对象订阅了你的信息</li>
</ol>
<p>&nbsp;</p>
<p>
    <img src="http://pic002.cnblogs.com/images/2011/349491/2011121322335379.gif" alt="" />
</p>
<p>
    <br />
    <strong>Facade抽象</strong>
    应用程序的核心，避免各个模块之间直接通信，它从各模块上订阅信息，也负责授权检测，确保每个模块有用自己单独的授权。
</p>
<p>
    <img src="http://pic002.cnblogs.com/images/2011/349491/2011121322341871.gif" alt="" />
</p>
<p>
    <br />
    Mediator(应用程序核心）使用mediator模式扮演发布/订阅管理器的角色，负责模块管理以及启动/停止模块执行，可以动态加载以及重启有错误的模块。
</p>
<p>
    <img src="http://pic002.cnblogs.com/images/2011/349491/2011121322344391.gif" alt="" />
</p>
<p>
    <br />
    这个架构的结果是：各模块之间没有依赖，因为松耦合的应用，它们可以很容易地被测试和维护，各模块可以很容易地在其它项目里被重用，也可以在不影响程序的情况下动态添加和删除。
    <br />
    <br />
</p>

---
感谢Nicholas Zakas的原始贴，将思想总结在一起,感谢Andree Hansson的technical review,感谢Rebecca Murphey, Justin Meyer, John Hann, Peter Michaux, Paul Irish和Alex Sexton,他们所有的人都提供了和本Session相关的很多资料。

也非常感谢博客园的汤姆大叔(TomXu），将本Session的内容整理成中文版本，如对你有用，请推荐一把。
[阅读原文](http://www.cnblogs.com/TomXu/archive/2011/12/14/2286225.html)


