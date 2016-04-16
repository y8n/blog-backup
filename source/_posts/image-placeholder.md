title: 用canvas实现占位符图片
date: 2015-12-27 20:11:49
tags:
- 图片占位符
categories:
- HTML
thumbnail: /gallery/image-placeholder/thumbnail.png
banner: /gallery/image-placeholder/banner.png
---
前几天在一个segmentfault上看到一个文章，介绍的是作者用不到1KB的js代码实现的占位符图片，[传送门](http://segmentfault.com/a/1190000004161123)，看到之后觉得蛮有意思的，觉得这个轮子可以造一造。  
<!-- more -->
之前接触过一个项目，也有图片占位符的需求，当时的做法是当图片加载失败，也就是触发onerror事件的时候，替换图片的src为一张占位符图片，这样的做法简单易行，只是缺乏一定的可配置性，如果一个页面需要多种占位符图片的话，那就得有多个这样的图供替换，如果不按照这样传统的做法，而是用js进行可配置化的生成占位符图片又该怎么做呢？  

## 思路分析
有了目的，那从哪入手呢？首先有几个问题需要捋一捋。
- 如何生成占位符图片（的地址）
- 既然解决的是配置问题，有哪些可配置选项

以上问题依次分析一下。首先，img标签是需要src地址的，那么如果图片加载失败，就要替换地址，替换的地址哪来呢？又控制不了服务器...那就换一个思路，还有什么方式可以显示图片？答案就是[base64编码](https://zh.wikipedia.org/zh/Base64)。base64的概念以及用处说实话我了解不多，目前知道可以作为img标签的src属性值，也可以显示图片。那这就有点思路了，操作不了服务器，生成编码js还是有希望做到的。那么问题来了（挖掘机技术哪家强？），base64编码又该怎么生成，我可对生成编码的算法过敏。问了下Google，发现canvas对象有一个方法：toDataURL，可以将绘制好的canvas转化为base64编码。Bingo！这样就可以事先用js画好canvas，再调用toDataURL方法，不就有base64编码了嘛。    
第二个问题：哪些是可配置的呢？首先想一下一个占位符图片有哪些要素，文字得有，图标就视情况而定了，颜色也是，宽高什么的都是应该的。  

## 最终效果
![][1]
![][2]

## 干货代码
代码地址:[image-placeholder](https://github.com/y8n/image-placeholder)

#### 使用方法
首先引入脚本

``` html
<script src="path/to/image-placeholder/image-placeholder(.min).js"></script>
```
可以有多种方式进行调用

``` javascript
// 直接传递图片DOM对象
var img = document.querySelector('#image');
placeholder(img);

// 传递选择器
placeholder('#image');

// 添加配置项
placeholder('#image',{
    icon:true,
    text:'Image loading failure'
})

// 或者可以获取canvas对象用于其他用途
var canvas = placeholder.getCanvas({
    height:120,
    width:200
});
// 或者获取base64编码
var base64Data = placeholder.getBase64Data({
    height:120,
    width:200,
    icon:true
});
```
可配置项包括

```
var options = {
    width:300,		//生成图片的宽度
    height:200,		//生成图片的高度
    icon:true,   		//是否显示图标，默认不显示
    iconWidth:30,		//显示图标的话，设置图标的宽度
    iconHeight:20	,	//显示图片的话，设置图片的高度
    text:'图片失效',		//显示文字，默认是 'image is unavailable'
    fontSize:16,		//文字大小
    bgColor:'#eee',		//背景颜色
    textColor:'#000'	//文字和图标颜色
};
```

## 小亮点
虽然是重复造的轮子，但是中间很多地方还是下了功夫的。  
比如图标，是用canvas画出来的，虽然没有太大的技术含量，懂得[贝塞尔曲线](https://zh.wikipedia.org/wiki/%E8%B2%9D%E8%8C%B2%E6%9B%B2%E7%B7%9A)的都能画出来，我之前对于canvas画图了解的很浅，也没实际划过稍微复杂一点的图，这次边查资料边画，最终画出来一个图标，还是有点小激动的，高手轻喷~  
另一个地方就是性能优化方面的一个处理。一般占位符图片用得比较多的地方是有大量图片的页面，比如电商类的商品列表图片，或者其他图片列表，这种情况下经常会因为网络问题出现图片加载失败的情况，所以需要在图片加载失败的回调函数onerror里把图片替换为占位符图片。  
传统的方法，也就是用一个固定的图片作为占位符图片的好处之一就是即使有大量图片加载失败，占位符图片也只需要加载一次就行了，而不用每次都加载。js利用canvas生成base64编码作为占位符图片也是一样的道理，没有必要多次生成，同样配置项且同样大小的图片，只需要生成一次就够了。

``` javascript
image.onerror = function () {
    /* ... */
    var base64Data,key;
    key = height + ':' + width;   //根据图片的宽和高拼成key
    
    //如果缓存里有这样尺寸的占位符图片，直接获取缓存里的；如果没有的话，重新生成并放到缓存里
    if(key in cache){   
        base64Data = cache[key];
    }else{
        base64Data = placeholder.getBase64Data(options);
        cache[key] = base64Data;
    }
    image.src = base64Data;
};
```

周末的时候花了点时间造了这么一个小轮子，同步更新了第二篇博文，也蛮有意思的，过程中又发现了一些问题
- 贝塞尔曲线怎么能画着简单点呢
- canvas太菜了，接下来可以考虑再造一些相关的轮子

**最后一句!**欢迎提issue、pr，点个star也是极好的~😃


  [1]: /gallery/image-placeholder/default-image.png
  [2]: /gallery/image-placeholder/options-image.png
