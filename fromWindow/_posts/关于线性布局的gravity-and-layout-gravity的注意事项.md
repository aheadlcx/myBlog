title: 关于线性布局的divider,gravity and layout_gravity的注意事项
date: 2014-12-25 18:28:25
tags:
---
#线性布局的gravity和layout_gravity属性，发现了一个“bug”
api level 11之后，线性布局我们可以用divider属性，这个属性可以帮我们处理一些很好的UI效果，例如水平的线性布局有2个子view，Button A，Button B。
他们之间需要有间隔，我们可以用marginRight来处理，但是如果我们因为也许需要，在运行中需要隐藏B，这个时候A右边就会出现一些间隔了，这是我们不期望的。这个时候，我们用divider和drawable来处理，系统就能自动帮我们处理好了。先上个正确的代码，大家看看。

	 <LinearLayout
	            android:layout_width="fill_parent"
	            android:minHeight="50dp"
	            android:orientation="horizontal"
	            android:divider="@drawable/linearlayout_divider"
	            android:showDividers="middle"
	            android:layout_height="wrap_content">
	            <Button
	                android:id="@+id/tv_distribute"
	                android:text="派送"
	                style="@style/style_text_white_f4"
	                android:gravity="center"
	                android:layout_weight="1"
	                android:background="@color/blue"
	                android:layout_width="0dp"
	                android:layout_height="wrap_content"/>
	            <Button
	                android:text="派送清单"
	                android:gravity="center"
	                android:layout_weight="1"
	                style="@style/style_text_white_f4"
	                android:background="@color/blue"
	                android:layout_width="0dp"
	                android:id="@+id/tv_distribute_send"
	                android:layout_height="wrap_content"/>
	            

	            </LinearLayout>
正确的效果图如下
![正确的效果图](https://raw.githubusercontent.com/aheadlcx/learngit/master/good.png)

线性布局的divider的drawable代码如下
	<?xml version="1.0" encoding="utf-8"?>
	<shape xmlns:android="http://schemas.android.com/apk/res/android" 
	    android:shape="rectangle"
	    >
	    
	    <size
	        android:width="15dp"
	        android:height="15dp"
	         />
	    <solid 
	        android:color="@android:color/transparent"
	        />

	</shape>


我们都知道如果是线性布局设置为水平的，gravity属性就不应该设置为android:gravity="center_horizontal"，可是万一设置了呢。
我们可以在上面代码中直接加入并且看看结果，我们会发现，第一个button左边有点空白，而最后一个button右边会显示不齐全（如果你把button背景设置为圆角，会很容易观察到）。
加上了这个属性，效果图如下
![错误的效果图](https://raw.githubusercontent.com/aheadlcx/learngit/master/bad.png)

这个初步猜测为，水平的线性布局给子布局带了一个marginLeft或者paddingLeft之类的。

我们来跟踪下代码看看，跟到linearlayout的代码里面，我们第一时间，肯定去找构造函数看看。我们可以看到
	index = a.getInt(com.android.internal.R.styleable.LinearLayout_gravity, -1);
	        if (index >= 0) {
	            setGravity(index);
	        }

那我们再跟到setGravity这个方法中，看看。
	 public void setGravity(int gravity) {
	        if (mGravity != gravity) {
	            if ((gravity & Gravity.RELATIVE_HORIZONTAL_GRAVITY_MASK) == 0) {
	                gravity |= Gravity.START;
	            }

	            if ((gravity & Gravity.VERTICAL_GRAVITY_MASK) == 0) {
	                gravity |= Gravity.TOP;
	            }

	            mGravity = gravity;
	            requestLayout();
	        }
	    }

注意看到这行代码  gravity |= Gravity.START;还有 gravity |= Gravity.TOP;，看start和top，貌似隐若感觉到什么了。既然是布局方面的问题，我们就跟到Linearlayout的
	   protected void onLayout(boolean changed, int l, int t, int r, int b) {
	        if (mOrientation == VERTICAL) {
	            layoutVertical();
	        } else {
	            layoutHorizontal();
	        }
	    }

接着，我们再看layoutHorizontal方法去，可以看到如下代码
	   case Gravity.CENTER_HORIZONTAL:
	                // mTotalLength contains the padding already
	                childLeft = mPaddingLeft + (mRight - mLeft - mTotalLength) / 2;
	                break;

这个mPaddingLeft，是不是让我们前面的猜测有点依据了~~别高兴太早，这个mPaddingLeft到底是在什么时候赋值的呢。。点击一下，原来是view里面的变量。。。。view里面接近2W行的代码。。。



