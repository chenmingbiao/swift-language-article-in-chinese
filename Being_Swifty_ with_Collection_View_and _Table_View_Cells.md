title: "初识 iOS 9 中新的联系人框架"
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

在 `Objective-C` 中，最典型就是使用 `NSArray` 来记录 `collection view` 的数据

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
shudder

Okay so that’s not the most type-safe approach, although it’s surprisingly common in Obj-C iOS code. As a better alternative in Swift, we can use enum cases for the different item types, and perform lookups of the items themeselves as needed. Let’s look at an example.

###Example

In an entertainment app I’m working on I’ve got a few types of cells for the types of news items that come in:

```swift
enum NewsItem {
  case Trailer(index: Int)
  case Review(index: Int)
  case Playthrough(index: Int)
}
```

The index is just a way to keep track of which item this it supposed to represent in the database. We take this approach to keep the amount of data needed to produce the collection view down. We don’t need all the associated data with every video to be present when putting together a collection view, we just need the info on what cell is tapped, after it is tapped.

Let’s say we have a simple collection view that shows one of these three and picks a custom cell for each. In a Swift file `NewsFeed.swift` I have that acts as the dataSource of my collection view for the main news view. Of particular interest is the cellForItemAtIndexPath method, which runs the NewsItem record through a switch and produces the correct type of cell, with the relevant information populated:

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

```swift
enum NewsItem {
  case Trailer(index: Int)
  case Review(index: Int)
  case Playthrough(index: Int)
}
```

The associated index value is so that we can find the individual item in the DB when the collection view wants to display a cell.

Something about this code didn’t sit right with me though. I felt that too much of the code was boilerplate; in particular the switch felt bulky and like it had too much work being done inside each case.

But what if I created a protocol for any data source that could be presented as a collection view cell? It would change on a view-by-view basis so I don’t actually want this in my model.. but I do like having it on these particular CollectionViewCell subclasses.

So, I created a protocol called NewsCellPresentable, which I can adhere to in extensions with my custom collection view cells:

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

###P.S.

If you want to try this out yourself and don’t have the same DB layer as me… guess what? Neither do I! You can easily stub this out like this:

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