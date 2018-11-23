---
layout:     post   				    
title:    ListView  				 
subtitle:  列表控件     #副标题
date:       2018-7-16			   	# 时间
author:     Cc1over				# 作者
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Android
    - 控件
---


## ListView常用技巧

#### 处理空ListView
~~~
   listView.setEmptyView（View）；
~~~
#### 监听ListView滑动
~~~
    public void onScrollStateChanged(AbsListView view, int scrollState){
        
    }
    
    public void onScroll(AbsListView view, int firstVisibleItem, int visibleItemCount,int totalItemCount) {
        
    }
~~~
firstVisibleItem：当前能看见的第一Item的ID（从0开始）

visibleItemCount：当前能看见的Item总数

totalItemCount：整个ListView的Item总数

~~~
   if(firstVisibleItem+visibleItemCount==totalItemCount){
      // 滑动到最后一行   
   }
~~~
~~~
  if(firstVisibleItem>lastVisibleItemPosition){
      // 上滑
  }
  if(firstVisibleItem<lastVisibleItemPosition){
      // 下滑
  }
  lastVisibleItem = firstVisibleItem；
~~~
通过TouchListener进行监听
~~~
    final float mTouchSlop = ViewConfiguration.get(this).getScaledTouchSlop();
        View.OnTouchListener listener = new View.OnTouchListener() {
            @Override
            public boolean onTouch(View view, MotionEvent motionEvent) {
                switch (motionEvent.getAction()) {
                    case MotionEvent.ACTION_DOWN:
                        mFirstY = motionEvent.getY();
                        break;
                    case MotionEvent.ACTION_MOVE:
                        mCurrentY = motionEvent.getY();
                        if (mCurrentY - mFirstY > mTouchSlop) {
                            // down
                        }
                        if ( mFirstY - mCurrentY> mTouchSlop) {
                            // up
                        }
                        break;
                    case MotionEvent.ACTION_UP:
                        break;
                }
                return false;
            }
        };
~~~
通过了滑动点的坐标改变大小，判断移动方向。
**initAbsListView内设置ListView本身可以点击即可以消耗父View分发的事件： setClickable(true); **

#### 局部刷新ListView
**实现原理：**<br> 1）找到需要更新的item在adapter中的位置<br> 2）更新adapter中item的数据data<br> 3）如果该item在listView当前屏的可见范围内则更新内容，否则不需要更新，待下次adapter刷新全部时再刷新

方案一：获取itemView里面的控件直接设置

~~~
View view = mListView.getChildAt(targetIndex - startShownIndex);
TextView textView = (TextView) view.findViewById(R.id.textView);
textView.setText(datas.get(targetIndex));
~~~
方案二：获取itemView然后调用getView方法

~~~
View view = mListView.getChildAt(targetIndex - startShownIndex);         
myAdapter.getView(targetIndex, view, mListView);//核心方法
~~~
方案三：通过CommonAdapter和CommondViewHolder实现上面的操作

~~~
private void updateOneTest() {
        setContentView(R.layout.activity_main);
        listView = (ListView) findViewById(R.id.listview);
        datas = new ArrayList<>();
        for (int i = 0; i < 20; i++) {
            datas.add("万能适配器测试" + i);
        }
        commonAdapter = new CommonAdapter<String>(this, datas, R.layout.item) {

            @Override
            protected void convertView(View item, String s) {
                TextView textView = CommonViewHolder.get(item, R.id.textView);
                textView.setText(s);
            }
        };
        listView.setAdapter(commonAdapter);
        listView.setOnItemClickListener(new AdapterView.OnItemClickListener() {
            @Override
            public void onItemClick(AdapterView<?> parent, View view, int position, long id) {
                datas.set(position, "update 万能适配器测试" + position);
                updateSingle(position);
            }
        });
    }
~~~
~~~
private void updateSingle(int position) {
        // 第一个可见的位置
        int firstVisiblePosition = listView.getFirstVisiblePosition();
        // 最后一个可见的位置
        int lastVisiblePosition = listView.getLastVisiblePosition();

        // 在看见范围内才更新，不可见的滑动后自动会调用getView方法更新
        if (position >= firstVisiblePosition && position <= lastVisiblePosition) {
            // 获取指定位置view对象
            View view = listView.getChildAt(position - firstVisiblePosition);
            // findView方式
            // TextView textView = (TextView) view.findViewById(R.id.textView);
            // ViewHolder方式
            TextView textView = CommonViewHolder.get(view, R.id.textView);
            // 调用getView方式
            commonAdapter.getView(position, view, listView);
            textView.setText(datas.get(position));
        }
    }
~~~


#### RecycleBin机制
节选自部分的源码

~~~

/**
 * The RecycleBin facilitates reuse of views across layouts. The RecycleBin
 * has two levels of storage: ActiveViews and ScrapViews. ActiveViews are
 * those views which were onscreen at the start of a layout. By
 * construction, they are displaying current information. At the end of
 * layout, all views in ActiveViews are demoted to ScrapViews. ScrapViews
 * are old views that could potentially be used by the adapter to avoid
 * allocating views unnecessarily.
 * 
 * @see android.widget.AbsListView#setRecyclerListener(android.widget.AbsListView.RecyclerListener)
 * @see android.widget.AbsListView.RecyclerListener
 */
class RecycleBin {
	private RecyclerListener mRecyclerListener;
 
	/**
	 * The position of the first view stored in mActiveViews.
	 */
	private int mFirstActivePosition;
 
	/**
	 * Views that were on screen at the start of layout. This array is
	 * populated at the start of layout, and at the end of layout all view
	 * in mActiveViews are moved to mScrapViews. Views in mActiveViews
	 * represent a contiguous range of Views, with position of the first
	 * view store in mFirstActivePosition.
	 */
	private View[] mActiveViews = new View[0];
 
	/**
	 * Unsorted views that can be used by the adapter as a convert view.
	 */
	private ArrayList<View>[] mScrapViews;
 
	private int mViewTypeCount;
 
	private ArrayList<View> mCurrentScrap;
 
	/**
	 * Fill ActiveViews with all of the children of the AbsListView.
	 * 
	 * @param childCount
	 *            The minimum number of views mActiveViews should hold
	 * @param firstActivePosition
	 *            The position of the first view that will be stored in
	 *            mActiveViews
	 */
	void fillActiveViews(int childCount, int firstActivePosition) {
		if (mActiveViews.length < childCount) {
			mActiveViews = new View[childCount];
		}
		mFirstActivePosition = firstActivePosition;
		final View[] activeViews = mActiveViews;
		for (int i = 0; i < childCount; i++) {
			View child = getChildAt(i);
			AbsListView.LayoutParams lp = (AbsListView.LayoutParams) child.getLayoutParams();
			// Don't put header or footer views into the scrap heap
			if (lp != null && lp.viewType != ITEM_VIEW_TYPE_HEADER_OR_FOOTER) {
				// Note: We do place AdapterView.ITEM_VIEW_TYPE_IGNORE in
				// active views.
				// However, we will NOT place them into scrap views.
				activeViews[i] = child;
			}
		}
	}
 
	/**
	 * Get the view corresponding to the specified position. The view will
	 * be removed from mActiveViews if it is found.
	 * 
	 * @param position
	 *            The position to look up in mActiveViews
	 * @return The view if it is found, null otherwise
	 */
	View getActiveView(int position) {
		int index = position - mFirstActivePosition;
		final View[] activeViews = mActiveViews;
		if (index >= 0 && index < activeViews.length) {
			final View match = activeViews[index];
			activeViews[index] = null;
			return match;
		}
		return null;
	}
 
	/**
	 * Put a view into the ScapViews list. These views are unordered.
	 * 
	 * @param scrap
	 *            The view to add
	 */
	void addScrapView(View scrap) {
		AbsListView.LayoutParams lp = (AbsListView.LayoutParams) scrap.getLayoutParams();
		if (lp == null) {
			return;
		}
		// Don't put header or footer views or views that should be ignored
		// into the scrap heap
		int viewType = lp.viewType;
		if (!shouldRecycleViewType(viewType)) {
			if (viewType != ITEM_VIEW_TYPE_HEADER_OR_FOOTER) {
				removeDetachedView(scrap, false);
			}
			return;
		}
		if (mViewTypeCount == 1) {
			dispatchFinishTemporaryDetach(scrap);
			mCurrentScrap.add(scrap);
		} else {
			dispatchFinishTemporaryDetach(scrap);
			mScrapViews[viewType].add(scrap);
		}
 
		if (mRecyclerListener != null) {
			mRecyclerListener.onMovedToScrapHeap(scrap);
		}
	}
 
	/**
	 * @return A view from the ScrapViews collection. These are unordered.
	 */
	View getScrapView(int position) {
		ArrayList<View> scrapViews;
		if (mViewTypeCount == 1) {
			scrapViews = mCurrentScrap;
			int size = scrapViews.size();
			if (size > 0) {
				return scrapViews.remove(size - 1);
			} else {
				return null;
			}
		} else {
			int whichScrap = mAdapter.getItemViewType(position);
			if (whichScrap >= 0 && whichScrap < mScrapViews.length) {
				scrapViews = mScrapViews[whichScrap];
				int size = scrapViews.size();
				if (size > 0) {
					return scrapViews.remove(size - 1);
				}
			}
		}
		return null;
	}
 
	public void setViewTypeCount(int viewTypeCount) {
		if (viewTypeCount < 1) {
			throw new IllegalArgumentException("Can't have a viewTypeCount < 1");
		}
		// noinspection unchecked
		ArrayList<View>[] scrapViews = new ArrayList[viewTypeCount];
		for (int i = 0; i < viewTypeCount; i++) {
			scrapViews[i] = new ArrayList<View>();
		}
		mViewTypeCount = viewTypeCount;
		mCurrentScrap = scrapViews[0];
		mScrapViews = scrapViews;
｝
~~~
fillActiveViews() 这个方法接收两个参数，第一个参数表示要存储的view的数量，第二个参数表示ListView中第一个可见元素的position值。RecycleBin当中使用mActiveViews这个数组来存储View，调用这个方法后就会根据传入的参数来将ListView中的指定元素存储到mActiveViews数组当中。<br>

getActiveView() 这个方法和fillActiveViews()是对应的，用于从mActiveViews数组当中获取数据。该方法接收一个position参数，表示元素在ListView当中的位置，方法内部会自动将position值转换成mActiveViews数组对应的下标值。需要注意的是，mActiveViews当中所存储的View，一旦被获取了之后就会从mActiveViews当中移除，下次获取同样位置的View将会返回null，也就是说mActiveViews不能被重复利用。<br>

addScrapView() 用于将一个废弃的View进行缓存，该方法接收一个View参数，当有某个View确定要废弃掉的时候(比如滚动出了屏幕)，就应该调用这个方法来对View进行缓存，RecycleBin当中使用mScrapViews和mCurrentScrap这两个List来存储废弃View。<br>
**几个重要成员的描述：**<br>

mActiveViews          // View[ ]  存放当前可见View，也就是上图6个可见的Item <br>

mCurrentScrap       // ArrayList<View> 存放废弃的View，也就是当Item1滑出屏幕后，就被添加到这个list中 <br>

mScrapViews        // ArrayList<View>[ ]  存放废弃的Views，这个数组是在多类型布局中用到，与它有关的变量ViewTypeCount，在adapter使用了getViewTypeCount() 后，会把View缓存到这个数组中<br>

**简单扩展一下上述的getViewTypeCount（）方法：**<br>
如果在一个ListView中要实现多种样式的ListView布局样式，则需要在ListView的适配器Adapter中用到：getItemViewType()和getViewTypeCount()。getViewTypeCount()告诉ListView需要加载多少种类型的Item View，getItemViewType()则告诉ListView在某一位置（position）的Item View样式是什么。

好的，回正题了：继续分析上面源码中的关键方法<br>
getScrapView 用于从废弃缓存中取出一个View，这些废弃缓存中的View是没有顺序可言的，因此getScrapView()方法中的算法也非常简单，就是直接从mCurrentScrap当中获取尾部的一个scrap view进行返回。<br>

上面也提到了Adapter当中可以重写一个getViewTypeCount()来表示ListView中有几种类型的数据项，而setViewTypeCount()方法的作用就是为每种类型的数据项都单独启用一个RecycleBin缓存机制。<br>

#### 目前对ListView源码的理解
ListView也是一种View，它的工作原理自然离不开measure，layout，draw这三个流程，而且其实View显示在界面上至少会经过2次measure和layout的过程，所以其实在ListView中也会有这个流程，所以下面就来总结一下两次测量的差异：<br>

**第一次测量：**<br>
第一次测量的时候ListView中没有子View。<br>
PS:**(当ListView.setAdapter之后ListView里有数据才会有子View)**
查找到layout关键代码步骤如下：<br>
onLayout(AbsListView类) <br>
layoutChildren(ListView类) <br>
fillFromTop(ListView类) <br>
fillDown(ListView类) <br>
makeAndAddView(ListView类)<br> 
setupChild(ListView类)<br>

**第二次测量：**<br>
第二次测量的时候ListView中已经拥有了子View。 <br>
查找到layout关键代码步骤如下： <br>
onLayout(AbsListView类) <br>
layoutChildren(ListView类) <br>
fillSpecific(ListView类) <br>
fillDown(ListView类) <br>
makeAndAddView(ListView类) <br>
setupChild(ListView类) <br>

**小结：**ListView内部是使用RecycleBin进行子View的复用，而两次测量最终都会进入到setupchild的方法中，第一次进入把数据放入RecycleBin中，而第二次则是从里面去取出数据。

