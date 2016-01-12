title: "collectionView 和 tableView cells 的快捷方式"
date: 2016-01-05 16:11:00
tags: [AppCoda]
categories: []
permalink: 

---
原文链接=http://jamesonquave.com/blog/being-swifty-with-collection-view-and-table-view-cells/
作者=Jameson Quave
原文日期=2015/12/28
译者=CMB
校对=
定稿=
发布时间=

Here’s a common scenario: You have a table view or collection view that has a variety of different types of content. You want to display varying cells based on these types of content, and they’re all mixed within a single section. Pardon the stand-in art, but it looks roughly like this:

这是一个常见的场景：你有一个 `table view` 或者 `collection view` 里面含有大量不同种类的内容。你想做到基于不同种类的内容而展示不一样的 `cell` ，这些 `cell` 都是继承于同一个单元（原谅我站在艺术性的角度去设计），它看起来就如下图所示：

![](http://i4.tietuku.com/53092553e2ff9f43.png) 

In the Objective-C world, it was typical to just use an NSArray to hold whatever records your collection view was going to be using as a data source, and then for each element check what class it is before picking a cell. This case seems particularly not Swifty these days.

在 `Objective-C` 中，最典型就是使用 `NSArray` 来记录 `collection view` 的数据源，然后通过对比每个数据源的类型后再对 `cell` 进行操作，现在看来这种方式是特别不方便的。

```Objective-c
- (UICollectionViewCell *)collectionView:(UICollectionView *)collectionView cellForItemAtIndexPath:(NSIndexPath *)indexPath {

    UICollectionViewCell *cell = [collectionView dequeueReusableCellWithReuseIdentifier:@"identifier" forIndexPath:indexPath];
 
    id record = self.records[indexPath.row];
 
    if([record isKindOfClass:[PlaythroughItem class]]) {
        // ...
    }
    else if([record isKindOfClass:[ReviewItem class]]) {
        // ...
    }
    else if([record isKindOfClass:[TrailerItem class]]) {
        // ...
    }
 
    return cell;
}
```

Okay so that’s not the most type-safe approach, although it’s surprisingly common in Obj-C iOS code. As a better alternative in Swift, we can use enum cases for the different item types, and perform lookups of the items themeselves as needed. Let’s look at an example.

这不是最类型安全的方法，虽然这种方式在普通的 `Obj-c` 代码中令人惊讶。在 `Swift` 中，有更加好的方式去替换上述问题，那就是使用枚举来为每个项赋予类型，然后进行查找自己所需要的项。让我们看看以下的例子。

###Example
###例子

In an entertainment app I’m working on I’ve got a few types of cells for the types of news items that come in:

在一个我正在写的娱乐App里，我有一些不同新闻类型的 `cell` :

```swift
enum NewsItem {
  case Trailer(index: Int)
  case Review(index: Int)
  case Playthrough(index: Int)
}
```

The index is just a way to keep track of which item this it supposed to represent in the database. We take this approach to keep the amount of data needed to produce the collection view down. We don’t need all the associated data with every video to be present when putting together a collection view, we just need the info on what cell is tapped, after it is tapped.

该索引只是一个方法来跟踪它应该在数据库中表示的项目。我们采取这种方法，以保持所需的数据的量产生的收集视图。我们不需要所有的相关联的数据，每一个视频是在一起时，收集意见，我们只需要的信息是什么细胞被挖掘，在它被窃听。

Let’s say we have a simple collection view that shows one of these three and picks a custom cell for each. In a Swift file `NewsFeed.swift` I have that acts as the dataSource of my collection view for the main news view. Of particular interest is the cellForItemAtIndexPath method, which runs the NewsItem record through a switch and produces the correct type of cell, with the relevant information populated:

让我们说，我们有一个简单的集合视图，显示了这三个，并为每个选择一个自定义单元格。在名为`NewsFeed.swift` 的 `Swift` 文件中作为主要的新闻观我的集合视图的数据源。特别感兴趣的是 `cellforitematindexpath` 方法，它运行的 `newsitem` 记录通过开关和产生细胞正确的类型，与相关信息密集：

```swift
func collectionView(collectionView: UICollectionView, cellForItemAtIndexPath indexPath: NSIndexPath) -> UICollectionViewCell {
    let record = records[indexPath.row]
 
    switch(record) {
 
    case .Playthrough(let index):
        let cell = collectionView.dequeueReusableCellWithReuseIdentifier("PlaythroughCell", forIndexPath: indexPath) as! PlaythroughCollectionViewCell
        let playthrough = MediaDB.playthroughAtIndex(index)
        cell.titleLabel.text = playthrough.title
        cell.lengthLabel.text = playthrough.length.prettyTime
        return cell
 
    case .Review(let index):
        let cell = collectionView.dequeueReusableCellWithReuseIdentifier("ReviewCell", forIndexPath: indexPath) as! ReviewCollectionViewCell
        let review = MediaDB.reviewAtIndex(index)
        cell.ratingLabel.text = "\(review.rating) out of 10"
        cell.titleLabel.text = review.title
        return cell
 
    case .Trailer(let index):
        let cell = collectionView.dequeueReusableCellWithReuseIdentifier("TrailerCell", forIndexPath: indexPath) as! TrailerCollectionViewCell
        let trailer = MediaDB.trailerAtIndex(index)
        cell.titleLabel.text = trailer.title
        cell.lengthLabel.text = trailer.length.prettyTime
        return cell
    }
}
```

This code works well enough, record is of type `NewsItem` which can be one of three cases for the different news items we support:

这个代码的好作品，记录类型是 `newsitem` 可以为不同的新闻我们支持三例一：

```swift
enum NewsItem {
  case Trailer(index: Int)
  case Review(index: Int)
  case Playthrough(index: Int)
}
```

The associated index value is so that we can find the individual item in the DB when the collection view wants to display a cell.

相关的索引值是这样，当收集视图要显示单元格时，我们可以在数据库中找到单独的项。

Something about this code didn’t sit right with me though. I felt that too much of the code was boilerplate; in particular the switch felt bulky and like it had too much work being done inside each case.

关于这个代码的东西没有坐在我的右边。我觉得太多的代码模板；特别是开关感觉笨重，好像在每一种情况下所做的工作太多了。

But what if I created a protocol for any data source that could be presented as a collection view cell? It would change on a view-by-view basis so I don’t actually want this in my model.. but I do like having it on these particular CollectionViewCell subclasses.

但是，如果我创建了一个协议的任何数据源，可以作为一个集合视图单元？它会改变一个视图的基础上，所以我不希望这在我的模型中。但我确实喜欢在这些特殊的 `collectionviewcell` 子类。

So, I created a protocol called NewsCellPresentable, which I can adhere to in extensions with my custom collection view cells:

所以，我创建了一个叫做 `newscellpresentable` 协议，我可以坚持我的自定义集合视图单元格扩展：

```swift
protocol NewsCellPresentable {
    func configureForIndex(index: Int)
}
 
extension PlaythroughCollectionViewCell: NewsCellPresentable {
    func configureForIndex(index: Int) {
        let playthrough = MediaDB.playthroughAtIndex(index)
        self.titleLabel.text = playthrough.title
        self.lengthLabel.text = playthrough.length.prettyTime
    }
}
extension ReviewCollectionViewCell: NewsCellPresentable {
    func configureForIndex(index: Int) {
        let review = MediaDB.reviewAtIndex(index)
        self.titleLabel.text = review.title
        self.ratingLabel.text = "\(review.rating) out of 10"
    }
}
extension TrailerCollectionViewCell: NewsCellPresentable {
    func configureForIndex(index: Int) {
        let trailer = MediaDB.trailerAtIndex(index)
        self.titleLabel.text = trailer.title
        self.lengthLabel.text = trailer.length.prettyTime
    }
}
```

This feels much cleaner already. Now I can go back to my cellForItemAtIndexPath method and trim it down to just the following:

这感觉已经很干净了。现在我可以回到我的 `cellforitematindexpath` 方法和修剪下来的下面：

```swift
func collectionView(collectionView: UICollectionView, cellForItemAtIndexPath indexPath: NSIndexPath) -> UICollectionViewCell {
    let record = records[indexPath.row]
 
    var cell: NewsCellPresentable
    switch(record) {
 
    case .Playthrough(let index):
        cell = collectionView.dequeueReusableCellWithReuseIdentifier("PlaythroughCell", forIndexPath: indexPath) as! PlaythroughCollectionViewCell
        cell.configureForIndex(index)
 
    case .Review(let index):
        cell = collectionView.dequeueReusableCellWithReuseIdentifier("ReviewCell", forIndexPath: indexPath) as! ReviewCollectionViewCell
        cell.configureForIndex(index)
 
    case .Trailer(let index):
        cell = collectionView.dequeueReusableCellWithReuseIdentifier("TrailerCell", forIndexPath: indexPath) as! TrailerCollectionViewCell
        cell.configureForIndex(index)
    }
 
    return (cell as! MediaCollectionViewCell)
}
```

What do you think? Is this a cleaner approach? Let me know if you have a different method in the comment, or `let me know` on `Twitter`. My username is `@jquave`.

你觉得怎么样？这是一个清洁的方法吗？让我知道，如果你有一个不同的方法在评论，或“让我知道”在“推特”。我的用户名是 `@ jquave` 。

###P.S.

If you want to try this out yourself and don’t have the same DB layer as me… guess what? Neither do I! You can easily stub this out like this:

如果你想尝试一下自己，不要和我一样有相同的分贝……你猜怎么着？我也不！你可以很容易地这样的存根：

```swift
class MediaDB {
    class func titleForRecord(index: Int) -> String {
        return "Title!!"
    }
    class func trailerAtIndex(index: Int) -> Trailer {
        return Trailer()
    }
    class func reviewAtIndex(index: Int) -> Review {
        return Review()
    }
    class func playthroughAtIndex(index: Int) -> Playthrough {
        return Playthrough()
    }
}
 
struct Trailer {
    let title = "Trailer Title"
    let length = 190
}
 
struct Review {
    let title = "Review Title"
    let rating = 4
}
 
struct Playthrough {
    let title = "Playthrough Title"
    let length = 9365
}
 
 
enum NewsItem {
    case Trailer(index: Int)
    case Review(index: Int)
    case Playthrough(index: Int)
}
```

Personally I always stub things out with static values before I do the work of integrating with a backend service or whatever data provider is needed. This makes it much easier to iterate.

就个人而言，我总是在做与后端服务集成的工作，或者需要任何数据提供程序的工作之前，总会在静态的值上短的东西。这使得它更容易进行。