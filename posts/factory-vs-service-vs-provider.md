# factory、service、provider——依赖注入三剑客
## AngularJS中的依赖注入
AngularJS官方文档中对于依赖注入（Dependency Injection）以及AngularJS中的DI做出如下解释：
> Dependency Injection (DI) is a software design pattern that deals with how components get hold of their dependencies.
> 
> The Angular injector subsystem is in charge of creating components, resolving their dependencies, and providing them to other components as requested.

这是AngularJS进行分而治之的一种思路，即使用依赖注入来管理各个模块/组件，之间互相依赖同时不存在较强的耦合度。各个组件之间有不同的分工，其中，有一类组件就承担了“被注入”的角色，比如常量模块（constant）、服务（service）、提供者（provider）等。
`service`，`factory`和`provider`都是可以被注入的组件，它们有些地方十分相似，在很多场景里甚至都可以使用，那么又该如何区分具体的使用场景呢？

## factory
### 语法

```
module.factory('yourFactory',function );
```
### 结果
`factory`可以被注入到`controller`，`filter`，`factory`等多种组件中，它被注入之后获取到的值是传递到初始化方法中的函数的执行结果（有点拗口......）。

```
// 初始化factory
module.factory('foo',function(){
	return 'Hello';
});
function Controller(foo) {
  expect(foo).toEqual('Hello'); // 注入的结果就是 FunctionYouPassedToFactory()
}
```
再增加一点OO呢？

```
function Dog(){
	this.bark = function(word){
		return 'Dog say:' + word;
	};
}

// 初始化factory并返回一个Dog的示例
module.factory('dogFactory',function(){
	return new Dog();
});
function Controller(dogFactory) {
  expect(dogFactory instanceof Dog).toBe(true); 
  ecpect(dogFactory.bark('wang')).toEqual('Day say:wang');
}
```
明显可以看出注入的结果其实就是一个函数的执行结果，那么如果我们初始化`factory`的时候没有`return`任何结果呢？是不是注入的结果就是`undefined`呢？

```
module.factory('foo',function(){
	doSomeThingButReturnNothing();
});
function Controller(foo) {
  expect(foo).toBeUndefined();  // ????? 
}
```
答案是否定的。这样会出现错误
> $injector:undef 
> 
> Undefined Value:Provider 'foo' must return a value from $get factory method.

也就是说factory必须返回一个非undefined的值  

## Service
### 语法

```
module.service('yourSercice',function );
```
### 结果
于factory相同是service也会初始化出一个可以被注入的对象，而且这个对象是一个真正的对象，它返回的结果是把初始化函数作为构造函数实例化的对象。

```
// 初始化service
module.service('foo',function(){
	this.name = 'foo';
});
function Controller(foo) {
  expect(foo.name).toEqual('foo'); // 注入的结果是 new FunctionYouPassedToService()
}
```
同样，如果以上述`factory`中的`Dog`为例，使用`service`的话就没必要生成一个函数再`return`一个实例对象了。

```
module.service('foo',Dog);
function Controller(foo) {
  expect(foo instanceof Dog).toBe(true); 
  ecpect(foo.bark('wang')).toEqual('Day say:wang');
}
```

## provider
### 语法

```
module.provider('yourSercice',function(){
	this.$get = function(){
		...
	}
});
```
### 结果
`provider`的初始化函数中必须有`this.$get`且可以被AngularJS初始化，即一个方法或者显式注入列表+方法。注入`provider`的结果即是`$get`方法执行的结果。

```
// 初始化provider
module.provider('foo',function(){
	this.name = 'foo';
	this.$get = function(){
		return this.name;
	};
});
function Controller(foo) {
  expect(foo).toEqual('foo'); // 注入的结果是(new FunctionYouPassedToProvider()).$get()
}
```
与`factory`和`service`不同的是，`provider`可以在`module.config()`方法中进行注入，并进行配置，这也是`provider`和`factory`、`service`最大的不同点。

```
module.config(function(fooProvider){   
	expect(fooProvider.name).toEqual('foo');
});
```
在`config`函数中注入`provider`的话必须对初始化的`provider`名字添加“Provider”后缀，同时，这里注入的`provider`和在`controller`中注入的有所不同，与service相同，这里获得的结果是`new FunctionYouPassedToProvider()`。

## 总结
“三剑客”的基本用法都介绍完了，其实这三者在AngularJS中统一被称为`Provider`，同样的还是有`value`和`constant`，现在先不纠结这个问题。  
那么回到我们一开始提出的问题，三者该如何选择呢？原则如下：

1. 如果不仅需要在`controller`，`directive`中注入，同时需要在全局进行配置的话，选用`provider`
2. 如果仅仅希望注入的结果像是调用了一个函数，选用`factory`，而且`factory`可以返回原始数据类型。
3. 如果希望注入一个`new`出来的对象的话，选用`service`。

其中`factory`和`service`是比较难选择的，因为这俩货实在是太像了，大部分场景中二者都可以满足需求。上面的2，3条大概说明了如何选择，可是如果你还是不知道选什么的话，最好选择`factory`，它会让你用起来更舒服一些。AngularJS官方对此也有一个总结性的对比，如下表：

|  特性/类型      |         Factory  | Service| Provider|
| -------------            |:-----:|:------:|:-------:|
|  是否可以被注入            | 是     |   是   |是       |
|  使用依赖注入友好          | 否      |是      |否       |
|  对象是否可以在config中配置 | 否      |否     |是       |
|  是否可以创建方法          | 是      |是     |是       |
|  是否可以创建原始类型     | 是      |否      |是       |
[文档](https://docs.angularjs.org/guide/providers#conclusion)中有这样一句话，说明了`factory`和`service`最大的不同点。
> Factory and Service are the most commonly used recipes. The only difference between them is that the Service recipe works better for objects of a custom type, while the Factory can produce JavaScript primitives and functions.

