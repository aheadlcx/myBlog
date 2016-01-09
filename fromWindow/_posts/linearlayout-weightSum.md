title: about LinearLayout weightSum
date: 2014-12-09 23:14:30
tags:
---


#**关于线性布局的weight和weightSum属性**


	一般我们用比例自适配，都会用到线性布局的weight属性，这个属性可以理解为优先级，可以设置width属性为0dp，然后子View A和子View B
	，就可以根据各自的weight属性来自适配了。但是有时候，我们需要某个线性布局仅仅有一个子view，而且需要为屏幕的宽带的一半或者三分之一，这个时候，
	就需要给父View一个weightSum属性了，这个属性表示子view的weight属性总和多少，如果父view的weightSum为1，唯一的子view的weight属性为0.5或者0.33，则表示
	子view的宽度为屏幕的一半或者三分之一。


	


新浪博客地址: [我的新浪博客](http://weibo.com/u/1709380933/home?wvr=5)
