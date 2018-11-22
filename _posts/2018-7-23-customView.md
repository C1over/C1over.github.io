---
layout:     post   				    
title:    自定义view  （1）				 
subtitle:  view     #副标题
date:       2018-7-23			   	# 时间
author:     Cc1over				# 作者
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Android
    - view
---

## 自定义view的步骤：
* 自定义属性的声明与获取
* 测量onMeasure
* 布局onLayout(ViewGroup)
* 绘制onDraw
* onTouchEvent
* onInterceTouchEvent(ViewGroup)
### 1）自定义属性声明与获取
1) 分析需要的自定义属性<br>
2) 在res/values/attrs.xml定义声明<br>
3) 在layout xml文件中进行使用<br>
4) 在view的构造方法中进行获取

~~~
<?xml version="1.0" encoding="utf-8"?>
<resources>
    
    <declare-styleable name="CustomTitleView">
        <attr name="titleText" format="string" />
        <attr name="titleTextColor" format="color" />
        <attr name="titleTextSize" format="dimension" />
    </declare-styleable>
 
</resources>
~~~
~~~

    public ViewDemo(Context context, AttributeSet attrs) {
        super(context, attrs);
        /**
         * TypedArray：对象描述类似数组的一个潜在的二进制数据的缓冲区(官方描述)
         * 就是系统在默认的资源文件R.styleable中去获取相关的配置。
         * 如果appearance不为空，它就会去寻找获取相关属性
         * 也就是冲我们自定属性样式中，来引用你需要的某条属性
        */
        TypedArray typedArray = context.obtainStyledAttributes(attrs,R.styleable.Myview);
        int colors =typedArray.getColor(R.styleable.Myview_rect_color,0xffff0000);//给他赋值一个红色
        setBackgroundColor(colors);

        typedArray.recycle();
    }
~~~
### 2）测量onMeasure
1) EXACTLY，AT_MOST，UNSPECIFIED<br>
2) MeasureSpec<br>
3) setMeasuredDimension<br>
4) requestLayout（）

~~~
private int measureHeight(int heightMeasureSpec) {
        int result = 0;
        int mode = View.MeasureSpec.getMode(heightMeasureSpec);
        int size = View.MeasureSpec.getSize(heightMeasureSpec);
        if (mode == View.MeasureSpec.EXACTLY) {
            result = size
        } else {
            result = getNeedHeight()+getPaddingTop()+getPaddingBottom();
            if(mode==View.MeasureSpec.AT_MOST){
                result = Math.min(result,size);
            }
        }
        return result;
    }
~~~
### 3）布局onLayout（ViewGroup）
1) 决定子view的位置<br>
2) 尽可能将onMeasure中的一些操作移动到此方法中<br>
3) requestLayout（）

~~~
 private void onLayout(boolean change,int l,int t,int r,int b){
         final int childCount = getChildCount();
         for(int i=0;i<childCount;i++){
             final View child = getChildAt();
             if(child.getVisibility()==View.GONE){
                 continue;
             }
             left = caculateChildLeft(); // 计算childview左上角x坐标
             top = caculateChildTop(); // 计算childview左上角y坐标
             child.layout(left,top,left+cWidth,top+cWidth);
         }
    }
~~~
### 4）绘制onDraw
1) 绘制内容区域<br>
2) invalidate（），postInvalidate（）<br>
3) Canvas.drewXXX<br>
4) translate、rotate、scale、skew<br>
5) save（）、restore（）

~~~
 protected synchoronized void onDraw(Canvas canvas){
     // 使用canvas相关API绘制anything you want
 }
~~~
### 5）onTouchEvent
1) ACTION_DOWN、
  ACTION_MOVE、
  ACTION_UP<br>
2) ACTION_POINTER_DOWN、
  ACTION_POINTER_UP<br>
3) parent.requestDisallowIntercepetTouchEvent(true);<br>
4) VelocityTracker

~~~
public boolean onTouchEvent(MotionEvent ev) {
        initVelocityTrackerIfNotExists();
        mVelocityTracker.addMovement(ev);

        final int action = ev.getAction();
        switch (action) {
            case MotionEvent.ACTION_DOWN:
                //进行一些初始化赋值的操作
                break;
            case MotionEvent.ACTION_MOVE:
                break;
            case MotionEvent.ACTION_UP:
                // 如果需要进行加速度判断
                int initialVelocity = (int) veloctyTracker.getYVelocity(mActivePointerId);
                // 释放各种资源，重置变量
                break;
            case MotionEvent.ACTION_CANCEL:
                // 释放各种资源，重置变量
                break;
            case MotionEvent.ACTION_POINTER_DOWN:
                // 如果支持多指，在此设置activePointer
                final int index = ev.getActionIndex();
                mLastMotionY = (int) ev.getY(index);
                mActivePointerId = ev.getPointerId(newPointerIndex);
                break;
            case MotionEvent.ACTION_POINTER_UP:
                if (pointerId == mActivePointerId) {
                    final int newPointerIndex = pointerIndex == 0 ? 1 : 0;
                    mLastMotionY = (int) ev.getY(newPointerIndex);
                    mActivePointerId = ev.getPointerId(newPointerIndex);
                    if(mVelocityTracker!=null){
                        mVelocityTracker.clear();
                    }
                }
                break;
        }
        return true;
    }

~~~
### 6）onInterceptTouchEvent
1）ACTION_DOWN、
   ACTION_MOVE、
   ACTION_UP<br>
2）ACTION_POINTER_DOWN、
  ACTION_POINTER_UP
3）决定是否拦截该手势
~~~
  @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {

        final int action = ev.getAction();
        switch (action & MotionEvent.ACTION_MASK) {
            case MotionEvent.ACTION_MOVE: {
                final int activePointerId = mActivePointerId;
                if (activePointerId == INVALID_POINTER) {
                    break;
                }
                final int y = (int) ev.getY(pointerIndex);
                final int yDiff = Math.abs(y - mLastMotionY);
                if (yDiff > mTouchSlop
                        && (getNestedScrollAxes() & ViewCompat.SCROLL_AXIS_VERTICAL) == 0) {
                    mIsBeingDragged = true;
                    mLastMotionY = y;
                    initVelocityTrackerIfNotExists();
                    mVelocityTracker.addMovement(ev);
                    mNestedYOffset = 0;
                    final ViewParent parent = getParent();
                    if (parent != null) {
                        parent.requestDisallowInterceptTouchEvent(true);
                    }
                }
                break;
            }
            case MotionEvent.ACTION_DOWN: {
                    break;
            case MotionEvent.ACTION_CANCEL:
            case MotionEvent.ACTION_UP:
                break;
        }
        return mIsBeingDragged;
    }
~~~
