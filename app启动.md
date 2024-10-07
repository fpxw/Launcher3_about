## 桌面滑动相关过程分析
### 1.Launcher3源码中相关类如下：
'''
1. Launcher.java: launcher主要的activity，是launcher桌面第一次启动的activity.
2. Workspace.java: 抽象的桌面。由N个cellLayout组成,从cellLayout更高一级的层面上对事件的处理。
3. PagedView.java: 是workspace的父类，用来桌面的左右滑屏
4. CellLayout.java： 组成workspace的view,继承自viewgroup，既是一个dragSource，又是一个dropTarget,可以将它里面的item拖出去，也可以容纳拖动过来的item。在workspace_screen里面定了一些它的view参数。
5. 其他相关类 解读中
6. LauncherProvider.java: launcher的数据库，里面存储了桌面的item的信息。在创建数据库的时候会loadFavorites(db)方法，loadFavorites()会解析xml目录下的default_workspace.xml文件，把其中的内容读出来写到数据库中，这样就做到了桌面的预制

'''
### 2. 滑动原理
Launcher桌面的整体布局大致如下图
<img width="339" alt="launcher3桌面布局" src="https://github.com/user-attachments/assets/2dccd3fb-27b5-4659-ae07-b509e0fc3264">

#### 手指在Launcher上滑动分为三个阶段：手指按下，手指滑动，手指离开
#### 2.1 手指按下-->WorkSpace.java : onInterceptTouchEvent

WorkSpace.java 
'''java
    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        switch (ev.getAction() & MotionEvent.ACTION_MASK) {
            case MotionEvent.ACTION_DOWN:
                mXDown = ev.getX();
                mYDown = ev.getY();
                mTouchDownTime = System.currentTimeMillis();
                break;
            case MotionEvent.ACTION_POINTER_UP:
            case MotionEvent.ACTION_UP:
                if (mTouchState == TOUCH_STATE_REST) {
                    final CellLayout currentPage = (CellLayout) getChildAt(mCurrentPage);
                    if (currentPage != null) {
                        onWallpaperTap(ev);
                    }
                }
        }
        return super.onInterceptTouchEvent(ev);
    }
'''
PagedView.java 
'''java
    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        /*
         * This method JUST determines whether we want to intercept the motion.
         * If we return true, onTouchEvent will be called and we do the actual
         * scrolling there.
         */
        acquireVelocityTrackerAndAddMovement(ev);
        if (getChildCount() <= 0) return super.onInterceptTouchEvent(ev);
        /*
         * Shortcut the most recurring case: the user is in the dragging
         * state and he is moving his finger.  We want to intercept this
         * motion.
         */
        final int action = ev.getAction();
        if ((action == MotionEvent.ACTION_MOVE) && (mTouchState == TOUCH_STATE_SCROLLING)) {
            return true;
        }
        switch (action & MotionEvent.ACTION_MASK) {
            case MotionEvent.ACTION_MOVE: {
                /*
                 * mIsBeingDragged == false, otherwise the shortcut would have caught it. Check
                 * whether the user has moved far enough from his original down touch.
                 */
                if (mActivePointerId != INVALID_POINTER) {
                    determineScrollingStart(ev);
                }
                // if mActivePointerId is INVALID_POINTER, then we must have missed an ACTION_DOWN
                // event. in that case, treat the first occurence of a move event as a ACTION_DOWN
                // i.e. fall through to the next case (don't break)
                // (We sometimes miss ACTION_DOWN events in Workspace because it ignores all events
                // while it's small- this was causing a crash before we checked for INVALID_POINTER)
                break;
            }
            case MotionEvent.ACTION_DOWN: {
                final float x = ev.getX();
                final float y = ev.getY();
                // Remember location of down touch
                mDownMotionX = x;
                mDownMotionY = y;
                mDownScrollX = getScrollX();
                mLastMotionX = x;
                mLastMotionY = y;
                float[] p = mapPointFromViewToParent(this, x, y);
                mParentDownMotionX = p[0];
                mParentDownMotionY = p[1];
                mLastMotionXRemainder = 0;
                mTotalMotionX = 0;
                mActivePointerId = ev.getPointerId(0);
                /*
                 * If being flinged and user touches the screen, initiate drag;
                 * otherwise don't.  mScroller.isFinished should be false when
                 * being flinged.
                 */
                final int xDist = Math.abs(mScroller.getFinalX() - mScroller.getCurrX());
                final boolean finishedScrolling = (mScroller.isFinished() || xDist < mTouchSlop / 3);
                if (finishedScrolling) {
                    mTouchState = TOUCH_STATE_REST;
                    if (!mScroller.isFinished() && !mFreeScroll) {
                        setCurrentPage(getNextPage());
                        pageEndTransition();
                    }
                } else {
                    if (isTouchPointInViewportWithBuffer((int) mDownMotionX, (int) mDownMotionY)) {
                        mTouchState = TOUCH_STATE_SCROLLING;
                    } else {
                        mTouchState = TOUCH_STATE_REST;
                    }
                }
                break;
            }
            case MotionEvent.ACTION_UP:
            case MotionEvent.ACTION_CANCEL:
                resetTouchState();
                break;
            case MotionEvent.ACTION_POINTER_UP:
                onSecondaryPointerUp(ev);
                releaseVelocityTracker();
                break;
        }
        cancelPageMoving();    //ww+
        /*
         * The only time we want to intercept motion events is if we are in the
         * drag mode.
         */
        return mTouchState != TOUCH_STATE_REST;
    }
'''
CellLayout.java 
'''java
    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        if (mUseTouchHelper || (mInterceptTouchListener != null && mInterceptTouchListener.onTouch(this, ev))) {
            return true;
        }
        return false;
    }
    @Override
    public boolean onTouchEvent(MotionEvent ev) {
        boolean handled = super.onTouchEvent(ev);
        // Stylus button press on a home screen should not switch between overview mode and
        // the home screen mode, however, once in overview mode stylus button press should be
        // enabled to allow rearranging the different home screens. So check what mode
        // the workspace is in, and only perform stylus button presses while in overview mode.
        if (mLauncher.mWorkspace.isInOverviewMode() && mStylusEventHelper.onMotionEvent(ev)) {
            return true;
        }
        return handled;
    }
'''
当手指按下时，还没有准备滚动，此时mTouchState = TOUCH_STATE_REST, 前面几个方法都return 了 false
```mermaid
sequenceDiagram
  participant Alice
  participant Bob
  Alice->John: Hello John, how are you?
  loop Healthcheck
      John->John: Fight against hypochondria
  end
  Note right of John: Rational thoughts <br/>prevail...
  John-->Alice: Great!
  John->Bob: How about you?
  Bob-->John: Jolly good!
```















