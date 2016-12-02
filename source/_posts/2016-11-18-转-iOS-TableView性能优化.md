title: '[转]iOS TableView性能优化'
date: 2016-11-18 15:39:17
tags: [iOS]
---

**tableview的优化一直是一个很考验基本功的活儿，之前做项目的适合被这个问题困扰了很久，通过性能工具、查阅文档解决，整理思路和解决方案如下：**

<!-- more -->

tableview优化最主要：复用cell，header，footer实例；使用约束布局cell子控件时不多次添加约束；图片不过大，尽量不使用透明视图；避免阻塞主线程；计算高度方法不做大量逻辑处理。

**cell是否使用了复用机制而不是每一次都创建新的cell。**

如果每次都创建新的cell，在滑动的时候会表现为：刚开始的时候很顺畅，但是会越来越卡，内存跟着一直升高，停止滑动的时候也不会降下来。使用缓存机制创建的cell，开始滑动的时候内存会开始上升，等创建了一个屏幕再加半屏的cell之后，内存趋于平稳。

**cell是否添加了大量的子控件，或者对layer做了过多的操作。**

如果添加了大量的子控件，使用drawRect方法添加子控件，平衡GPU与CPU的负担。同时还需要注意尽量使用不透明视图和不重叠的渐变，否则会加大GPU的负担，造成性能不佳。

**高度计算方法时不做复杂的计算，尽量只使用加减乘除。**

自适应高度的cell实现方式有很多种，比如，1.使用iOS7以上系统的                        

```swift
func tableView(tableView: UITableView, estimatedHeightForRowAtIndexPath indexPath: NSIndexPath) -> CGFloat
```

这个方法中，可以先给一个估计的高度，系统会从你给定的高度再去计算实际高度。但是在使用过程中会出现cell突然变高变得低的情况，不适用于高度变化太大的cell。
2.如果使用约束布局创建的cell子控件，子控件之间都建立了相互约束，最上面的子控件与cell顶部建立约束，最下面的子控件与cell底部建立了约束，相当于子控件把cell撑开了。

![img](http://upload-images.jianshu.io/upload_images/1423286-40cf751136c4f2ca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这时在高度计算方法中，走一遍cell的loaddata方法后可以通过

```swift
func systemLayoutSizeFittingSize(targetSize: CGSize) -> CGSize
```

取得cell的size，进而得到cell高度。通过这个方法获取的cell高度是十分精确的，只要创建好子控件的约束就能获得cell的size。比较不好的是这种方法会重走一遍cell的loaddata方法。除此之外在调用cell的loaddata之前需要得到cell的实例，实例创建的方式应该与cellForRow方法一样，优先从缓存池中取得。
这个方案可能会创建多个cell。如果能在内存汇总保存一份cell的实例就能解决这个问题了！我讲讲我实现的思路：首先先注册cell,当缓存池中没有cell时系统会自动创建，有的话会直接取缓存中的cell返回给你。

```swift
override func viewDidLodad() { 
      tableView.registerClass(CardCell.self, forCellReuseIdentifier: ID)
}
```

用lazy创建一个cell实例，由于lazy 关键字，cell的创建只会执行一次lazy 

```swift
var cell:CardCell = {
 //已经注册过cell，当缓存池中没有cell时系统会自动创建，有的话会直接取缓存中的cell返回
 let v = self.myTableView?.dequeueReusableCellWithIdentifier(self.ID) as! CardCell return v 
}()
```

通过懒加载的方式，只创建一次cell的实例，避免内存浪费。接下来要做的步骤就是之前讲的，调用cell的loadData方法，计算高度

```swift
func tableView(tableView: UITableView, heightForRowAtIndexPath indexPath: NSIndexPath) -> CGFloat { 
      self.imageCell.loadData(d) 
      let height:CGFloat = self.cell.contentView.systemLayoutSizeFittingSize(UILayoutFittingCompressedSize).heightreturn height
}
```

之前查资料的时候还有用空间换取时间的方案：
1）在请求网络数据成功后就计算好高度并通过字典或者数组保存高度值，在高度方法中直接根据数组下标或者key值取得高度并返回。
2）还有建立一个frameModel的方法，与1中相似，只是获得网络数据后保存到frameModel中，在frameModel中定义一个类方法，通过获得的model值计算高度后返回。

**避免快速滑动情况下开过多线程。**
cell中的图片开线程异步加载，相信每个人都会想到。线程开过多了会造成资源浪费，内存开销过大。图片过多时可以不要一滚动就走cellForRow方法，可以在scrollview的代理方法中做限制，当滚动开始减速的时候才加载显示在当前屏幕上的cell（通过tableview的dragging和declearating两个状态也能判断）

```swift
func tableView(tableView: UITableView, cellForRowAtIndexPath indexPath: NSIndexPath) -> UITableViewCell 
{ 
      var canLoad:Bool = !tableView.dragging && !tableView.decelearating 
      if canLoad { 
      //开始loaddata，异步加载图片 
}}
```

**图片处理**
1）后台下载图片后再回主线程刷新UI，避免阻塞主线程。
2）图片过大回造成GPU负担过大，可以在图片下载后压缩尺寸后显示
3）避免对layer做过多的操作，尽量设置图片为不透明

**补充：**
简单的设置cornerRadius是不会影响性能的，但是设置了maskToBounds，会导致离屏渲染，应减少设置图层 maskToBounds = YES ，；
使用懒加载图片的方式避免重复下载图片，浪费资源。图片下载后并做压缩处理后将其保存到缓存中，下次加载此图片之前先从缓存中取，如果取不到该图片就在后台下载保存。
使用Core Graphics实现圆角等功能。
**重写drawRect方法会离屏渲染，导致内存急剧上升，即使在这个方法里面不写一句代码，也会让内存升高。**

文／青葱烈马（简书作者）
原文链接：http://www.jianshu.com/p/76cb55c6329f
著作权归作者所有，转载请联系作者获得授权，并标注“简书作者”。