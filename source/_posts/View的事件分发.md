---
title: View的事件分发
categories:
  - Android
tags:
  - Android
  - View
  - 源码
  - 事件分发
comments: true
copyright: true
feature_img: false
cover: 'https://cdn.littlecorgi.top/blog/Snipaste_2019-10-31_23-03-31.png'
abbrlink: bfb9e8a9
date: 2019-10-31 22:58:27
updated: 2019-10-31 22:58:27
keywords:
description:
---
从我刚进实验室的时候，学长学姐就说View的事件分发机制是Android里面一个很重要的内容，要我们好好学。

但是随着自己对Android了解的深入，越发觉得这个东西很有必要了解下，正好Android艺术开发探索也看到了View这块，也看了[郭霖大神的博客](https://blog.csdn.net/guolin_blog/article/details/9097463)和[另一位大神的博客](https://www.jianshu.com/p/ea0108d8510e)，所以就好好学习了一番，并写了此博客。


# 1. MotionEvent
在开始讲View事件分发之前，我们先来了解下MotionEvent。

这个就是手指解除到屏幕后所产生的一系列事件，主要为一下三个典型事件：
- ACTION_DOWM——手指刚接触屏幕
- ACTION_MOVE——手指在屏幕上移动
- ACTION_UP——手指从屏幕上松开
- ACTION_CANCEL——结束事件（非人为）

正常情况下，一次手指触摸屏幕然后离开可能触发一下两种情况：
- 点击屏幕然后松开：ACTION_DOWN -> ACTION_UP
- 点击屏幕然后滑动再松开：ACTION_DOWN -> ACTION_MOVE -> ACTION_UP

下面一个图片来概括下：
![MotionEvent](https://cdn.littlecorgi.top/mweb/2019-10-31/MotionEvent.png)

# 2. 事件分发传递规则
众所周知，AndroidUI是由Activity、ViewGroup、View及其派生类组成的。

大致示意图如下：
![AndroidUI](https://cdn.littlecorgi.top/mweb/2019-10-31/AndroidUI.jpg)

其中：
- Activity：控制生命周期或者处理事件
- ViewGroup：一组View或者多个View的集合。也是布局Layout的基类。但是特别的是，他也集成自View。
- View：所有UI组件的基类

从上图我们就可以看出来，事件分发的顺序就是：Activity -> ViewGroup -> View。也就是说一个点击事件产生，先交由Activity，再传到ViewGroup，再传到View。这个过程中只要有一个部分说要拦截，就不会再继续往下传递。

# 3. 事件分发的核心方法
其实时间分发的核心方法很简单，就由三个方法组成：

- public boolean dispatchTouchEvent(MotionEvent ev)
用来进行事件的分发。如果事件能够传递给当前View，那么此方法一定会被调用，返回结果受当前View的onTouchEvent和下级View的dispatchTouchEvent方法的影响，表示是否消耗当前事件。
- public boolean onInterceptTouchEvent(MotionEvent event)
在dispatchTouchEvent方法内部调用，用来判断是否拦截某个事件，如果当前View拦截了某个事件，那么在同一事件序列当中，此方法不会再次被调用，返回结果表示是否拦截当前事件。（只有ViewGroup中才有此方法，View中没有）

- public boolean onTouchEvent(MotionEvent event)
在dispatchTouchEvent方法中调用，用来处理点击事件，返回结果表示是否消耗当前事件，如果不消耗，则在同一个事件序列中，当前View无法再次接收到事件。

三个方法间的关系可以按下面一段伪代码表示：（[参考](https://www.jianshu.com/p/ea0108d8510e)的代码）
```java

/**
  * 点击事件产生后 
  */ 
  // 步骤1：调用dispatchTouchEvent（）
  public boolean dispatchTouchEvent(MotionEvent ev) {

    boolean consume = false; //代表 是否会消费事件

    // 步骤2：判断是否拦截事件
    if (onInterceptTouchEvent(ev)) {
      // a. 若拦截，则将该事件交给当前View进行处理
      // 即调用onTouchEvent (）方法去处理点击事件
        consume = onTouchEvent (ev) ;

    } else {

      // b. 若不拦截，则将该事件传递到下层
      // 即 下层元素的dispatchTouchEvent（）就会被调用，重复上述过程
      // 直到点击事件被最终处理为止
      consume = child.dispatchTouchEvent (ev) ;
    }

    // 步骤3：最终返回通知 该事件是否被消费（接收 & 处理）
    return consume;
}
```
对于一个根ViewGroup，点击事件产生后，就会传递给他，这时他的dispatchTouchEvent就会被调用，然后就开始判断他是否拦截。如果拦截，那么点击事件就会给ViewGroup去处理，如果不拦截，就调用child.dispatchTouchEvent (ev) 传给子控件的dispatchTouchEvent方法。然后继续循环，直到到最底层view，也就是没有child的时候，或者直到事件被拦截。

# 4. 源码分析
## 4.1 Activity事件分发
首先先来看dispatchTouchEvent方法：
### 源码
```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();
        // ->> 分析1
    }
    if (getWindow().superDispatchTouchEvent(ev)) {
    // ->> 分析2
        return true;
    }
    return onTouchEvent(ev);
    // ->> 分析5
}

/**
  * 分析1：onUserInteraction()
  * 这个方法就是个空方法，但是我是在没搞懂这个方法是干嘛的，所以就直接粘贴了carson这位大神的说明
  * 作用：实现屏保功能
  * 注：
  *    a. 该方法为空方法
  *    b. 当此activity在栈顶时，触屏点击按home，back，menu键等都会触发此方法
  */
public void onUserInteraction() {
}

/**
  * 分析2：getWindow().superDispatchTouchEvent(ev)
  * 点开之后进入Window类，
  * 但是发现这个类是个抽象类，这个方法是个抽象方法。
  * 
  * 了解过View的同学都知道，Window的唯一实现类就是PhoneWindow
  * 那我们进入PhoneWindow看下他的superDispatchTouchEvent方法
  */
public abstract boolean superDispatchTouchEvent(MotionEvent event);
// ->> 分析3

/**
  * 分析3：PhoneWindow.superDispatchTouchEvent()
  * 实际上又调用了DecorView的superDispatchTouchEvent方法，DecorView是最顶层的View
  */
@Override
public boolean superDispatchTouchEvent(MotionEvent event) {
    return mDecor.superDispatchTouchEvent(event);
    // ->> 分析4
}

/**
 * 分析4：DecorView.superDispatchTouchEvent()
 * 注：
 *     a. DecorView继承自FrameLayout，而FrameLayout继承自ViewGroup，也就是说DecorView就是ViewGroup。
 *     b. DecorView调用了父类的dispatchTouchEvent方法，也就相当于ViewGroup的dispatchTouchEvent方法，就把事件交给了ViewGroup去处理，这块后面再说
 */
public boolean superDispatchTouchEvent(MotionEvent event) {
    return super.dispatchTouchEvent(event);
}

/**
 * 分析5：return onTouchEvent(ev)
 * 当触摸屏事件未由其下的任何视图处理时调用
 * 
 */
public boolean onTouchEvent(MotionEvent event) {
    if (mWindow.shouldCloseOnTouch(this, event)) {
    // ->> 分析6
        finish();
        return true;
    }

    return false;
    // 只有在点击事件在Window边界外才会返回true，一般情况都返回false，分析完毕
}

/** 
 * 分析6：Window.shouldCloseOnTouch()
 * 
 * 返回：
 *         返回true：说明事件在边界外，即 消费事件
 *         返回false：未消费（默认）
 */
public boolean shouldCloseOnTouch(Context context, MotionEvent event) {
    final boolean isOutside =
            event.getAction() == MotionEvent.ACTION_DOWN && isOutOfBounds(context, event)
            || event.getAction() == MotionEvent.ACTION_OUTSIDE;
    // 主要是对于处理边界外点击事件的判断：是否是DOWN事件，event的坐标是否在边界内等
    if (mCloseOnTouchOutside && peekDecorView() != null && isOutside) {
        // peekDecorView是返回PhoneWindow的mDecor
        return true;
    }
    return false;
}
```

### 流程
![Activity事件分发流程](https://cdn.littlecorgi.top/mweb/2019-10-31/Activity%E4%BA%8B%E4%BB%B6%E5%88%86%E5%8F%91%E6%B5%81%E7%A8%8B.jpg)

## 4.2 ViewGroup事件分发
### 源码
由于此部分代码过长，我们将代码拆分成两部分：
#### part1
```java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    ...// 仅贴出关键代码
    
    boolean handled = false;
    /**
     * onFilterTouchEventForSecurity(ev)
     * 筛选touch事件，进去之后的判断核心就是当前视图是否被其它窗口遮挡或者隐藏
     */
    if (onFilterTouchEventForSecurity(ev)) {
        final int action = ev.getAction();
        final int actionMasked = action & MotionEvent.ACTION_MASK;

        // Handle an initial down.
        // ->> 分析1
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            // Throw away all previous state when starting a new touch gesture.
            // The framework may have dropped the up or cancel event for the previous gesture
            // due to an app switch, ANR, or some other state change.
            cancelAndClearTouchTargets(ev);
            resetTouchState();
        }

        // Check for interception.
        final boolean intercepted;
        /**
         * 分析2
         * 可以看到会有两种情况下拦截事件：事件类型为DOWN，或者mFirstTouchTarget != null。
         * 那么这个mFirstTouchTarget是什么呢？
         * 从后面的代码我们可以得知，当ViewGroup不拦截事件交给子元素处理的时候，mFirstTouchTarget不为null。
         * 所以，也就是说当MotionEvent为UP或者MOVE的时候，都进不去这个方法，也就是不调用ViewGroup的onInterceptTouchEvent，他不拦截事件
         */
        if (actionMasked == MotionEvent.ACTION_DOWN
                || mFirstTouchTarget != null) {
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            if (!disallowIntercept) {
                intercepted = onInterceptTouchEvent(ev);
                ev.setAction(action); // restore action in case it was changed
            } else {
                intercepted = false;
            }
        } else {
            // There are no touch targets and this action is not an initial down
            // so this view group continues to intercept touches.
            intercepted = true;
        }
        
/**
 * 分析1
 * 事件开始时会调用resetTouchState()来清空mFirstAndClearTouchTarget
 */
if (actionMasked == MotionEvent.ACTION_DOWN) {
    cancelAndClearTouchTargets(ev);
    // 核心目的就是清空mFirstAndClearTouchTarget
    resetTouchState();
}
```

#### part2
```java
        ...//省略中间代码
        // 进入if语句，判断条件为没有对事件进行拦截，同时事件没有结束。对ViewGroup的子元素进行遍历
        if (!canceled && !intercepted) {
        
            ...//继续省略中间代码
            
                    // 得到所有的子View
                    final View[] children = mChildren;
                    // 对子View进行遍历
                    for (int i = childrenCount - 1; i >= 0; i--) {
                        final int childIndex = getAndVerifyPreorderedIndex(
                                childrenCount, i, customOrder);
                        final View child = getAndVerifyPreorderedView(
                                preorderedList, children, childIndex);

                        // If there is a view that has accessibility focus we want it
                        // to get the event first and if not handled we will perform a
                        // normal dispatch. We may do a double iteration but this is
                        // safer given the timeframe.
                        // 该viewgroup设置事件了指向焦点View并且焦点View在前面已经找到了
                        if (childWithAccessibilityFocus != null) {
                            // 判断遍历的此View是否是焦点View，如果不是就直接下一遍循环
                            if (childWithAccessibilityFocus != child) {
                                continue;
                            }
                            // 如果是的话就将找到的焦点view置空
                            // i回到到数第一个下标
                            // 这样做的目的是先让该焦点view尝试进行下面的普通分发操作
                            // 如果成功了，会在下面跳出循环。
                            // 如果不成功，就将记录的焦点view置空，
                            // 从最后一个开始重新遍历，不再进入这个判断。
                            childWithAccessibilityFocus = null;
                            i = childrenCount - 1;
                        }

                        // 判断当前子View是否能获取焦点或者是否正在做动画
                        if (!canViewReceivePointerEvents(child)
                                || !isTransformedTouchPointInView(x, y, child, null)) {
                            ev.setTargetAccessibilityFocus(false);
                            continue;
                        }

                        newTouchTarget = getTouchTarget(child);
                        if (newTouchTarget != null) {
                            // Child is already receiving touch within its bounds.
                            // Give it the new pointer in addition to the ones it is handling.
                            newTouchTarget.pointerIdBits |= idBitsToAssign;
                            break;
                        }

                        resetCancelNextUpFlag(child);
                        // ->> 分析1
                        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                            // Child wants to receive touch within its bounds.
                            mLastTouchDownTime = ev.getDownTime();
                            if (preorderedList != null) {
                                // childIndex points into presorted list, find original index
                                for (int j = 0; j < childrenCount; j++) {
                                    if (chifcldren[childIndex] == mChildren[j]) {
                                        mLastTouchDownIndex = j;
                                        break;
                                    }
                                }
                            } else {
                                mLastTouchDownIndex = childIndex;
                            }
                            mLastTouchDownX = ev.getX();
                            mLastTouchDownY = ev.getY();
                            // ->> 分析2
                            newTouchTarget = addTouchTarget(child, idBitsToAssign);
                            alreadyDispatchedToNewTouchTarget = true;
                            break;
                        }

                        // The accessibility focus didn't handle the event, so clear
                        // the flag and do a normal dispatch to all children.
                        ev.setTargetAccessibilityFocus(false);
                    }
                  
                  ... //省略
        }

/**
 * 分析1 dispatchTransformedTouchEvent()
 * 
 * 核心就是那个if (child == null)。
 * 如果child不为空，那么事件就交给子View处理
 */
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
        View child, int desiredPointerIdBits) {
    final boolean handled;

    // Canceling motions is a special case.  We don't need to perform any transformations
    // or filtering.  The important part is the action, not the contents.
    final int oldAction = event.getAction();
    if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
        event.setAction(MotionEvent.ACTION_CANCEL);
        if (child == null) {
            handled = super.dispatchTouchEvent(event);
        } else {
            handled = child.dispatchTouchEvent(event);
        }
        event.setAction(oldAction);
        return handled;
    }
    ...//省略
}

/**
 * 分析2
 * 如果子View能处理点击事件，那么就调用addTouchTarget方法，对mFirstTouchTarget方法进行复制，然后再进入part1中分析2。
 */
private TouchTarget addTouchTarget(@NonNull View child, int pointerIdBits) {
    final TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
    target.next = mFirstTouchTarget;
    mFirstTouchTarget = target;
    return target;
}
```

#### part3
```java
        // Dispatch to touch targets.
        // 如果part2开头的那个if没有进，也就是对事件进行拦截的话，直接到这来
        // 并且如果没有进入part2，那么mFirstTouchTarget仍然为空，那么就进入if
        if (mFirstTouchTarget == null) {
            // No touch targets so treat this as an ordinary view.
            // 此处在上面分析过了，但是不同的是由于child直接传入了null，那么就执行super.dispatchTouchEven。
            // 那么super是谁呢？我们在前面说过，ViewGroup是继承自View的，那么他就是执行View的dispatchTouchEvent。
            handled = dispatchTransformedTouchEvent(ev, canceled, null,
                    TouchTarget.ALL_POINTER_IDS);
        } else {
            ... //省略
        }

        // Update list of touch targets for pointer up or cancel, if needed.
        if (canceled
                || actionMasked == MotionEvent.ACTION_UP
                || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                // ->> 分析1
            resetTouchState();
        } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
            final int actionIndex = ev.getActionIndex();
            final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
            removePointersFromTouchTargets(idBitsToRemove);
        }
    }
    
    ...//省略
    
}

/**
 * 分析1
 * 在同一事件系列结束后调用resetTouchState();
 */
private void resetTouchState() {
    // 对mFirstTouchTarget清空还原
    clearTouchTargets();
    resetCancelNextUpFlag(this);
    mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
    mNestedScrollAxes = SCROLL_AXIS_NONE;
}
```

### 流程
此图搬自[Carson_Ho大佬的博客](https://www.jianshu.com/p/38015afcdb58)
![](https://cdn.littlecorgi.top/mweb/2019-10-31/15725314562031.jpg)

## 4.3 View事件分发
### 源码
#### dispatchTouchEvent
```java
public boolean dispatchTouchEvent(MotionEvent event) {
    ...//省略

    boolean result = false;

    ...//省略

    // 和ViewGroup一样的判断，判断当前视图是否被遮挡或者不可见
    if (onFilterTouchEventForSecurity(event)) {
        if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
            result = true;
        }
        //noinspection SimplifiableIfStatement
        ListenerInfo li = mListenerInfo;
        // 检测有没有设置OnTOuchListener，如果有，并且onTouch方法返回true那么进入if，结果导致onTouchEvent不被执行
        if (li != null && li.mOnTouchListener != null
                && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) {
                
            result = true;
        }

        // 如果没进入上面的if，也就相当于没有设置OnTouchListener，那么执行onTOuchEvent
        if (!result && onTouchEvent(event)) {
            result = true;
        }
    }

    ...//省略

    return result;
}
```

从上面可以看出，**onTouch的优先级高于onTouchEvent**。

#### onTouchEvent
```java
public boolean onTouchEvent(MotionEvent event) {
    final float x = event.getX();
    final float y = event.getY();
    final int viewFlags = mViewFlags;
    final int action = event.getAction();

    /**
     * 注释1 
     * 只有在Click、LongClick、contextClick都不可用的时候才为false
     */
    final boolean clickable = ((viewFlags & CLICKABLE) == CLICKABLE
            || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
            || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE;

    // 首先判断是不是不可用，如果是，则进入if
    if ((viewFlags & ENABLED_MASK) == DISABLED) {
        if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
            setPressed(false);
        }
        mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
        // A disabled view that is clickable still consumes the touch
        // events, it just doesn't respond to them.
        // ->> 注释1（见上面👆）
        return clickable;
    }
    // 如果View有代理会执行这个方法
    if (mTouchDelegate != null) {
        if (mTouchDelegate.onTouchEvent(event)) {
            return true;
        }
    }
    
    // 这块我们只调出ACTION_UP来看
    // 只要clickable为true(见上面注释1👆)或者TOOLTIP(可能是Android8.0新出的提示功能吧)
    if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
        switch (action) {
            case MotionEvent.ACTION_UP:

                ...//省略
                
                    if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                        // This is a tap, so remove the longpress check
                        removeLongPressCallback();

                        // Only perform take click actions if we were in the pressed state
                        if (!focusTaken) {
                            // Use a Runnable and post this rather than calling
                            // performClick directly. This lets other visual state
                            // of the view update before click actions start.
                            if (mPerformClick == null) {
                                mPerformClick = new PerformClick();
                            }
                            if (!post(mPerformClick)) {
                                // ->> 分析1
                                performClickInternal();
                            }
                        }
                    }

                    ... //省略
                    
                }
                mIgnoreNextUpEvent = false;
                break;
            
            ... //省略其它几种MotionEvent
        }

        return true;
    }
    // 由代码可知只要上面的if语句成立，不管进入switch中的任何ACTION或是都不进入，返回值都是true，即事件消费了。
    return false;
}

/**
  * 分析1：performClick（）
  */
public boolean performClick() {  

    if (mOnClickListener != null) {  
        playSoundEffect(SoundEffectConstants.CLICK);  
        mOnClickListener.onClick(this);  
        return true;  
        // 只要我们通过setOnClickListener（）为控件cvView注册1个点击事件
        // 那么就会给mOnClickListener变量赋值（即不为空）
        // 则会往下回调onClick（） & performClick（）返回true
    }  
    return false;  
} 
```

### 流程
![](https://cdn.littlecorgi.top/mweb/2019-10-31/15725337657670.jpg)
