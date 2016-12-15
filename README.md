#继承关系
The stage, layers, groups, and shapes all extend Node.     

Stage->Container-Node
Group->Container->Node
Layer->BaseLayer->Container->Node    
Rect->Shape->Node  

#使用时，注意事项
Stage下只能添加Layer，不是Layer会报错的。

Konva中的集合对象
Konva.Collection 继承自数组，主要提供了each方法。
把Konva.Node.prototype上的方法全部map到Konva.Collection.prototype上。
这样的话：如果一个Collection对象有10个Node对象，调用collection.method()时，会遍历该集合分别调用Node对象的相应方法。
Konva.Collection.mapMethods(Konva.Node);


Konva.idCounter++ 全局计数器，用来设置id

1、事件机制，支持自定义事件
事件机制相关的代码在基类Node.js中，记住 Node 与 Stage 支持的事件类型不一样。虽然 Stage 是 Node 的子类。比如：click contentClick
on:
   注册事件，处理程序保存到了eventListeners对象中(每一个Node都有这么一个对象)。暂不考虑事件代理。
   on("click.namespace",handler) 
   eventListener={
     "click":[
		{
			name:"namespace",
			handler:handler
	    }
     ]
   }
off:
   注销事件，从this.eventListener中删除事件处理程序。
fire: 
   function(eventType, evt, bubble)   bubble为true则表示触发事件并冒泡
   this._fireAndBubble(eventType, evt);  向父级冒泡this._fireAndBubble.call(this.parent, eventType, evt);
   this._fire(eventType, evt);  从this.eventListener中查找并触发事件处理程序，currentTarget 就是节点自己。

2、Node的初始化
   this._id = Konva.idCounter++;
   this.eventListeners = {};
   this.attrs = {};
   this._cache = {};
   this._filterUpToDate = false;
   this.setAttrs(config);  // 属性保存到attrs中
   另外还绑定了一些系统事件
3、Container的初始化
   this.children = new Konva.Collection(); //实例化集合对象
   Konva.Node.call(this, config); //直接调用基类的构造函数
   初始化过程如此简单，那就再深究一下吧，Container 到底有什么功能。
   add(child)
   destory()
   getChildren()
       get only circles
       var circles = layer.getChildren(function(node){
          return node.getClassName() === 'Circle';
       });
   hasChildren()
   removeChildren()
   destroyChildren()
   等等吧，主要集中了集合的增、删、找、克隆。剩下的就是与canvas绘画有关的api了。
4、Stage初始化
   this._buildDOM()
      构建content并append到container中 <div class="konvajs-content" role="presentation" style="position: relative;"></div>
      this.bufferCanvas = new Konva.SceneCanvas();  设置Canvas
      this.bufferHitCanvas = new Konva.HitCanvas({pixelRatio: 1});  设置Canvas
      _resizeDOM()  调整画布大小和画布容器大小
   _bindContentEvents()  
      在<div class="konvajs-content" role="presentation" style="position: relative;"></div>上绑定事件。

5、Canvas.js解析
   Konva.SceneCanvas
   Konva.HitCanvas
   都继承自Konva.Canvas，该类在初始化的时候创建了一个canvas，this._canvas，并设置了一些style。

   Konva.SceneCanvas
   Konva.HitCanvas 初始化的时候，设置this._canvas的宽高。设置对应的context,两个上下文如下：

   Konva.SceneContext(this)
   Konva.HitContext(this)  请详看Context.js解析

   其实这样的架构也很好理解。在原生Canvas中有一个Context对象。
   在Konvas中分别对原生的Canvas和Context对象进行了封装。

6、Context.js解析
   Konva.SceneContext-->>Konva.Context
   Konva.HitContext-->>Konva.Context

   Konva.Context 初始化时 设置this._context = canvas._canvas.getContext('2d');
        定义了一些与矩阵变化有关的接口(用来设置当前坐标系)
   		setTransform 调用原生setTransform方法
   		transform 调用原生的transform方法
   		translate 调用原生的translate方法
   		scale 调用原生的scale方法
   		rotate 调用原生的rotate方法
   		reset 设置单位矩阵,与原生的ResetTransform类似
   		其他api也大多是对原生api的封装

   Konva.SceneContext
        在基类的基础上提供了几个工具方法

   Konva.HitContext
        在基类的基础上提供了几个工具方法
7、Group.js解析
   定义Group中只能添加Group、Shape，其它的其实没做什么。
8、Layer.js解析
   Layer中只能添加Group、Shape
   Layer中包含自己的canvas元素,经验证一个层会生成一个canvas。
   Layer初始化时
      this.canvas = new Konva.SceneCanvas();
      this.hitCanvas = new Konva.HitCanvas({
          pixelRatio: 1
      });
9、一些全局对象意义
   Konva.shapes={ "key":"value" }  key为一个颜色值，value是一个Shape对象
10、BaseLayer.js解析
   初始化其实除了调用基类的构造函数，其它什么也没做。
   getCanvas() 返回Layer所属的SceneCanvas对象。
   getContext() 返回的是 Konva.SceneContext(this); 对象。每一个SceneCanvas对象，都有一个SceneContext上下文。
11、Shape.js解析
   初始化时，把自己保存在Konva.shapes中。
   getCanvas  调用BaseLayer.js中的getCanvas
   getContext 调用BaseLayer.js中的getContext
   destroy 销毁自己
12、Rect.js解析
   这个类继承自Shape，其实在这个类里的主要工作就是指定怎么画。即指定绘画函数 this.sceneFunc(this._sceneFunc);
13、Layer.js中draw处理流程
   调用Node.js中定义的方法
   draw: function() {
      this.drawScene(); 
      this.drawHit();
      return this;
   }

   Layer.drawScene 由子类 Layer.js 提供
      默认清空画布
      接着调用Container.drawScene 在这个方法中最重要的就是 this._drawChildren(canvas, 'drawScene', top, false, caching);
      Container._drawChildren 中就干了点什么呢?
         Clip相关的操作
         遍历孩子，调用孩子的drawScene方法，测试例子调用Shape.drawScene方法
         drawScene: function(can, top, caching, skipBuffer) {
            var layer = this.getLayer(),
                canvas = can || layer.getCanvas(),
                context = canvas.getContext(),
                cachedCanvas = this._cache.canvas,
                drawFunc = this.sceneFunc(),
                hasShadow = this.hasShadow(),
                hasStroke = this.hasStroke(),
                stage, bufferCanvas, bufferContext;

            context.save();
            // if buffer canvas is needed
            if (this._useBufferCanvas(caching) && !skipBuffer) {
            }
            // if buffer canvas is not needed
            else {
                context._applyLineJoin(this);
                // layer might be undefined if we are using cache before adding to layer
                if (!caching) {
                    if (layer) {
                        //由层来决定如何应用变化矩阵,this这里是一个Rect。具体可查看代码中注释。
                        layer._applyTransform(this, context, top);
                    }
                }

                drawFunc.call(this, context);//真正的绘画逻辑
                
            }
            context.restore();
            return this;
        }
15、Konva.Transform对矩阵操作的封装

阅读所得待总结：
继承方法
向后兼容提示
Konva.Collection
absolute就是相对于根吧