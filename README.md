# Viewpage-TouchEvent
该片主要讲解ViewPage的事件分发机制，需要对于View的事件分发有一定的了解，我们知道ViewPage+滑动列表正常情况下会出现滑动冲突，但在ViewPage中，自动帮我们处理了滑动冲突事件，在onInterceptTouchEvent方法中，接下来就来分析一下：
    
    /**
     * 判断是否拦截当前触摸事件，return true，表示需要内部处理滑动事件
     */

    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {

        //当前状态
        final int action = ev.getAction() & MotionEvent.ACTION_MASK;

        // 判断滑动状态是否停止
        if (action == MotionEvent.ACTION_CANCEL || action == MotionEvent.ACTION_UP) {
            // Release the drag.
            if (DEBUG) Log.v(TAG, "Intercept done!");
            //重置一些跟是否拦截的配置
            resetTouch();
            //触摸结束，无需拦截
            return false;
        }

        //判断是否正在滑动和是否能够拖曳
        if (action != MotionEvent.ACTION_DOWN) {
            //ViewPage正在滑动，拦截掉所有触摸事件
            if (mIsBeingDragged) {
                if (DEBUG) Log.v(TAG, "Intercept returning true!");
                return true;
            }
            //不能滑动，则所有触摸全部交给child处理
            if (mIsUnableToDrag) {
                if (DEBUG) Log.v(TAG, "Intercept returning false!");
                return false;
            }
        }

        switch (action) {

            //能执行到这里，说明mIsBeingDragged==false,也就是当前ViewPage没有滑动
            case MotionEvent.ACTION_MOVE: {

                final int activePointerId = mActivePointerId;
                if (activePointerId == INVALID_POINTER) {
                    // 如果当前触摸点不是一个有效的点，则不处理，因为下面返回mIsBeingDragged==false
                    break;
                }
                //执行到这里说明该点是一个有效的点，根据ID获取当前手指的index序列号（多点触摸）
                final int pointerIndex = ev.findPointerIndex(activePointerId);
                //接下来就是获取手指的xy点坐标，计算滑动的绝对值
                final float x = ev.getX(pointerIndex);
                final float dx = x - mLastMotionX;
                final float xDiff = Math.abs(dx);
                final float y = ev.getY(pointerIndex);
                final float yDiff = Math.abs(y - mInitialMotionY);
                if (DEBUG) Log.v(TAG, "Moved x to " + x + "," + y + " diff=" + xDiff + "," + yDiff);

                /**
                 * 判断当前显示的页面是否可以滑动，如果可以滑动，则将滑动事件丢给当前显示页面,否则ViewPage获取触摸事件
                 *
                 * ：：：：：：：解决滑动冲突
                 * isGutterDrag:判断是否在两页面的间隙滑动
                 * canScroll：判断当前页面是否可以滑动
                 */
                if (dx != 0 && !isGutterDrag(mLastMotionX, dx)
                        && canScroll(this, false, (int) dx, (int) x, (int) y)) {
                    mLastMotionX = x;
                    mLastMotionY = y;
                    //标记ViewPage不去拦截事件
                    mIsUnableToDrag = true;
                    //ViewPage不拦截
                    return false;
                }

                //能走到这，说明ViewPage获取了滑动触摸事件
                //若果X轴移动大于最小移动，且斜率小于0.5，则属于X轴
                if (xDiff > mTouchSlop && xDiff * 0.5f > yDiff) {
                    if (DEBUG) Log.v(TAG, "Starting drag!");
                    //ViePage内部处理
                    mIsBeingDragged = true;
                    //若果ViewPage还有父View，则申请触摸请求
                    requestParentDisallowInterceptTouchEvent(true);
                    //设置滚动状态为正在滚动
                    setScrollState(SCROLL_STATE_DRAGGING);
                    //保存当前位置
                    mLastMotionX = dx > 0
                            ? mInitialMotionX + mTouchSlop : mInitialMotionX - mTouchSlop;
                    mLastMotionY = y;
                    //启用缓存
                    setScrollingCacheEnabled(true);
                } else if (yDiff > mTouchSlop) {

                    if (DEBUG) Log.v(TAG, "Starting unable to drag!");

                    //这里表明ViewPage和child都不拦截滑动触摸事件，交给外部
                    //若Y轴移动大于最小移动，表示竖直方向的滑动
                    mIsUnableToDrag = true;
                }

                //move事件交由ViewPage，则跟随move页面滑动
                if (mIsBeingDragged) {
                    // 跟随手指滑动
                    if (performDrag(x)) {
                        ViewCompat.postInvalidateOnAnimation(this);
                    }
                }
                break;
            }

            case MotionEvent.ACTION_DOWN: {

                mLastMotionX = mInitialMotionX = ev.getX();
                mLastMotionY = mInitialMotionY = ev.getY();
                //获取当前点id
                mActivePointerId = ev.getPointerId(0);
                //默认能够拖动
                mIsUnableToDrag = false;
                //标记开始滚动
                mIsScrollStarted = true;
                //手动计算滚动偏移量
                mScroller.computeScrollOffset();

                //当前滚动状态为将要滚动到最终位置且当前位置距离最终位置足够远
                if (mScrollState == SCROLL_STATE_SETTLING
                        && Math.abs(mScroller.getFinalX() - mScroller.getCurrX()) > mCloseEnough) {
                    // 停止滑动
                    mScroller.abortAnimation();
                    mPopulatePending = false;
                    populate();
                    mIsBeingDragged = true;
                    //想父View申请触摸事件传递给ViewPage
                    requestParentDisallowInterceptTouchEvent(true);
                    setScrollState(SCROLL_STATE_DRAGGING);
                } else {
                    //停止滑动
                    completeScroll(false);
                    mIsBeingDragged = false;
                }

                if (DEBUG) {
                    Log.v(TAG, "Down at " + mLastMotionX + "," + mLastMotionY
                            + " mIsBeingDragged=" + mIsBeingDragged
                            + "mIsUnableToDrag=" + mIsUnableToDrag);
                }
                break;
            }

            case MotionEvent.ACTION_POINTER_UP:
                onSecondaryPointerUp(ev);
                break;
        }

        //获取点击事件的速度
        if (mVelocityTracker == null) {
            mVelocityTracker = VelocityTracker.obtain();
        }
        mVelocityTracker.addMovement(ev);

        /*
         * The only time we want to intercept motion events is if we are in the
         * drag mode.
         */
   
        return mIsBeingDragged;
    }
    
主要部分代码都已经谢了注释，这里讲一下流程ViewPage会首先判断用户触摸是否结束，来重置一些数据，接下来通过
    
    //判断是否正在滑动和是否能够拖曳
        if (action != MotionEvent.ACTION_DOWN) {
            //ViewPage正在滑动，拦截掉所有触摸事件
            if (mIsBeingDragged) {
                if (DEBUG) Log.v(TAG, "Intercept returning true!");
                return true;
            }
            //不能滑动，则所有触摸全部交给child处理
            if (mIsUnableToDrag) {
                if (DEBUG) Log.v(TAG, "Intercept returning false!");
                return false;
            }
        }
        
该方法mIsBeingDragged参数用来判断ViewPage是否滑动，若恩给你滑动，则ViewPage自身消费所有事件，mIsUnableToDrag参数判断ViewPage是否能够拖曳，若不能够则触摸事件交给子View处理，注意这里mIsUnableToDrag默认是不能滑动。杰西莱具体分析MOVE，UP事件：
    
    if (dx != 0 && !isGutterDrag(mLastMotionX, dx)
                        && canScroll(this, false, (int) dx, (int) x, (int) y)) {
                    mLastMotionX = x;
                    mLastMotionY = y;
                    //标记ViewPage不去拦截事件
                    mIsUnableToDrag = true;
                    //ViewPage不拦截
                    return false;
                }
                
该出处理了滑动冲突事件，若果当前手指移动距离>0且当前View可以滑动，则当前View获取触摸事件，mIsUnableToDrag==true表示ViewPage不能拖曳，直接返回false，表示child处理该事件
    
    if (xDiff > mTouchSlop && xDiff * 0.5f > yDiff) {
                    if (DEBUG) Log.v(TAG, "Starting drag!");
                    //ViePage内部处理
                    mIsBeingDragged = true;
                    //若果ViewPage还有父View，则申请触摸请求
                    requestParentDisallowInterceptTouchEvent(true);
                    //设置滚动状态为正在滚动
                    setScrollState(SCROLL_STATE_DRAGGING);
                    //保存当前位置
                    mLastMotionX = dx > 0
                            ? mInitialMotionX + mTouchSlop : mInitialMotionX - mTouchSlop;
                    mLastMotionY = y;
                    //启用缓存
                    setScrollingCacheEnabled(true);
                } else if (yDiff > mTouchSlop) {

                    if (DEBUG) Log.v(TAG, "Starting unable to drag!");

                    //这里表明ViewPage和child都不拦截滑动触摸事件，交给外部
                    //若Y轴移动大于最小移动，表示竖直方向的滑动
                    mIsUnableToDrag = true;
                }
若child不消费事件，则通过xDiff和yDiff来判断事件属于ViewPage还是外部View，最后通过该方法，来刷新ViewPage，该出有一句代码ViewCompat.postInvalidateOnAnimation(this)，通过点进去可以发信啊，最终还是调用view.invalidate()来刷新页面。


    if (mIsBeingDragged) {
                    // 跟随手指滑动
                    if (performDrag(x)) {
                        ViewCompat.postInvalidateOnAnimation(this);
                    }
                }
                
讲一下DOWN事件，期初刚看时，也是一脸懵逼，看不懂，多看两遍就行了
    
     //当前滚动状态为将要滚动到最终位置且当前位置距离最终位置足够远
                if (mScrollState == SCROLL_STATE_SETTLING
                        && Math.abs(mScroller.getFinalX() - mScroller.getCurrX()) > mCloseEnough) {
                    // 停止滑动
                    mScroller.abortAnimation();
                    mPopulatePending = false;
                    populate();
                    mIsBeingDragged = true;
                    //想父View申请触摸事件传递给ViewPage
                    requestParentDisallowInterceptTouchEvent(true);
                    setScrollState(SCROLL_STATE_DRAGGING);
                } else {
                    //停止滑动
                    completeScroll(false);
                    mIsBeingDragged = false;
                }

主要讲讲该方法，分为另种状态，如果ViewPage处于将要滑动到最终点，则停止滑动，像父控件申请触摸事件，设置滑动状态为正在拖曳，反之咋停止滑动，就是DOWN事件按下时判断是否处理，根据mIsBeingDragged可以判断该事件是否由ViewPage处理还是其child处理，若mIsBeingDragged==true，则后续事件交由ViewPage处理，
这里可能会感觉很错乱，这就需要对Android的事件分发机制有一定的了解：这里主要分析ViewPage的onInterceptTouchEvent中的事件分发：  
收件当用户触摸屏幕时，触发onInterceptTouchEvent的DOWN事件，在DOWN事件中会进行两种判断，具体可以参考上面源码的注释，返回mIsBeingDragged==true，则ViewPage自身消耗该事件，反之交由子child，当手指开始滑动时，首先触发onInterceptTouchEvent的MOVE事件（因为DOWN事件只会触发一次），会先判断触摸点当前页面是否可以滑动，以及水平滑动距离，若符合条件，则交由child处理，反之判断水平方向dx，和竖直方向dy，来判断时ViewPgae处理还是外部处理。若都不符合，则交由child处理，onInterceptTouchEvent中省略了该方法。在onInterceptTouchEvent中若mIsBeingDragged==true；则接下来所有事件交由onTouchEvent处理。接下来就看一下onTouchEvent的源码部分：
    
    case MotionEvent.ACTION_DOWN: {

                Log.i("mao","--------------------touch_down");
                mScroller.abortAnimation();
                mPopulatePending = false;
                populate();

                // Remember where the motion event started
                mLastMotionX = mInitialMotionX = ev.getX();
                mLastMotionY = mInitialMotionY = ev.getY();
                mActivePointerId = ev.getPointerId(0);
                break;
            }
            
主要是一些初始化终点，手指的ID的一些参数，
    
     case MotionEvent.ACTION_MOVE:

                Log.i("mao","--------------------touch_move");

                //若当前ViewPage没有滑动，则判断是否需要滑动
                if (!mIsBeingDragged) {
                    final int pointerIndex = ev.findPointerIndex(mActivePointerId);
                    //触摸点不可用，则重置触摸条件
                    if (pointerIndex == -1) {
                        // A child has consumed some touch events and put us into an inconsistent
                        // state.
                        needsInvalidate = resetTouch();
                        break;
                    }
                    final float x = ev.getX(pointerIndex);
                    final float xDiff = Math.abs(x - mLastMotionX);
                    final float y = ev.getY(pointerIndex);
                    final float yDiff = Math.abs(y - mLastMotionY);
                    if (DEBUG) {
                        Log.v(TAG, "Moved x to " + x + "," + y + " diff=" + xDiff + "," + yDiff);
                    }
                    //若横向距离足够大，且大于纵向距离，则可以滑动
                    if (xDiff > mTouchSlop && xDiff > yDiff) {
                        if (DEBUG) Log.v(TAG, "Starting drag!");
                        //设置为可以滑动
                        mIsBeingDragged = true;
                        //请求父类不拦截触摸时事件
                        requestParentDisallowInterceptTouchEvent(true);
                        mLastMotionX = x - mInitialMotionX > 0 ? mInitialMotionX + mTouchSlop :
                                mInitialMotionX - mTouchSlop;
                        mLastMotionY = y;
                        setScrollState(SCROLL_STATE_DRAGGING);
                        setScrollingCacheEnabled(true);

                        // Disallow Parent Intercept, just in case
                        ViewParent parent = getParent();
                        if (parent != null) {
                            parent.requestDisallowInterceptTouchEvent(true);
                        }
                    }
                }
                // ViewPage水平滑动
                if (mIsBeingDragged) {
                    // 获取当前手指的小标
                    final int activePointerIndex = ev.findPointerIndex(mActivePointerId);
                    //获取当前手指的x坐标
                    final float x = ev.getX(activePointerIndex);
                    //调用该方法，实现页面的滑动效果
                    needsInvalidate |= performDrag(x);
                }
                break;
                
在获取MOVE事件后判断ViewPage的滑动条件，实现滑动.
    
    case MotionEvent.ACTION_UP:

                Log.i("mao","--------------------touch_up");

                //手指抬起,判断是否可以继续滑动
                if (mIsBeingDragged) {
                    final VelocityTracker velocityTracker = mVelocityTracker;
                    velocityTracker.computeCurrentVelocity(1000, mMaximumVelocity);
                    int initialVelocity = (int) velocityTracker.getXVelocity(mActivePointerId);
                    mPopulatePending = true;
                    //获取实际ViewPage宽度减去左右边距（显示宽度）
                    final int width = getClientWidth();
                    //返回View左边界的位置
                    final int scrollX = getScrollX();
                    //计算出当前页面信息
                    final ItemInfo ii = infoForCurrentScrollPosition();
                    //边缘占比
                    final float marginOffset = (float) mPageMargin / width;
                    //当前页面
                    final int currentPage = ii.position;
                    //计算当前页面偏移
                    final float pageOffset = (((float) scrollX / width) - ii.offset)
                            / (ii.widthFactor + marginOffset);
                    final int activePointerIndex = ev.findPointerIndex(mActivePointerId);
                    final float x = ev.getX(activePointerIndex);
                    //横向偏移
                    final int totalDelta = (int) (x - mInitialMotionX);
                    //获取需要切换的页面
                    int nextPage = determineTargetPage(currentPage, pageOffset, initialVelocity,
                            totalDelta);
                    //实现页面切换
                    setCurrentItemInternal(nextPage, true, true, initialVelocity);

                    needsInvalidate = resetTouch();
                }
                break;
                
UP 事件，先判断手指抬起后，ViewPage是否继续滑动，若滑动则计算出滑动到的页面，实现页面切换。后面还有一些多只触摸的操作：
    
    /**
             * 多指触摸
             */
            case MotionEvent.ACTION_POINTER_DOWN: {
                final int index = ev.getActionIndex();
                final float x = ev.getX(index);
                mLastMotionX = x;
                mActivePointerId = ev.getPointerId(index);
                break;
            }
            case MotionEvent.ACTION_POINTER_UP:
                onSecondaryPointerUp(ev);
                mLastMotionX = ev.getX(ev.findPointerIndex(mActivePointerId));
                break;
                
为了防止多只触摸作的一些处理，最后调用以下代码：
    
    if (needsInvalidate) {


            ViewCompat.postInvalidateOnAnimation(this);
        }
        
本质上还是调用view.invalidate()方法刷新页面。好了，对于ViewPage的滑动事件先介绍到这里。

