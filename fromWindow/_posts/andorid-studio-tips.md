title: andorid studio tips
date: 2014-12-15 20:07:56
tags:
---

#android studio 小技巧

##重构篇

### refactor
invert boolean的意思是，如下面代码,isGird = true;如果我在下面的oncreate方法中，选中isGird refactor和invert boolean，可以把isGird改成isBoy，并且原始的默认值改为false。
#java
	public class MainActivity extends Activity {
	    private boolean isGird = true;

	//    public SwipeRefreshLayout mSwipeRefreshLayout = null;
	    @Override
	    protected void onCreate(Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	        setContentView(R.layout.activity_main);
	        if(!isGird){
	            String temp = "2";
	        }
	    }



	}