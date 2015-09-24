title: Object转发Protocol
date: 2014-09-23
type:
tags: [UITableView]
---

在封装UITableViewDataSource和UITableViewDelegate遇到如下的问题.

##### HDTableViewManager.h

``` objc
/**
 * HDTableViewManager去实现UITableView的相关协议,简化了UITableView的使用,
 * 不用重复的实现相关协议，减轻ViewController的负担.
 */
@interface HDTableViewManager : NSObject <UITableViewDataSource, UITableViewDelegate>

/**
 * 省略了其他相关代码
 */


//HDTableViewManager的dataSource和delegate用于转发协议到相关的使用类中,方便一些个性化的需求.
/**
 *  <UITableViewDataSource>,默认为nil
 */
@property (nonatomic, weak, readwrite) id<UITableViewDataSource> dataSource;

/**
 *  <UITableViewDelegate>,默认为nil
 */
@property (nonatomic, weak, readwrite) id<UITableViewDelegate> delegate;

@end
```

##### HDTableViewManager.m

``` objc
@implementation HDTableViewManager

/**
 * 省略了其他相关代码
 */

- (UITableViewCell *)tableView:(UITableView *)tableView
         cellForRowAtIndexPath:(NSIndexPath *)indexPath
{
    UITableViewCell *cell = nil;
    if ([self.dataSource respondsToSelector:@selector(tableView:cellForRowAtIndexPath:)])
    {
        cell = [self.dataSource tableView:tableView cellForRowAtIndexPath:indexPath];
    }
    // check cell is load;(when datasource is uitableviewcontroller)
    if (!cell)
    {
    	cell = [[[self class] alloc] initWithStyle:style reuseIdentifier:identifier];
    }
    return cell;
}

@end
```
[HDTableViewManager](https://github.com/Haidora/HaidoraTableViewManager)内部实现UITableView的相关协议,判断如果[HDTableViewManager](https://github.com/Haidora/HaidoraTableViewManager)的dataSource和delegate如果实现相关协议就返回[HDTableViewManager](https://github.com/Haidora/HaidoraTableViewManager)的dataSource和delegate.这样一开始是没问题的,但是久而久之随着苹果公司SDK的更新,UITableView的相关协议可能会添加,会删除.[HDTableViewManager](https://github.com/Haidora/HaidoraTableViewManager)就需要对相关协议做适配.如果有些协议是无关痛痒的，还要去适配,既不优雅,又累死人.
本着懒人的原则,能不能动态的转发协议.如果HDTableViewManager内部实现了就调用[HDTableViewManager](https://github.com/Haidora/HaidoraTableViewManager)的实现,如果[HDTableViewManager](https://github.com/Haidora/HaidoraTableViewManager)没有实现,就看dataSource和delegate实现没有,实现了就调用,没有实现就算了.

##### 函数调用流程
![图片来源网络](/resource/images/forwarding_protocol/1.png)

###### 方法1
通过重写resolveInstanceMethod固定返回NO,然后通过forwardingTargetForSelector来转发SEL,结果啥反应都没有,不知道是我弄错了还是理解错误.

```objc
+ (BOOL)resolveInstanceMethod:(SEL)sel
{
    return NO;
}

- (id)forwardingTargetForSelector:(SEL)aSelector
{
//在这里转发SEL
}
```

###### 方法2
最后通过万能的Google找到了[Colin Eberhardt](https://github.com/ColinEberhardt)大神的[文章](http://blog.scottlogic.com/2012/11/19/a-multicast-delegate-pattern-for-ios-controls.html),大神在文章中给出了UIScrollViewDelegate的转发,代码如下.

```objc
/**
 * 如果self,dataSource,delegate有实现就返回YES.
 */
- (BOOL)respondsToSelector:(SEL)aSelector
{
    if ([super respondsToSelector:aSelector])
    {
        return YES;
    }
    else if ([self.dataSource respondsToSelector:aSelector])
    {
        return YES;
    }
    else if ([self.delegate respondsToSelector:aSelector])
    {
        return YES;
    }
    else
    {
        return NO;
    }
}

/**
 * 如果respondsToSelector针对某一个SEL返回是YES,刚刚self没有实现该方法,
 * 就通过forwardingTargetForSelector转发SEL
 */
- (id)forwardingTargetForSelector:(SEL)aSelector
{
    if ([self.dataSource respondsToSelector:aSelector])
    {
        return self.dataSource;
    }
    else if ([self.delegate respondsToSelector:aSelector])
    {
        return self.delegate;
    }
    else
    {
        // never call
        return nil;
    }
}
```

##### 总结

到此就实现了UITableView相关协议的转发,如果以后苹果升级UITableView的协议,也可以动态的去适配,不用苦逼的去写重复的代码.
如果发现文中有啥问题,望指正.

##### 参考链接    
* [继承自NSObject的不常用又很有用的函数（2）](http://www.cnblogs.com/biosli/p/NSObject_inherit_2.html)
* [A MULTICAST DELEGATE PATTERN FOR IOS CONTROLS](http://blog.scottlogic.com/2012/11/19/a-multicast-delegate-pattern-for-ios-controls.html)