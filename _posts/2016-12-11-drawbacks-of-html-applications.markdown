---
layout: post
title:  "HTML应用的缺陷有哪些？"
date:   2016-12-11 22:44:01 +0800
categories: jekyll update
---
首先声明，我不是一个前端开发，虽然我自己对前端技术很感兴趣，但是我自己也只是懂一点点皮毛，大部分问题还是靠Google，SO和MDN来解决。另外，我自己对框架完全不了解，最近在看angularJS。这里仅说说我在saralin开发中遇到的的自己觉得不太爽的一些例子。
## 滚动加载更多的问题
Saralin的帖子详情页面是通过webview加载一个静态网页来展示的。在saralin早先版本中，网页加载完毕之后，页面就不再与webview控制器（webview controller）交互了。在1.8版本中，我加入了动态加载更多内容这一功能。1.8.1加入了向下加载更多，1.8.2加入了向上（即反向）加载更旧的内容，问题主要出在后者。
为了保持在插入新的dom节点之后页面停留在原位置不动，我们需要修改页面的scroll offset，类似下面这样：

```javascript
var oldHeight = document.client.clientHeight
var oldScrollOffset = document.scrollTop
//insert new dom
var newHeight = document.client.clientHeight
var newScrollOffset = document.scrollTop
//set offset
document.scrollTop = newScrollOffset + newHeight – oldHeight
```


问题在于有时候这样做会失效，导致页面直接滚动到了最顶端。我怀疑是因为我自己手指仍停留在网页上，所以滚动并没有终止，iOS系统为了保持滚动的连贯性，会自动设置网页scrollView的scrollOffset，而这一操作在我设置scrollTop之后，从而导致我的设置被覆盖了。
为了解决这个问题，我现在将插入dom节点的时机选在松手之后（即页面的touchend事件），当然如果当时已经没有手指放在屏幕上，那就直接插入dom节点，然后设置scrollTop属性。
## 数据刷新的问题
我一直想把帖子详情页面做成一个类似于原生的UITableView一样的组件。但是在开发过程中,我发现了一些功能上的不足。
首先是缺少一种数据刷新的回调。UITableView可以通过各种delegate方法来获知cell重新被展示，但是web却只有一个onload事件。当我滚动到某一个element显示的时候，我无法对这个element变更外观。例如，一个展示发帖时间的span，我总不可能在页面初始化设置好了之后就不再改变这个值了吧？除非显示绝对时间，但是那种时间格式太难看了（“2016-11-23 12:45”，我觉得我宁愿不要看到这样子的时间）。一般都是显示模糊时间，例如“3分钟前”。我很奇怪为什么HTML没有一种可以定时收到回调的element,那样的话就可以完美解决上述问题。我现在的做法是定时遍历所有时间span，然后逐个更新。

## onscroll回调事件
onscroll事件在UIWebview和WKWebview中行为不一样，在UIWebview中，onscroll回调事件在滚动的中途并不会被调用，而WKWebview不存在这一问题。参见：[Affix doesn't update in iOS UIWebview](https://github.com/twbs/bootstrap/issues/16202)
