# JavaScript 模式设计-02


>JQuery源码-Deferred 寥寥代码，犀利锋锐，设计思想，值得学习       ——Binge

<!--more-->

<br>
<br>
<div>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important"> <strong>Deferred的概念请看第一篇</strong>
    </p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        <a target="_blank" href="http://www.cnblogs.com/aaronjs/p/3348569.html" style="color:rgb(26,139,200); text-decoration:none">http://www.cnblogs.com/aaronjs/p/3348569.html</a>
    </p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important"> <strong><span style="color:rgb(255,0,0)">
                <span style="line-height:27px; font-size:18px">
                    ******************
                    <strong>构建Deferred对象时候的流程图**************************</strong>
                </span>
            </span></strong> 
    </p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        <img src="http://images.cnitblog.com/blog/329084/201310/07212436-77fb87abe03846cd857da70ab1698c32.jpg" alt="" style="border:0px"></p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        <strong><span style="line-height:27px; color:rgb(255,0,0); font-size:18px">**********************源码解析**********************</span></strong> 
    </p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">因为callback被剥离出去后，整个deferred就显得非常的精简</p>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <div class="cnblogs_code_toolbar" style="margin-top:5px">
            <span class="cnblogs_code_copy" style="padding-right:5px; line-height:1.5!important">
                <a target="_blank" title="复制代码" style="color:rgb(26,139,200); border:none!important">
                    <img src="http://common.cnblogs.com/images/copycode.gif" alt="复制代码" style="border:none!important"></a>
            </span>
        </div>
        <pre style="margin-top:0px; margin-bottom:0px; white-space:pre-wrap; word-wrap:break-word; font-family:'Courier New'!important"><span style="line-height:1.5!important">jQuery.extend({ 

    Deferred : </span><span style="color:rgb(0,0,255); line-height:1.5!important">function</span><span style="line-height:1.5!important">(){}
    
    when : </span><span style="color:rgb(0,0,255); line-height:1.5!important">function</span><span style="line-height:1.5!important">()

)}</span></pre>
        <div class="cnblogs_code_toolbar" style="margin-top:5px">
            <span class="cnblogs_code_copy" style="padding-right:5px; line-height:1.5!important">
                <a target="_blank" title="复制代码" style="color:rgb(26,139,200); border:none!important">
                    <img src="http://common.cnblogs.com/images/copycode.gif" alt="复制代码" style="border:none!important"></a>
            </span>
        </div>
    </div>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">对于extend的继承这个东东，在之前就提及过jquery如何处理内部jquery与init相互引用this的问题</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        对于JQ的整体架构一定要弄懂&nbsp;
        <a target="_blank" href="http://www.cnblogs.com/aaronjs/p/3278578.html" style="color:rgb(26,139,200); text-decoration:none">http://www.cnblogs.com/aaronjs/p/3278578.html</a>
    </p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">所以当jQuery.extend只有一个参数的时候，其实就是对jQuery静态方法的一个扩展</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">我们在具体看看2个静态方法内部都干了些什么：</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">Deferred整体结构：</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">源码精简了部分代码</p>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <div class="cnblogs_code_toolbar" style="margin-top:5px">
            <span class="cnblogs_code_copy" style="padding-right:5px; line-height:1.5!important">
                <a target="_blank" title="复制代码" style="color:rgb(26,139,200); border:none!important">
                    <img src="http://common.cnblogs.com/images/copycode.gif" alt="复制代码" style="border:none!important"></a>
            </span>
        </div>
        <blockquote style="border:2px solid rgb(239,239,239); color:rgb(51,51,51); margin-left:25px; padding:5px 10px; margin-top:10px; margin-bottom:10px">
            <pre style="margin-top:0px; margin-bottom:0px; white-space:pre-wrap; word-wrap:break-word; font-family:'Courier New'!important">Deferred: <span style="color:rgb(0,0,255); line-height:1.5!important">function</span><span style="color:rgb(0,0,0); line-height:1.5!important">( func ) {</span><span style="color:rgb(0,0,255); line-height:1.5!important">var</span> tuples =<span style="color:rgb(0,0,0); line-height:1.5!important">[</span><span style="color:rgb(0,128,0); line-height:1.5!important">//</span><span style="color:rgb(0,128,0); line-height:1.5!important">action, add listener, listener list, final state</span>
                [ "resolve", "done", jQuery.Callbacks("once memory"), "resolved"<span style="color:rgb(0,0,0); line-height:1.5!important"> ],
                [ </span>"reject", "fail", jQuery.Callbacks("once memory"), "rejected"<span style="color:rgb(0,0,0); line-height:1.5!important"> ],
                [ </span>"notify", "progress", jQuery.Callbacks("memory"<span style="color:rgb(0,0,0); line-height:1.5!important">) ]
            ],
            state </span>= "pending"<span style="color:rgb(0,0,0); line-height:1.5!important">,
            promise </span>=<span style="color:rgb(0,0,0); line-height:1.5!important"> {
                state: </span><span style="color:rgb(0,0,255); line-height:1.5!important">function</span><span style="color:rgb(0,0,0); line-height:1.5!important">() {},
                always: </span><span style="color:rgb(0,0,255); line-height:1.5!important">function</span><span style="color:rgb(0,0,0); line-height:1.5!important">() {},
                then: </span><span style="color:rgb(0,0,255); line-height:1.5!important">function</span>( <span style="color:rgb(0,128,0); line-height:1.5!important">/*</span><span style="color:rgb(0,128,0); line-height:1.5!important">fnDone, fnFail, fnProgress </span><span style="color:rgb(0,128,0); line-height:1.5!important">*/</span><span style="color:rgb(0,0,0); line-height:1.5!important">) { },</span><span style="color:rgb(0,128,0); line-height:1.5!important">//</span><span style="color:rgb(0,128,0); line-height:1.5!important">Get a promise for this deferred</span>
                <span style="color:rgb(0,128,0); line-height:1.5!important">//</span><span style="color:rgb(0,128,0); line-height:1.5!important"> If obj is provided, the promise aspect is added to the object</span>
                promise: <span style="color:rgb(0,0,255); line-height:1.5!important">function</span><span style="color:rgb(0,0,0); line-height:1.5!important">( obj ) {}
            },
            deferred </span>=<span style="color:rgb(0,0,0); line-height:1.5!important"> {};
        jQuery.each( tuples, </span><span style="color:rgb(0,0,255); line-height:1.5!important">function</span><span style="color:rgb(0,0,0); line-height:1.5!important">( i, tuple ) {
            deferred[ tuple[</span>0] + "With" ] =<span style="color:rgb(0,0,0); line-height:1.5!important"> list.fireWith;
        });
        promise.promise( deferred );
        </span><span style="color:rgb(0,128,0); line-height:1.5!important">//</span><span style="color:rgb(0,128,0); line-height:1.5!important">All done!</span>
        <span style="color:rgb(0,0,255); line-height:1.5!important">return</span><span style="color:rgb(0,0,0); line-height:1.5!important"> deferred;
    },</span></pre>
        </blockquote>
        <div class="cnblogs_code_toolbar" style="margin-top:5px">
            <span class="cnblogs_code_copy" style="padding-right:5px; line-height:1.5!important">
                <a target="_blank" title="复制代码" style="color:rgb(26,139,200); border:none!important">
                    <img src="http://common.cnblogs.com/images/copycode.gif" alt="复制代码" style="border:none!important"></a>
            </span>
        </div>
    </div>
    <ol style="padding-left:50px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; line-height:24px">
        <li style="list-style-type:decimal">
            <span style="color:rgb(155,0,211)">
                <span style="color:rgb(0,0,0)">
                    <span style="color:rgb(255,0,0)">显而易见Deferred是个工厂类，返回的是内部构建的deferred对象</span>
                </span>
            </span>
        </li>
        <li style="list-style-type:decimal">
            <span style="color:rgb(255,0,0)">tuples 创建三个$.Callbacks对象，分别表示成功，失败，处理中三种状态</span>
        </li>
        <li style="list-style-type:decimal">
            <span style="color:rgb(255,0,0)">创建了一个promise对象，具有state、always、then、primise方法</span>
        </li>
        <li style="list-style-type:decimal">
            <span style="color:rgb(255,0,0)">扩展primise对象生成最终的Deferred对象，返回该对象</span>
        </li>
    </ol>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">这里其实就是3个处理,但是有个优化代码的地方,就是把共性的代码给抽象出来,通过动态生成了</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <hr style="font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; line-height:24px">
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        <strong>具体源码分析:</strong>
    </p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        <strong>
            <span style="color:rgb(255,0,0)">Deferred自身则围绕这三个对象进行更高层次的抽象</span>
        </strong>
    </p>
    <ul style="list-style-position:initial; margin:0px 0px 10px 45px; padding-left:5px; font-family:微软雅黑,PTSans,Arial,sans-serif; line-height:24px">
        <li style="list-style:inherit!important; font-size:16px">触发回调函数列表执行(函数名)</li>
        <li style="list-style:inherit!important; font-size:16px">添加回调函数（函数名）</li>
        <li style="list-style:inherit!important; font-size:16px">回调函数列表（jQuery.Callbacks对象）</li>
        <li style="list-style:inherit!important; font-size:16px">deferred最终状态（第三组数据除外）</li>
    </ul>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <div class="cnblogs_code_toolbar" style="margin-top:5px">
            <span class="cnblogs_code_copy" style="padding-right:5px; line-height:1.5!important">
                <a target="_blank" title="复制代码" style="color:rgb(26,139,200); border:none!important">
                    <img src="http://common.cnblogs.com/images/copycode.gif" alt="复制代码" style="border:none!important"></a>
            </span>
        </div>
        <pre style="margin-top:0px; margin-bottom:0px; white-space:pre-wrap; word-wrap:break-word; font-family:'Courier New'!important"><span style="color:rgb(0,0,255); line-height:1.5!important">var</span> tuples =<span style="line-height:1.5!important">[</span><span style="color:rgb(0,128,0); line-height:1.5!important">//</span><span style="color:rgb(0,128,0); line-height:1.5!important">action, add listener, listener list, final state</span>
        [ "resolve", "done", jQuery.Callbacks("once memory"), "resolved"<span style="line-height:1.5!important"> ],
        [ </span>"reject", "fail", jQuery.Callbacks("once memory"), "rejected"<span style="line-height:1.5!important"> ],
        [ </span>"notify", "progress", jQuery.Callbacks("memory"<span style="line-height:1.5!important">) ]
    ],</span></pre>
        <div class="cnblogs_code_toolbar" style="margin-top:5px">
            <span class="cnblogs_code_copy" style="padding-right:5px; line-height:1.5!important">
                <a target="_blank" title="复制代码" style="color:rgb(26,139,200); border:none!important">
                    <img src="http://common.cnblogs.com/images/copycode.gif" alt="复制代码" style="border:none!important"></a>
            </span>
        </div>
    </div>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        <span style="color:rgb(255,0,0)">这里抽象出2组阵营：</span>
    </p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        <strong>1组：回调方法/事件订阅&nbsp;&nbsp;</strong>
    </p>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <span style="color:rgb(255,0,0); line-height:1.5!important">done，fail，progress</span>
    </div>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        <strong>2组：通知方法/事件发布&nbsp;&nbsp;&nbsp;&nbsp;</strong>
    </p>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <span style="color:rgb(255,0,0); line-height:1.5!important">resolve，reject，notify，resolveWith，rejectWith，notifyWith</span>
    </div>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">tuples 元素集 其实是把相同有共同特性的代码的给合并成一种结构，然后通过一次处理</p>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <div class="cnblogs_code_toolbar" style="margin-top:5px">
            <span class="cnblogs_code_copy" style="padding-right:5px; line-height:1.5!important">
                <a target="_blank" title="复制代码" style="color:rgb(26,139,200); border:none!important">
                    <img src="http://common.cnblogs.com/images/copycode.gif" alt="复制代码" style="border:none!important"></a>
            </span>
        </div>
        <pre style="margin-top:0px; margin-bottom:0px; white-space:pre-wrap; word-wrap:break-word; font-family:'Courier New'!important">jQuery.each( tuples, <span style="color:rgb(0,0,255); line-height:1.5!important">function</span><span style="line-height:1.5!important">( i, tuple ) {</span><span style="color:rgb(0,0,255); line-height:1.5!important">var</span> list = tuple[ 2<span style="line-height:1.5!important"> ],
                stateString </span>= tuple[ 3<span style="line-height:1.5!important"> ];
            promise[ tuple[</span>1] ] =<span style="line-height:1.5!important">list.add;</span><span style="color:rgb(0,0,255); line-height:1.5!important">if</span><span style="line-height:1.5!important"> ( stateString ) {
                list.add(</span><span style="color:rgb(0,0,255); line-height:1.5!important">function</span><span style="line-height:1.5!important">() {
                    state </span>=<span style="line-height:1.5!important">stateString;</span><span style="color:rgb(0,128,0); line-height:1.5!important">//</span><span style="color:rgb(0,128,0); line-height:1.5!important">[ reject_list | resolve_list ].disable; progress_list.lock</span>
                }, tuples[ i ^ 1 ][ 2 ].disable, tuples[ 2 ][ 2<span style="line-height:1.5!important"> ].lock );
            }
            deferred[ tuple[</span>0] ] = <span style="color:rgb(0,0,255); line-height:1.5!important">function</span><span style="line-height:1.5!important">() {
                deferred[ tuple[</span>0] + "With" ]( <span style="color:rgb(0,0,255); line-height:1.5!important">this</span> === deferred ? promise : <span style="color:rgb(0,0,255); line-height:1.5!important">this</span><span style="line-height:1.5!important">, arguments );</span><span style="color:rgb(0,0,255); line-height:1.5!important">return</span> <span style="color:rgb(0,0,255); line-height:1.5!important">this</span><span style="line-height:1.5!important">;
            };
            deferred[ tuple[</span>0] + "With" ] =<span style="line-height:1.5!important"> list.fireWith;
        });</span></pre>
        <div class="cnblogs_code_toolbar" style="margin-top:5px">
            <span class="cnblogs_code_copy" style="padding-right:5px; line-height:1.5!important">
                <a target="_blank" title="复制代码" style="color:rgb(26,139,200); border:none!important">
                    <img src="http://common.cnblogs.com/images/copycode.gif" alt="复制代码" style="border:none!important"></a>
            </span>
        </div>
    </div>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">对于tuples的3条数据集是分2部分处理的</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <hr style="font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; line-height:24px">
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        <span style="color:rgb(155,0,211)">
            <strong>第一部分将回调函数存入</strong>
        </span>
    </p>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <pre style="margin-top:0px; margin-bottom:0px; white-space:pre-wrap; word-wrap:break-word; font-family:'Courier New'!important">promise[ tuple[1] ] = list.add;</pre>
    </div>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">其实就是给promise赋予3个回调函数</p>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <pre style="margin-top:0px; margin-bottom:0px; white-space:pre-wrap; word-wrap:break-word; font-family:'Courier New'!important">promise.done = $.Callbacks("once memory").add</pre>
    </div>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <pre style="margin-top:0px; margin-bottom:0px; white-space:pre-wrap; word-wrap:break-word; font-family:'Courier New'!important">promise.fail = $.Callbacks("once memory").add</pre>
    </div>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <pre style="margin-top:0px; margin-bottom:0px; white-space:pre-wrap; word-wrap:break-word; font-family:'Courier New'!important">promise.progressl = $.Callbacks("memory").add</pre>
    </div>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">如果存在deferred最终状态</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">默认会预先向doneList,failList中的list添加三个回调函数</p>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <div class="cnblogs_code_toolbar" style="margin-top:5px">
            <span class="cnblogs_code_copy" style="padding-right:5px; line-height:1.5!important">
                <a target="_blank" title="复制代码" style="color:rgb(26,139,200); border:none!important">
                    <img src="http://common.cnblogs.com/images/copycode.gif" alt="复制代码" style="border:none!important"></a>
            </span>
        </div>
        <pre style="margin-top:0px; margin-bottom:0px; white-space:pre-wrap; word-wrap:break-word; font-family:'Courier New'!important"><span style="color:rgb(0,0,255); line-height:1.5!important">if</span><span style="line-height:1.5!important"> ( stateString ) {
    list.add(</span><span style="color:rgb(0,0,255); line-height:1.5!important">function</span><span style="line-height:1.5!important">() {</span><span style="color:rgb(0,128,0); line-height:1.5!important">//</span><span style="color:rgb(0,128,0); line-height:1.5!important">state = [ resolved | rejected ]</span>
        state =<span style="line-height:1.5!important">stateString;</span><span style="color:rgb(0,128,0); line-height:1.5!important">//</span><span style="color:rgb(0,128,0); line-height:1.5!important">[ reject_list | resolve_list ].disable; progress_list.lock</span>
    }, tuples[ i ^ 1 ][ 2 ].disable, tuples[ 2 ][ 2<span style="line-height:1.5!important"> ].lock );
}</span></pre>
        <div class="cnblogs_code_toolbar" style="margin-top:5px">
            <span class="cnblogs_code_copy" style="padding-right:5px; line-height:1.5!important">
                <a target="_blank" title="复制代码" style="color:rgb(26,139,200); border:none!important">
                    <img src="http://common.cnblogs.com/images/copycode.gif" alt="复制代码" style="border:none!important"></a>
            </span>
        </div>
    </div>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        *************************************************************
        <br>这里有个小技巧</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        i ^ 1&nbsp;
        <a target="_blank" href="https://developer.mozilla.org/zh-CN/docs/JavaScript/Reference/Operators/Bitwise_Operators" style="color:rgb(26,139,200); text-decoration:none">按位异或运算符</a>
    </p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">所以实际上第二个传参数是1、0索引对调了，所以取值是failList.disable与doneList.disable</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        *************************************************************
    </p>
    <p align="right" style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        <strong>
            <span style="color:rgb(255,0,0)">通过stateString有值这个条件，预先向doneList,failList中的list添加三个回调函数</span>
        </strong>
    </p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">分别是:</p>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <pre style="margin-top:0px; margin-bottom:0px; white-space:pre-wrap; word-wrap:break-word; font-family:'Courier New'!important"><span style="line-height:1.5!important">doneList : [changeState, failList.disable, <span style="color:rgb(155,0,211); line-height:1.5!important">processList.lock</span>]
failList : [changeState, doneList.disable, <span style="color:rgb(155,0,211); line-height:1.5!important">processList.lock</span>]</span></pre>
    </div>
    <pre style="margin-top:0px; margin-bottom:0px; white-space:pre-wrap; word-wrap:break-word; font-size:16px; line-height:24px; background-color:rgb(255,255,255)"></pre>
    <ul style="list-style-position:initial; margin:0px 0px 10px 45px; padding-left:5px; font-family:微软雅黑,PTSans,Arial,sans-serif; line-height:24px">
        <li style="list-style:inherit!important; font-size:16px">
            changeState 改变状态的匿名函数，deferred的状态，分为三种：pending(初始状态), resolved(解决状态), rejected(拒绝状态)
        </li>
        <li style="list-style:inherit!important; font-size:16px">
            不论deferred对象最终是resolve（还是reject），在首先改变对象状态之后，都会disable另一个函数列表failList(或者doneList)
        </li>
        <li style="list-style:inherit!important; font-size:16px">然后lock processList保持其状态，最后执行剩下的之前done（或者fail）进来的回调函数</li>
    </ul>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        <span style="color:rgb(155,0,211)">所以第一步最终都是围绕这add方法</span>
    </p>
    <ul style="list-style-position:initial; margin:0px 0px 10px 45px; padding-left:5px; font-family:微软雅黑,PTSans,Arial,sans-serif; line-height:24px">
        <li style="list-style:inherit!important; font-size:16px">
            done/fail/是list.add也就是
            <a target="_blank" href="http://www.cnblogs.com/snandy/archive/2012/11/15/2770237.html#add" style="color:rgb(26,139,200); text-decoration:none">callbacks.add</a>
            ，将回调函数存入回调对象中
        </li>
    </ul>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        <span style="color:rgb(155,0,211)">第二部分很简单，给deferred对象扩充6个方法</span>
    </p>
    <ul style="list-style-position:initial; margin:0px 0px 10px 45px; padding-left:5px; font-family:微软雅黑,PTSans,Arial,sans-serif; line-height:24px">
        <li style="list-style:inherit!important; font-size:16px">
            resolve/reject/notify 是&nbsp;
            <a target="_blank" href="http://www.cnblogs.com/snandy/archive/2012/11/15/2770237.html#fireWith" style="color:rgb(26,139,200); text-decoration:none">callbacks.fireWith</a>
            ，执行回调函数
        </li>
        <li style="list-style:inherit!important; font-size:16px">
            resolveWith/rejectWith/notifyWith 是&nbsp;
            <a target="_blank" href="http://www.cnblogs.com/snandy/archive/2012/11/15/2770237.html#fireWith" style="color:rgb(26,139,200); text-decoration:none">callbacks.fireWith</a>
            &nbsp;队列方法引用
        </li>
    </ul>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">最后合并promise到deferred</p>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <pre style="margin-top:0px; margin-bottom:0px; white-space:pre-wrap; word-wrap:break-word; font-family:'Courier New'!important"><span style="line-height:1.5!important">promise.promise( deferred );
jQuery.extend( obj, promise )</span></pre>
    </div>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">所以最终通过工厂方法Deferred构建的异步对象带的所有的方法了</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">return 内部的deferred对象了</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        <a target="_blank" href="http://images.cnitblog.com/blog/329084/201310/03131510-2375bbda3f2f4524a53e5c5df5eccda0.png" style="color:rgb(26,139,200); text-decoration:none">
            <img title="image" src="http://images.cnitblog.com/blog/329084/201310/03131518-8caf4716aa724a66af4e53291e9a408d.png" alt="image" width="414" height="207" border="0" style="border:0px; padding-left:0px; padding-right:0px; display:inline; padding-top:0px"></a>
    </p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">由此可见我们在</p>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <pre style="margin-top:0px; margin-bottom:0px; white-space:pre-wrap; word-wrap:break-word; font-family:'Courier New'!important"><span style="color:rgb(0,0,255); line-height:1.5!important">var</span> defer = $.Deferred(); <span style="color:rgb(0,128,0); line-height:1.5!important">//</span><span style="color:rgb(0,128,0); line-height:1.5!important">构建异步对象</span></pre>
    </div>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">的时候,内部的对象就有了4个属性方法了</p>
    <ol style="padding-left:50px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; line-height:24px">
        <li style="list-style-type:decimal">
            deferred: Object
            <ol style="padding-left:50px">
                <li style="list-style-type:decimal">always: function () {</li>
                <li style="list-style-type:decimal">done: function () {</li>
                <li style="list-style-type:decimal">fail: function () {</li>
                <li style="list-style-type:decimal">notify: function () {</li>
                <li style="list-style-type:decimal">notifyWith: function ( context, args ) {</li>
                <li style="list-style-type:decimal">pipe: function ( /* fnDone, fnFail, fnProgress */ ) {</li>
                <li style="list-style-type:decimal">progress: function () {</li>
                <li style="list-style-type:decimal">promise: function ( obj ) {</li>
                <li style="list-style-type:decimal">reject: function () {</li>
                <li style="list-style-type:decimal">rejectWith: function ( context, args ) {</li>
                <li style="list-style-type:decimal">resolve: function () {</li>
                <li style="list-style-type:decimal">resolveWith: function ( context, args ) {</li>
                <li style="list-style-type:decimal">state: function () {</li>
                <li style="list-style-type:decimal">then: function ( /* fnDone, fnFail, fnProgress */ ) {</li>
            </ol>
        </li>
        <li style="list-style-type:decimal">
            promise: Object
            <ol style="padding-left:50px">
                <li style="list-style-type:decimal">always: function () {</li>
                <li style="list-style-type:decimal">done: function () {</li>
                <li style="list-style-type:decimal">fail: function () {</li>
                <li style="list-style-type:decimal">pipe: function ( /* fnDone, fnFail, fnProgress */ ) {</li>
                <li style="list-style-type:decimal">progress: function () {</li>
                <li style="list-style-type:decimal">promise: function ( obj ) {</li>
                <li style="list-style-type:decimal">state: function () {</li>
                <li style="list-style-type:decimal">then: function ( /* fnDone, fnFail, fnProgress */ ) {</li>
            </ol>
        </li>
        <li style="list-style-type:decimal">state: "pending"</li>
        <li style="list-style-type:decimal">tuples: Array[3]</li>
    </ol>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">构造图</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        <img src="http://images.cnitblog.com/blog/329084/201310/07212451-50defafec7bc468f817adb82476c9bb9.jpg" alt="" style="border:0px"></p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">以上只是在初始化构建的时候，我们往下看看动态执行时候的处理</p>
    <hr style="font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; line-height:24px">
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        <strong>
            <span style="color:rgb(255,0,0)">*****************执行期***********************</span>
        </strong>
    </p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">一个最简单的demo为例子</p>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <div class="cnblogs_code_toolbar" style="margin-top:5px">
            <span class="cnblogs_code_copy" style="padding-right:5px; line-height:1.5!important">
                <a target="_blank" title="复制代码" style="color:rgb(26,139,200); border:none!important">
                    <img src="http://common.cnblogs.com/images/copycode.gif" alt="复制代码" style="border:none!important"></a>
            </span>
        </div>
        <pre style="margin-top:0px; margin-bottom:0px; white-space:pre-wrap; word-wrap:break-word; font-family:'Courier New'!important"><span style="color:rgb(0,0,255); line-height:1.5!important">var</span> d =<span style="line-height:1.5!important"> $.Deferred();

 setTimeout(</span><span style="color:rgb(0,0,255); line-height:1.5!important">function</span><span style="line-height:1.5!important">(){
        d.resolve(</span>22<span style="line-height:1.5!important">)
  },</span>0<span style="line-height:1.5!important">);

 d.then(</span><span style="color:rgb(0,0,255); line-height:1.5!important">function</span><span style="line-height:1.5!important">(val){
      console.log(val);
 })</span></pre>
        <div class="cnblogs_code_toolbar" style="margin-top:5px">
            <span class="cnblogs_code_copy" style="padding-right:5px; line-height:1.5!important">
                <a target="_blank" title="复制代码" style="color:rgb(26,139,200); border:none!important">
                    <img src="http://common.cnblogs.com/images/copycode.gif" alt="复制代码" style="border:none!important"></a>
            </span>
        </div>
    </div>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <span style="color:rgb(155,0,211); line-height:1.5!important">
            当延迟对象被 resolved 时，任何通过 deferred.then或deferred.done 添加的 doneCallbacks，都会被调用。回调函数的执行顺序和它们被添加的顺序是一样的。传递给 deferred.resolve() 的 args 参数，会传给每个回调函数。当延迟对象进入 resolved 状态后，再添加的任何 doneCallbacks，当它们被添加时，就会被立刻执行，并带上传入给
 .resolve()的参数
        </span>
    </div>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">换句话说，我们调用d.resolve(22) 就等于是调用</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">匿名函数并传入参数值 22</p>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <pre style="margin-top:0px; margin-bottom:0px; white-space:pre-wrap; word-wrap:break-word; font-family:'Courier New'!important"><span style="color:rgb(0,0,255); line-height:1.5!important">function</span><span style="line-height:1.5!important">(val){
      console.log(val); //22
 }</span></pre>
    </div>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">当前实际的使用中会有各种复杂的组合情况，但是整的外部调用流程就是这样的</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        <span style="color:rgb(255,0,0)">
            <strong>***************** resolve的实现 *******************</strong>
        </span>
    </p>
    <hr style="font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; line-height:24px">
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        <span style="color:rgb(255,0,0)">我们回顾下，其实Deferred对象，内部的实现还是Callbacks对象，只是在外面再封装了一层API，供接口调用</span>
    </p>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <pre style="margin-top:0px; margin-bottom:0px; white-space:pre-wrap; word-wrap:break-word; font-family:'Courier New'!important">d.resolve(22)</pre>
    </div>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">实际上调用的就是通过这个代码生成的</p>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <pre style="margin-top:0px; margin-bottom:0px; white-space:pre-wrap; word-wrap:break-word; font-family:'Courier New'!important">deferred[ tuple[0] ] = <span style="color:rgb(0,0,255); line-height:1.5!important">function</span><span style="line-height:1.5!important">() {
    deferred[ tuple[</span>0] + "With" ]( <span style="color:rgb(0,0,255); line-height:1.5!important">this</span> === deferred ? promise : <span style="color:rgb(0,0,255); line-height:1.5!important">this</span><span style="line-height:1.5!important">, arguments );</span><span style="color:rgb(0,0,255); line-height:1.5!important">return</span> <span style="color:rgb(0,0,255); line-height:1.5!important">this</span><span style="line-height:1.5!important">;
};</span></pre>
        <pre style="margin-top:0px; margin-bottom:0px; white-space:pre-wrap; word-wrap:break-word; font-family:'Courier New'!important">deferred[ tuple[0] + "With" ] = list.fireWith;</pre>
    </div>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <pre style="margin-top:0px; margin-bottom:0px; white-space:pre-wrap; word-wrap:break-word; font-family:'Courier New'!important">deferred.resolveWith()</pre>
    </div>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        最终执行的就是
        <span style="color:rgb(155,0,211)">&nbsp;list.fireWith</span>
    </p>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <span style="color:rgb(255,0,0); line-height:1.5!important">callbacks.fireWith()</span>
    </div>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        所以最终又回到回调对象
        <span style="color:rgb(255,0,0)">callbacks中的私有方法</span>
        <span style="color:rgb(255,0,0)">fire()了</span>
    </p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <hr style="font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; line-height:24px">
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">Callbacks会通过</p>
    <h3 style="margin:10px 0px; font-size:14px; color:rgb(102,102,102); font-family:Verdana; line-height:24px">
        <a name="t1"></a>
        <a target="_blank" href="http://www.css88.com/jqapi-1.9/callbacks.add/" style="color:rgb(26,139,200); text-decoration:none">callbacks.add()</a>
    </h3>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        把回调函数给注册到内部的
        <span style="color:rgb(255,0,0)">list = []</span>
        上,我们回来过看看
    </p>
    <h3 style="margin:10px 0px; font-size:14px; color:rgb(102,102,102); font-family:Verdana; line-height:24px">
        <a name="t2"></a>
        <a target="_blank" href="http://www.css88.com/jqapi-1.9/deferred.then/" style="color:rgb(26,139,200); text-decoration:none">deferred.then()</a>
    </h3>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <pre style="margin-top:0px; margin-bottom:0px; white-space:pre-wrap; word-wrap:break-word; font-family:'Courier New'!important">d.then(<span style="color:rgb(0,0,255); line-height:1.5!important">function</span><span style="line-height:1.5!important">(val){
      console.log(val);
 })</span></pre>
    </div>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        <span style="color:rgb(255,0,0)">
            <strong>***************** then的实现 *******************</strong>
        </span>
    </p>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <div class="cnblogs_code_toolbar" style="margin-top:5px">
            <span class="cnblogs_code_copy" style="padding-right:5px; line-height:1.5!important">
                <a target="_blank" title="复制代码" style="color:rgb(26,139,200); border:none!important">
                    <img src="http://common.cnblogs.com/images/copycode.gif" alt="复制代码" style="border:none!important"></a>
            </span>
        </div>
        <pre style="margin-top:0px; margin-bottom:0px; white-space:pre-wrap; word-wrap:break-word; font-family:'Courier New'!important">then: <span style="color:rgb(0,0,255); line-height:1.5!important">function</span>( <span style="color:rgb(0,128,0); line-height:1.5!important">/*</span><span style="color:rgb(0,128,0); line-height:1.5!important">fnDone, fnFail, fnProgress </span><span style="color:rgb(0,128,0); line-height:1.5!important">*/</span><span style="line-height:1.5!important">) {</span><span style="color:rgb(0,0,255); line-height:1.5!important">var</span> fns =<span style="line-height:1.5!important">arguments;</span><span style="color:rgb(0,0,255); line-height:1.5!important">return</span> jQuery.Deferred(<span style="color:rgb(0,0,255); line-height:1.5!important">function</span><span style="line-height:1.5!important">( newDefer ) {
        jQuery.each( tuples, </span><span style="color:rgb(0,0,255); line-height:1.5!important">function</span><span style="line-height:1.5!important">( i, tuple ) {</span><span style="color:rgb(0,0,255); line-height:1.5!important">var</span> action = tuple[ 0<span style="line-height:1.5!important"> ],
                fn </span>= jQuery.isFunction( fns[ i ] ) &amp;&amp;<span style="line-height:1.5!important">fns[ i ];</span><span style="color:rgb(0,128,0); line-height:1.5!important">//</span><span style="color:rgb(0,128,0); line-height:1.5!important"> deferred[ done | fail | progress ] for forwarding actions to newDefer</span>
            deferred[ tuple[1] ](<span style="color:rgb(0,0,255); line-height:1.5!important">function</span><span style="line-height:1.5!important">() {</span><span style="color:rgb(0,128,0); line-height:1.5!important">//</span><span style="color:rgb(0,128,0); line-height:1.5!important">省略............</span>
<span style="line-height:1.5!important">            });
        });
        fns </span>= <span style="color:rgb(0,0,255); line-height:1.5!important">null</span><span style="line-height:1.5!important">;
    }).promise();
},</span></pre>
        <div class="cnblogs_code_toolbar" style="margin-top:5px">
            <span class="cnblogs_code_copy" style="padding-right:5px; line-height:1.5!important">
                <a target="_blank" title="复制代码" style="color:rgb(26,139,200); border:none!important">
                    <img src="http://common.cnblogs.com/images/copycode.gif" alt="复制代码" style="border:none!important"></a>
            </span>
        </div>
    </div>
    <ul style="list-style-position:initial; margin:0px 0px 10px 45px; padding-left:5px; font-family:微软雅黑,PTSans,Arial,sans-serif; line-height:24px">
        <li style="list-style:inherit!important; font-size:16px">递归jQuery.Deferred</li>
        <li style="list-style:inherit!important; font-size:16px">传递了func</li>
        <li style="list-style:inherit!important; font-size:16px">链式调用了promise()</li>
    </ul>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <hr style="font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; line-height:24px">
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        <strong>因为在异步对象的方法都是嵌套找作用域属性方法的</strong>
    </p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        <strong>这里我额外的提及一下作用域</strong>
    </p>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <pre style="margin-top:0px; margin-bottom:0px; white-space:pre-wrap; word-wrap:break-word; font-family:'Courier New'!important"><span style="color:rgb(0,0,255); line-height:1.5!important">var</span> d = $.Deferred();</pre>
    </div>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">这个异步对象d是作用域是如何呢？</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">第一层：无可争议，浏览器环境下最外层是 window</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">第二层：jquery本身是一个闭包</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">第三层: Deferred工厂方法产生的作用域</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">如果用d.then()方法呢？</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">很明显then方法又是嵌套在内部的函数，所以执行的时候都默认会包含以上三层作用域+自己本身函数产生的作用域了</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">我们用个简单图描绘下</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        <a target="_blank" href="http://images.cnitblog.com/blog/329084/201310/03210232-7602c75e6dab41b6bff219c0a288f7d3.jpg" style="color:rgb(26,139,200); text-decoration:none">
            <img title="演示文稿1" src="http://images.cnitblog.com/blog/329084/201310/03210240-ff84208212224cbe8700048def6bf0dc.jpg" alt="演示文稿1" width="680" height="384" border="0" style="border:0px; padding-left:0px; padding-right:0px; display:inline; padding-top:0px"></a>
    </p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        <strong>
            <span style="color:rgb(155,0,211)">根据规则，在最内部的函数能够访问上层作用域的所有的变量</span>
        </strong>
    </p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <hr style="font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; line-height:24px">
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        <strong>我们先从使用的层面去考虑下结构设计:</strong>
    </p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        <strong>demo 1</strong>
    </p>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <div class="cnblogs_code_toolbar" style="margin-top:5px">
            <span class="cnblogs_code_copy" style="padding-right:5px; line-height:1.5!important">
                <a target="_blank" title="复制代码" style="color:rgb(26,139,200); border:none!important">
                    <img src="http://common.cnblogs.com/images/copycode.gif" alt="复制代码" style="border:none!important"></a>
            </span>
        </div>
        <pre style="margin-top:0px; margin-bottom:0px; white-space:pre-wrap; word-wrap:break-word; font-family:'Courier New'!important"><span style="color:rgb(0,0,255); line-height:1.5!important">var</span> defer =<span style="line-height:1.5!important">$.Deferred();</span><span style="color:rgb(0,0,255); line-height:1.5!important">var</span> filtered = defer.then(<span style="color:rgb(0,0,255); line-height:1.5!important">function</span><span style="line-height:1.5!important">( value ) {</span><span style="color:rgb(0,0,255); line-height:1.5!important">return</span> value * 2<span style="line-height:1.5!important">;
      });

  defer.resolve( </span>5<span style="line-height:1.5!important"> );

  filtered.done(</span><span style="color:rgb(0,0,255); line-height:1.5!important">function</span><span style="line-height:1.5!important">( value ) {
      console.log(value) //10
  });</span></pre>
        <div class="cnblogs_code_toolbar" style="margin-top:5px">
            <span class="cnblogs_code_copy" style="padding-right:5px; line-height:1.5!important">
                <a target="_blank" title="复制代码" style="color:rgb(26,139,200); border:none!important">
                    <img src="http://common.cnblogs.com/images/copycode.gif" alt="复制代码" style="border:none!important"></a>
            </span>
        </div>
    </div>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        <strong>demo 2</strong>
    </p>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <div class="cnblogs_code_toolbar" style="margin-top:5px">
            <span class="cnblogs_code_copy" style="padding-right:5px; line-height:1.5!important">
                <a target="_blank" title="复制代码" style="color:rgb(26,139,200); border:none!important">
                    <img src="http://common.cnblogs.com/images/copycode.gif" alt="复制代码" style="border:none!important"></a>
            </span>
        </div>
        <pre style="margin-top:0px; margin-bottom:0px; white-space:pre-wrap; word-wrap:break-word; font-family:'Courier New'!important"><span style="color:rgb(0,0,255); line-height:1.5!important">var</span> defer =<span style="line-height:1.5!important"> $.Deferred();

  defer.then(</span><span style="color:rgb(0,0,255); line-height:1.5!important">function</span><span style="line-height:1.5!important">(value) {</span><span style="color:rgb(0,0,255); line-height:1.5!important">return</span> value * 2<span style="line-height:1.5!important">;
  }).then(</span><span style="color:rgb(0,0,255); line-height:1.5!important">function</span><span style="line-height:1.5!important">(value) {</span><span style="color:rgb(0,0,255); line-height:1.5!important">return</span> value * 2<span style="line-height:1.5!important">;
  }).done(</span><span style="color:rgb(0,0,255); line-height:1.5!important">function</span><span style="line-height:1.5!important">(value) {
      alert(value)  </span><span style="color:rgb(0,128,0); line-height:1.5!important">//</span><span style="color:rgb(0,128,0); line-height:1.5!important">20</span>
<span style="line-height:1.5!important">  });

  defer.resolve( </span>5 );</pre>
        <div class="cnblogs_code_toolbar" style="margin-top:5px">
            <span class="cnblogs_code_copy" style="padding-right:5px; line-height:1.5!important">
                <a target="_blank" title="复制代码" style="color:rgb(26,139,200); border:none!important">
                    <img src="http://common.cnblogs.com/images/copycode.gif" alt="复制代码" style="border:none!important"></a>
            </span>
        </div>
    </div>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        <strong>其实这里就是涉及到defer.then().then().done()&nbsp; 链式调用了</strong>
    </p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        <strong>API是这么定义的:</strong>
    </p>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <pre style="margin-top:0px; margin-bottom:0px; white-space:pre-wrap; word-wrap:break-word; font-family:'Courier New'!important">deferred.then( doneFilter [, failFilter ] [, progressFilter ] )</pre>
    </div>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <span style="color:rgb(155,0,211); line-height:1.5!important">
            从jQuery 1.8开始, 方法返回一个新的promise（承诺），通过一个函数，可以过滤deferred（延迟）的状态和值。替换现在过时的deferred.pipe()方法。 doneFilter 和 failFilter函数过滤原deferred（延迟）的解决/拒绝的状态和值。 progressFilter 函数过滤器的任何调用到原有的deferred（延迟）的notify 和 notifyWith的方法。
 这些过滤器函数可以返回一个新的值传递给的 promise（承诺）的.done() 或 .fail() 回调，或他们可以返回另一个观察的对象（递延，承诺等）传递给它的解决/拒绝的状态和值promise（承诺）的回调。 如果过滤函数是空，或没有指定，promise（承诺）将得到与原来值相同解决（resolved）或拒绝（rejected）。
        </span>
    </div>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">我们抓住几点：</p>
    <ul style="list-style-position:initial; margin:0px 0px 10px 45px; padding-left:5px; font-family:微软雅黑,PTSans,Arial,sans-serif; line-height:24px">
        <li style="list-style:inherit!important; font-size:16px">返回的是新的promise对象</li>
        <li style="list-style:inherit!important; font-size:16px">内部有一个滤器函数</li>
    </ul>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">从demo 1中我们就能看到</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        经过x.then()方法处理的代码中返回的this（filtered ）,不是原来的$.Deferred()所有产生的那个异步对象(defer )了
    </p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        <span style="color:rgb(255,0,0)">所以，每经过一个then那么内部处理的this都要被重新设置，那么为什么要这样处理呢？</span>
    </p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">源码</p>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <div class="cnblogs_code_toolbar" style="margin-top:5px">
            <span class="cnblogs_code_copy" style="padding-right:5px; line-height:1.5!important">
                <a target="_blank" title="复制代码" style="color:rgb(26,139,200); border:none!important">
                    <img src="http://common.cnblogs.com/images/copycode.gif" alt="复制代码" style="border:none!important"></a>
            </span>
        </div>
        <pre style="margin-top:0px; margin-bottom:0px; white-space:pre-wrap; word-wrap:break-word; font-family:'Courier New'!important">then: <span style="color:rgb(0,0,255); line-height:1.5!important">function</span>( <span style="color:rgb(0,128,0); line-height:1.5!important">/*</span><span style="color:rgb(0,128,0); line-height:1.5!important">fnDone, fnFail, fnProgress </span><span style="color:rgb(0,128,0); line-height:1.5!important">*/</span><span style="line-height:1.5!important">) {</span><span style="color:rgb(0,0,255); line-height:1.5!important">var</span> fns =<span style="line-height:1.5!important">arguments;</span><span style="color:rgb(0,128,0); line-height:1.5!important">//</span><span style="color:rgb(0,128,0); line-height:1.5!important">分别为deferred的三个callbacklist添加回调函数，根据fn的是否是函数，分为两种情况</span>
                    <span style="color:rgb(0,0,255); line-height:1.5!important">return</span> jQuery.Deferred(<span style="color:rgb(0,0,255); line-height:1.5!important">function</span><span style="line-height:1.5!important">( newDefer ) {
                        jQuery.each( tuples, </span><span style="color:rgb(0,0,255); line-height:1.5!important">function</span><span style="line-height:1.5!important">( i, tuple ) {</span><span style="color:rgb(0,0,255); line-height:1.5!important">var</span> action = tuple[ 0<span style="line-height:1.5!important"> ],
                                fn </span>= jQuery.isFunction( fns[ i ] ) &amp;&amp;<span style="line-height:1.5!important">fns[ i ];</span><span style="color:rgb(0,128,0); line-height:1.5!important">//</span><span style="color:rgb(0,128,0); line-height:1.5!important"> deferred[ done | fail | progress ] for forwarding actions to newDefer</span>
                            deferred[ tuple[1] ](<span style="color:rgb(0,0,255); line-height:1.5!important">function</span><span style="line-height:1.5!important">() {</span><span style="color:rgb(0,0,255); line-height:1.5!important">var</span> returned = fn &amp;&amp; fn.apply( <span style="color:rgb(0,0,255); line-height:1.5!important">this</span><span style="line-height:1.5!important">, arguments );</span><span style="color:rgb(0,0,255); line-height:1.5!important">if</span> ( returned &amp;&amp;<span style="line-height:1.5!important"> jQuery.isFunction( returned.promise ) ) {
                                    returned.promise()
                                        .done( newDefer.resolve )
                                        .fail( newDefer.reject )
                                        .progress( newDefer.notify );
                                } </span><span style="color:rgb(0,0,255); line-height:1.5!important">else</span><span style="line-height:1.5!important"> {
                                    newDefer[ action </span>+ "With" ]( <span style="color:rgb(0,0,255); line-height:1.5!important">this</span> === promise ? newDefer.promise() : <span style="color:rgb(0,0,255); line-height:1.5!important">this</span>, fn ?<span style="line-height:1.5!important"> [ returned ] : arguments );
                                }
                            });
                        });
                        fns </span>= <span style="color:rgb(0,0,255); line-height:1.5!important">null</span><span style="line-height:1.5!important">;
                    }).promise();
                },</span></pre>
        <div class="cnblogs_code_toolbar" style="margin-top:5px">
            <span class="cnblogs_code_copy" style="padding-right:5px; line-height:1.5!important">
                <a target="_blank" title="复制代码" style="color:rgb(26,139,200); border:none!important">
                    <img src="http://common.cnblogs.com/images/copycode.gif" alt="复制代码" style="border:none!important"></a>
            </span>
        </div>
    </div>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">在Deferred传递实参的时候，支持一个flag，jQuery.Deferred(func)</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">传递一个回调函数</p>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <pre style="margin-top:0px; margin-bottom:0px; white-space:pre-wrap; word-wrap:break-word; font-family:'Courier New'!important"><span style="color:rgb(0,128,0); line-height:1.5!important">//</span><span style="color:rgb(0,128,0); line-height:1.5!important">Call given func if any</span>
<span style="color:rgb(0,0,255); line-height:1.5!important">if</span><span style="line-height:1.5!important"> ( func ) {
    func.call( deferred, deferred );
}</span></pre>
    </div>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">所以newDefer可以看作是</p>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <pre style="margin-top:0px; margin-bottom:0px; white-space:pre-wrap; word-wrap:break-word; font-family:'Courier New'!important">newDefer = $.Deferred();</pre>
    </div>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">那么func回调的处理的就是过滤函数了</p>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <div class="cnblogs_code_toolbar" style="margin-top:5px">
            <span class="cnblogs_code_copy" style="padding-right:5px; line-height:1.5!important">
                <a target="_blank" title="复制代码" style="color:rgb(26,139,200); border:none!important">
                    <img src="http://common.cnblogs.com/images/copycode.gif" alt="复制代码" style="border:none!important"></a>
            </span>
        </div>
        <pre style="margin-top:0px; margin-bottom:0px; white-space:pre-wrap; word-wrap:break-word; font-family:'Courier New'!important">deferred[ tuple[1] ](<span style="color:rgb(0,0,255); line-height:1.5!important">function</span><span style="line-height:1.5!important">() {</span><span style="color:rgb(0,0,255); line-height:1.5!important">var</span> returned = fn &amp;&amp; fn.apply( <span style="color:rgb(0,0,255); line-height:1.5!important">this</span><span style="line-height:1.5!important">, arguments );</span><span style="color:rgb(0,0,255); line-height:1.5!important">if</span> ( returned &amp;&amp;<span style="line-height:1.5!important"> jQuery.isFunction( returned.promise ) ) {
        returned.promise()
            .done( newDefer.resolve )
            .fail( newDefer.reject )
            .progress( newDefer.notify );
    } </span><span style="color:rgb(0,0,255); line-height:1.5!important">else</span><span style="line-height:1.5!important"> {
        newDefer[ action </span>+ "With" ]( <span style="color:rgb(0,0,255); line-height:1.5!important">this</span> === promise ? newDefer.promise() : <span style="color:rgb(0,0,255); line-height:1.5!important">this</span>, fn ?<span style="line-height:1.5!important"> [ returned ] : arguments );
    }
});</span></pre>
        <div class="cnblogs_code_toolbar" style="margin-top:5px">
            <span class="cnblogs_code_copy" style="padding-right:5px; line-height:1.5!important">
                <a target="_blank" title="复制代码" style="color:rgb(26,139,200); border:none!important">
                    <img src="http://common.cnblogs.com/images/copycode.gif" alt="复制代码" style="border:none!important"></a>
            </span>
        </div>
    </div>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">这里其实也有编译函数的概念，讲未来要执行的代码，预先通过闭包函数也保存起来，使其访问各自的作用域</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">第一步</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">分解tuples元素集</p>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <pre style="margin-top:0px; margin-bottom:0px; white-space:pre-wrap; word-wrap:break-word; font-family:'Courier New'!important">jQuery.each( tuples, <span style="color:rgb(0,0,255); line-height:1.5!important">function</span><span style="line-height:1.5!important">( i, tuple ) {</span><span style="color:rgb(0,128,0); line-height:1.5!important">//</span><span style="color:rgb(0,128,0); line-height:1.5!important">过滤函数第一步处理</span>
}）</pre>
    </div>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">第二步</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        分别为deferred[ done | fail | progress ]执行对应的add方法，增加过滤函数给done | fail | progress 方法
    </p>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <pre style="margin-top:0px; margin-bottom:0px; white-space:pre-wrap; word-wrap:break-word; font-family:'Courier New'!important">deferred[ tuple[1] ]（</pre>
        <span style="color:rgb(255,0,0); line-height:1.5!important">传入过滤函数</span>
        <pre style="margin-top:0px; margin-bottom:0px; white-space:pre-wrap; word-wrap:break-word; font-family:'Courier New'!important">）//过滤函数 执行的时候在分解</pre>
    </div>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">代码即</p>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <pre style="margin-top:0px; margin-bottom:0px; white-space:pre-wrap; word-wrap:break-word; font-family:'Courier New'!important">deferred[done] = list.add = callback.add</pre>
    </div>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">第三步</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">返回return jQuery.Deferred().promise()</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        此时构建了一个新的Deferred对象，但是返回的的是经过promise()方法处理后的，返回的是一个受限的promise对象
    </p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">所以整个then方法就处理了2个事情</p>
    <ul style="list-style-position:initial; margin:0px 0px 10px 45px; padding-left:5px; font-family:微软雅黑,PTSans,Arial,sans-serif; line-height:24px">
        <li style="list-style:inherit!important; font-size:16px">构建一个新的deferred对象，返回受限的promise对象</li>
        <li style="list-style:inherit!important; font-size:16px">给父deferred对象的[ done | fail | progress ]方法都增加一个过滤函数的方法</li>
    </ul>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">我们知道defer.then方法返回的是一个新的jQuery.Deferred().promise()对象</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        那么我们把defer.then返回的称之为子对象,那么如何与父对象
        <span style="color:rgb(0,0,255)">var</span>
        &nbsp;defer =&nbsp;$.Deferred()&nbsp;关联的起来的
    </p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">我看看源码</p>
    <div class="cnblogs_code" style="background-color:rgb(245,245,245); border:1px solid rgb(204,204,204); padding:5px; overflow:auto; margin:5px 0px; line-height:24px; font-family:'Courier New'!important">
        <pre style="margin-top:0px; margin-bottom:0px; white-space:pre-wrap; word-wrap:break-word; font-family:'Courier New'!important">deferred[ tuple[1] ](<span style="color:rgb(0,128,0); line-height:1.5!important">//</span><span style="color:rgb(0,128,0); line-height:1.5!important">过滤函数//)</span></pre>
    </div>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        deferred其实就是根级父对象的引用,所以就嵌套再深,其实都是调用了父对象deferred[ done | fail | progress 执行add罢了
    </p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        <img src="http://images.cnitblog.com/blog/329084/201310/07212521-e2749ec11b1c44049e104d9124320667.jpg" alt="" style="border:0px"></p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        从图中就能很明显的看到 2个不同的deferred对象中 done fail progress分别都保存了不同的处理回调了
    </p>
    <hr style="font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; line-height:24px">
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <h6 style="font-family:微软雅黑,PTSans,Arial,sans-serif; line-height:24px">
        <a name="t3"></a>
        <span style="font-size:14px">deferred.resolve( args )</span>
    </h6>
    <ul style="list-style-position:initial; margin:0px 0px 10px 45px; padding-left:5px; font-family:微软雅黑,PTSans,Arial,sans-serif; line-height:24px">
        <li style="list-style:inherit!important; font-size:16px">
            当延迟对象被 resolved 时，任何通过&nbsp;
            <code><a target="_blank" href="http://www.css88.com/jqapi-1.9/deferred.then" style="color:rgb(26,139,200); text-decoration:none">deferred.then</a></code>
            或
            <a target="_blank" href="http://www.css88.com/jqapi-1.9/deferred.done" style="color:rgb(26,139,200); text-decoration:none">
                <code>deferred.done</code>
            </a>
            &nbsp;添加的
 doneCallbacks，都会被调用
        </li>
        <li style="list-style:inherit!important; font-size:16px">回调函数的执行顺序和它们被添加的顺序是一样的</li>
        <li style="list-style:inherit!important; font-size:16px">
            传递给&nbsp;
            <code>deferred.resolve()</code>
            &nbsp;的&nbsp;
            <code>args</code>
            &nbsp;参数，会传给每个回调函数
        </li>
        <li style="list-style:inherit!important; font-size:16px">
            当延迟对象进入 resolved 状态后，再添加的任何 doneCallbacks，当它们被添加时，就会被立刻执行，并带上传入给
            <code>.resolve()</code>
            的参数
        </li>
    </ul>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">流程如图</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        <img src="http://images.cnitblog.com/blog/329084/201310/07212507-bba369612b92425a908b5bbb2977b586.jpg" alt="" style="border:0px"></p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">流程解析：</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">1 执行fire()方法，递归执行list所有包含的处理方法</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">2 执行了默认的 changeState, disable, lock 方法、</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">3 执行过滤函数</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        &nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 根据 var returned = fn.apply( this, arguments )的返回值(称作returnReferred)是否是deferred对象
    </p>
    <ul style="list-style-position:initial; margin:0px 0px 10px 45px; padding-left:5px; font-family:微软雅黑,PTSans,Arial,sans-serif; line-height:24px">
        <li style="list-style:inherit!important; font-size:16px">
            返回值是deferred对象，那么在returnReferred对象的三个回调函数列表中添加newDeferred的resolve(reject,notify)方法，也就是说newDeferrred的执行依赖returnDeferred的状态
        </li>
        <li style="list-style:inherit!important; font-size:16px">
            不是函数的情况（如值为undefined或者null等），直接链接到newDeferred的resolve(reject,notify)方法，也就是说&nbsp; newDeferrred的执行依赖外层的调用者deferred的状态或者说是执行动作（resolve还是reject或者是notify）&nbsp; 此时deferred.then()相当于将自己的callbacklist和newDeferred的callbacklist连接起来
        </li>
    </ul>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        <a target="_blank" href="http://images.cnitblog.com/blog/329084/201310/07224100-f83ea4187cc74c05baec6eaf437efb2c.png" style="color:rgb(26,139,200); text-decoration:none">
            <img title="image" src="http://images.cnitblog.com/blog/329084/201310/07224128-b173cd377afd4ea9bf6caf1bf449872e.png" alt="image" width="689" height="369" border="0" style="border:0px; padding-left:0px; padding-right:0px; display:inline; padding-top:0px"></a>
    </p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">下面就是嵌套deferred对象的划分了</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        <a target="_blank" href="http://images.cnitblog.com/blog/329084/201310/08084453-a88aa66681f542548f2878fb4e932a56.png" style="color:rgb(26,139,200); text-decoration:none">
            <img title="image" src="http://images.cnitblog.com/blog/329084/201310/08084454-e443324b4f894f809174a49590366a0d.png" alt="image" width="382" height="480" style="border:0px; display:inline"></a>
    </p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">源码还是要靠自己去折腾的</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">思想的提高比较难的，我们可以借鉴设计的思路，代码书写方式都是有益无害的</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">流程的分析已经比较透彻了，下一章在讲解when的实现</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">&nbsp;</p>
    <p style="line-height:24px; font-family:微软雅黑,PTSans,Arial,sans-serif; font-size:16px; margin:10px auto!important">
        写这东西太耗精力了，如果对您有帮助，请点击
        <strong>
            <span style="line-height:27px; color:rgb(255,0,0); font-size:18px">推荐支持</span>
        </strong>
        一下……………
    </p>
    <br></div>
非常感谢《 jQuery 2.0.3 源码分析 Deferred（最细的实现剖析，带图）》转载文章和Aaron的教程图示，如对你有用，请推荐一把。
[阅读原文](http://blog.csdn.net/yixuan_1/article/details/12836379)

<script type="text/javascript" src="/js/jquery.min.js"></script>
<script type="text/javascript">
$(function(){
    $('img').each(function(index,element){
        if($(this).attr('alt') == '复制代码'){
            $(this).hide();
        }
    });    
});

</script>
