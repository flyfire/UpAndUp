##View的invalidate和requestLayout有什么区别？

View的invalidate不会导致ViewRootImpl的invalidate被调用，而是递归调用父view的invalidateChildInParent，直到ViewRootImpl的invalidateChildInParent，然后触发peformTraversals，会导致当前view被重绘,由于mLayoutRequested为false，不会导致onMeasure和onLayout被调用，而OnDraw会被调用 。

requestLayout会直接递归调用父窗口的requestLayout，直到ViewRootImpl,然后触发peformTraversals，由于mLayoutRequested为true，会导致onMeasure和onLayout被调用。不一定会触发OnDraw 

##属性动画原理
ObjectAnimator.ofFloat()会将数据保存在KeyframeSet的关键帧里，setInterpolator，setDuration会保存TimeInterpolator和具体时间。ObjectAnimator的start会调用super.start(),也就是ValueAnimator的start，在ValueAnimator的start中将ValueAnimator作为AnimatorHandler的AnimationFrameCallback添加到ArrayList中，AnimationHandler中使用Choreographer VSYNC机制每隔一段时间回调ValueAnimator的doAnimationFrame函数，在doAnimation中，根据时间调用animationBasedOnTime，animationBasedOnTime中计算时间消耗的百分比，调用animateValue，ObjectAnimator重写了animateValue，在ObjectAnimator的animateValue中调用了父类也就是ValueAnimator的animateValue。

```
ObjectAnimator
    @CallSuper
    @Override
    void animateValue(float fraction) {
        final Object target = getTarget();
        if (mTarget != null && target == null) {
            // We lost the target reference, cancel and clean up. Note: we allow null target if the
            /// target has never been set.
            cancel();
            return;
        }

        super.animateValue(fraction);
        int numValues = mValues.length;
        for (int i = 0; i < numValues; ++i) {
            mValues[i].setAnimatedValue(target);
        }
    }
```

```
ValueAnimator 
    void animateValue(float fraction) {
        fraction = mInterpolator.getInterpolation(fraction);
        mCurrentFraction = fraction;
        int numValues = mValues.length;
        for (int i = 0; i < numValues; ++i) {
            mValues[i].calculateValue(fraction);
        }
        if (mUpdateListeners != null) {
            int numListeners = mUpdateListeners.size();
            for (int i = 0; i < numListeners; ++i) {
                mUpdateListeners.get(i).onAnimationUpdate(this);
            }
        }
    }
```

```
PropertyValuesHolder
    void calculateValue(float fraction) {
        Object value = mKeyframes.getValue(fraction);
        mAnimatedValue = mConverter == null ? value : mConverter.convert(value);
    }
```

```
KeyframeSet 
    public Object getValue(float fraction) {
        // Special-case optimization for the common case of only two keyframes
        if (mNumKeyframes == 2) {
            if (mInterpolator != null) {
                fraction = mInterpolator.getInterpolation(fraction);
            }
            return mEvaluator.evaluate(fraction, mFirstKeyframe.getValue(),
                    mLastKeyframe.getValue());
        }
        if (fraction <= 0f) {
            final Keyframe nextKeyframe = mKeyframes.get(1);
            final TimeInterpolator interpolator = nextKeyframe.getInterpolator();
            if (interpolator != null) {
                fraction = interpolator.getInterpolation(fraction);
            }
            final float prevFraction = mFirstKeyframe.getFraction();
            float intervalFraction = (fraction - prevFraction) /
                (nextKeyframe.getFraction() - prevFraction);
            return mEvaluator.evaluate(intervalFraction, mFirstKeyframe.getValue(),
                    nextKeyframe.getValue());
        } else if (fraction >= 1f) {
            final Keyframe prevKeyframe = mKeyframes.get(mNumKeyframes - 2);
            final TimeInterpolator interpolator = mLastKeyframe.getInterpolator();
            if (interpolator != null) {
                fraction = interpolator.getInterpolation(fraction);
            }
            final float prevFraction = prevKeyframe.getFraction();
            float intervalFraction = (fraction - prevFraction) /
                (mLastKeyframe.getFraction() - prevFraction);
            return mEvaluator.evaluate(intervalFraction, prevKeyframe.getValue(),
                    mLastKeyframe.getValue());
        }
        Keyframe prevKeyframe = mFirstKeyframe;
        for (int i = 1; i < mNumKeyframes; ++i) {
            Keyframe nextKeyframe = mKeyframes.get(i);
            if (fraction < nextKeyframe.getFraction()) {
                final TimeInterpolator interpolator = nextKeyframe.getInterpolator();
                final float prevFraction = prevKeyframe.getFraction();
                float intervalFraction = (fraction - prevFraction) /
                    (nextKeyframe.getFraction() - prevFraction);
                // Apply interpolator on the proportional duration.
                if (interpolator != null) {
                    intervalFraction = interpolator.getInterpolation(intervalFraction);
                }
                return mEvaluator.evaluate(intervalFraction, prevKeyframe.getValue(),
                        nextKeyframe.getValue());
            }
            prevKeyframe = nextKeyframe;
        }
        // shouldn't reach here
        return mLastKeyframe.getValue();
    }
```

```
PropertyValuesHolder 
    void setAnimatedValue(Object target) {
        if (mProperty != null) {
            mProperty.set(target, getAnimatedValue());
        }
        if (mSetter != null) {
            try {
                mTmpValueArray[0] = getAnimatedValue();
                mSetter.invoke(target, mTmpValueArray);
            } catch (InvocationTargetException e) {
                Log.e("PropertyValuesHolder", e.toString());
            } catch (IllegalAccessException e) {
                Log.e("PropertyValuesHolder", e.toString());
            }
        }
    }
```

mSetter是怎么来的？
ValueAnimator--》start 开启动画
首先调用子类start，后调用父类start方法
开启动画
ValueAnimator->startAnimation() 初始化动画
ObjectAnimtor->initAnimation()   （重点： 先调用子类方法 后调用父类）  1
ProteryValueHodler->setupSetterAndGetter  将成员变量mSetter 赋值  （注意调用顺序）  2
ValueAnimator->initAnimator()   （被子类调用）   3


##View绘制流程
在ActivityThread的handleResumeActivity调用了wm.addView，WindowManagerGlobal.addView(decorView,layoutparams),会实例化ViewRootImpl，decorView.assignParent(viewrootimpl)，然后调用requestLayout，将mLayoutRequested=true，调用scheduleTraversals,Choreographer将TraversalRunnable田间道callback中，收到vsync信号后会调用doTraversal,doTraversal会调用performTraversal，performTraversal中会依次调用performMeasure，performLayout，performDraw，performMeasure会调用decorView.measure进行子View的measure,递归调用，performLayout会调用decorView的layout方法，递归调用子View的layout方法，performDraw会调用view的draw方法，依次drawBackground,onDraw(绘制内容)，dispathDraw(绘制子View),onDrawForeground。

```
        /*
         * Draw traversal performs several drawing steps which must be executed
         * in the appropriate order:
         *
         *      1. Draw the background
         *      2. If necessary, save the canvas' layers to prepare for fading
         *      3. Draw view's content
         *      4. Draw children
         *      5. If necessary, draw the fading edges and restore layers
         *      6. Draw decorations (scrollbars for instance)
         */
```

##事件处理流程
从Activity的dispatchTouchEvent方法开始，调用decorView的dispatchTouchEvent

```
View 
dispatchTouchEvent

            //noinspection SimplifiableIfStatement
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }

            if (!result && onTouchEvent(event)) {
                result = true;
            }
```

View的onTouchEvent的ACTION_UP中执行了performClick,Action_DOWN时进行long click检查。

Android的触摸事件，是由windowManagerService进行采集，之后传递到Activiy进行处理。我们这里从Activity#dispatchTouchEvent方法开始解析

```
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();
    }
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    return onTouchEvent(ev);
}
```
上述代码中，onUserInteraction()是一个空的实现，我们直接来看下 getWindow().superDispatchTouchEvent(ev)方法。window是一个抽象的方法，不过系统给它提供了一个实现类PhoneWindow，我们这里看下它的superDispatchTouchEvent(ev)方法。

```
public boolean superDispatchTouchEvent(MotionEvent event) {
    return mDecor.superDispatchTouchEvent(event);
}
```
上述代码调用了DecorView类的superDispatchTouchEvent方法，继续跟进

```
public boolean superDispatchTouchEvent(MotionEvent event) {
    return super.dispatchTouchEvent(event);
}
```
上述代码调用了父类的dispatchTouchEvent方法，DecorView的父类为FrameLayout，其直接继承了ViewGroup#dispatchTouchEvent方法。

二、ViewGroup中的事件分发

ViewGroup#dispatchTouchEvent方法比较长，这里只截取部分进行分析

```
public boolean dispatchTouchEvent(MotionEvent ev) {
	
	...
		// 在ACTION_DOWN事件时，初始化Touch标记
		// Handle an initial down.
		if (actionMasked == MotionEvent.ACTION_DOWN) {
			// Throw away all previous state when starting a new touch gesture.
			// The framework may have dropped the up or cancel event for the previous gesture
			// due to an app switch, ANR, or some other state change.
			cancelAndClearTouchTargets(ev);
			resetTouchState();
		}

		// Check for interception.
		final boolean intercepted;
		if (actionMasked == MotionEvent.ACTION_DOWN
				|| mFirstTouchTarget != null) {
			// 是否拦截的标志位，假如设置requestDisallowInterceptTouchEvent(true)，
			// 则为true，不拦截事件
			final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
			if (!disallowIntercept) {
				// 默认返回false
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

		...

		// Check for cancelation.
		final boolean canceled = resetCancelNextUpFlag(this)
				|| actionMasked == MotionEvent.ACTION_CANCEL;

		// Update list of touch targets for pointer down, if needed.
		final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
		TouchTarget newTouchTarget = null;
		boolean alreadyDispatchedToNewTouchTarget = false;
		// 不是ACTION_CANCEL事件，并且不拦截事件
		if (!canceled && !intercepted) {

			// If the event is targeting accessiiblity focus we give it to the
			// view that has accessibility focus and if it does not handle it
			// we clear the flag and dispatch the event to all children as usual.
			// We are looking up the accessibility focused host to avoid keeping
			// state since these events are very rare.
			View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
					? findChildWithAccessibilityFocus() : null;

			if (actionMasked == MotionEvent.ACTION_DOWN
					|| (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
					|| actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
				final int actionIndex = ev.getActionIndex(); // always 0 for down
				final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
						: TouchTarget.ALL_POINTER_IDS;

				// Clean up earlier touch targets for this pointer id in case they
				// have become out of sync.
				removePointersFromTouchTargets(idBitsToAssign);

				final int childrenCount = mChildrenCount;
				if (newTouchTarget == null && childrenCount != 0) {
					// 获取触摸坐标
					final float x = ev.getX(actionIndex);
					final float y = ev.getY(actionIndex);
					// Find a child that can receive the event.
					// Scan children from front to back.
					final ArrayList<View> preorderedList = buildOrderedChildList();
					final boolean customOrder = preorderedList == null
							&& isChildrenDrawingOrderEnabled();
					final View[] children = mChildren;
					// 遍历所有子View
					for (int i = childrenCount - 1; i >= 0; i--) {
						final int childIndex = customOrder
								? getChildDrawingOrder(childrenCount, i) : i;
						final View child = (preorderedList == null)
								? children[childIndex] : preorderedList.get(childIndex);

						...
						
						resetCancelNextUpFlag(child);
						// 把事件(ACTION_DOWN、ACTION_POINTER_DOWN、ACTION_HOVER_MOVE)传递给子View处理
						if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
							// Child wants to receive touch within its bounds.
							mLastTouchDownTime = ev.getDownTime();
							if (preorderedList != null) {
								// childIndex points into presorted list, find original index
								for (int j = 0; j < childrenCount; j++) {
									if (children[childIndex] == mChildren[j]) {
										mLastTouchDownIndex = j;
										break;
									}
								}
							} else {
								mLastTouchDownIndex = childIndex;
							}
							mLastTouchDownX = ev.getX();
							mLastTouchDownY = ev.getY();
							newTouchTarget = addTouchTarget(child, idBitsToAssign);
							alreadyDispatchedToNewTouchTarget = true;
							break;
						}
						...
					}
					...
				}
				...
				}
			}
		}
		
		// 分发事件到目标View
		// Dispatch to touch targets.
		if (mFirstTouchTarget == null) {
			// 没有找到事件分发目标的情况，将会调用自己的onTouchEvent方法
			// No touch targets so treat this as an ordinary view.
			handled = dispatchTransformedTouchEvent(ev, canceled, null,
					TouchTarget.ALL_POINTER_IDS);
		} else {
			
			// Dispatch to touch targets, excluding the new touch target if we already
			// dispatched to it.  Cancel touch targets if necessary.
			TouchTarget predecessor = null;
			TouchTarget target = mFirstTouchTarget;
			// 这里找到了事件分发的目标
			while (target != null) {
				final TouchTarget next = target.next;
				// ACTION_DOWN已经完成事件分发，并消费了事件，直接返回true
				if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
					handled = true;
				} else {
					final boolean cancelChild = resetCancelNextUpFlag(target.child)
							|| intercepted;
					// 其余事件则需要传递给目标View进行处理
					if (dispatchTransformedTouchEvent(ev, cancelChild,
							target.child, target.pointerIdBits)) {
						handled = true;
					}
					if (cancelChild) {
						if (predecessor == null) {
							mFirstTouchTarget = next;
						} else {
							predecessor.next = next;
						}
						target.recycle();
						target = next;
						continue;
					}
				}
				predecessor = target;
				target = next;
			}
		}
		
		// 对ACTION_CANCEL事件进行处理
		// Update list of touch targets for pointer up or cancel, if needed.
		if (canceled
				|| actionMasked == MotionEvent.ACTION_UP
				|| actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
			// 重置Touch状态
			resetTouchState();
		} else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
			final int actionIndex = ev.getActionIndex();
			final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
			removePointersFromTouchTargets(idBitsToRemove);
		}
	}

	...
	return handled;
}
```

```
// 默认返回false
public boolean onInterceptTouchEvent(MotionEvent ev) {
    return false;
}
```
我们现在来看看传递事件的dispatchTransformedTouchEvent方法，同样我也只是截取了其中比较关键的部分

```
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
		View child, int desiredPointerIdBits) {
	final boolean handled;
	...
	final MotionEvent transformedEvent;
	// 对transformedEvent的一系列计算
	...
	if (child == null) {
		// 如果没有子View，则执行super.dispatchTouchEvent方法，
		// 调用自己的onTouchEvent方法
		handled = super.dispatchTouchEvent(transformedEvent);
	} else {
		final float offsetX = mScrollX - child.mLeft;
		final float offsetY = mScrollY - child.mTop;
		transformedEvent.offsetLocation(offsetX, offsetY);
		if (! child.hasIdentityMatrix()) {
			transformedEvent.transform(child.getInverseMatrix());
		}
		// 如果有子View，则调用子View#dispatchTouchEvent方法
		handled = child.dispatchTouchEvent(transformedEvent);
	}

	// Done.
	transformedEvent.recycle();
	return handled;
}
```
三、View中的事件处理

ViewGroup中不拦截事件，调用子View#dispatchTouchEvent方法进行处理

```
public boolean dispatchTouchEvent(MotionEvent event) {
	
	...
	if (onFilterTouchEventForSecurity(event)) {
		// 如果设置了OnTouchListener，使用onTouch对事件进行处理，
		// 并返回true，则不需要再执行onTouchEvent方法
		//noinspection SimplifiableIfStatement
		ListenerInfo li = mListenerInfo;
		if (li != null && li.mOnTouchListener != null
				&& (mViewFlags & ENABLED_MASK) == ENABLED
				&& li.mOnTouchListener.onTouch(this, event)) {
			result = true;
		}

		if (!result && onTouchEvent(event)) {
			result = true;
		}
	}
	...

	return result;
}
```
这里继续看看View#onTouchEvent方法

```
public boolean onTouchEvent(MotionEvent event) {
	final float x = event.getX();
	final float y = event.getY();
	final int viewFlags = mViewFlags;
	final int action = event.getAction();

	...

	if (((viewFlags & CLICKABLE) == CLICKABLE ||
			(viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
			(viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
		switch (action) {
			case MotionEvent.ACTION_UP:
				...
				// 移除长按
				removeLongPressCallback();
				...
					// 检查单击
					performClick();
				...
				break;

			case MotionEvent.ACTION_DOWN:
				...
				// 检测是否为长按
				checkForLongClick(0);
				...
				break;

			....
		}

		return true;
	}

	return false;
}
```
上述代码，主要是检查View是否可以点击，如果可点击，则会返回true，同时也会执行可点击的事件。

##LRU 

LruCache 源码解析

1. 简介

LRU 是 Least Recently Used 最近最少使用算法。
曾经，在各大缓存图片的框架没流行的时候。有一种很常用的内存缓存技术：SoftReference 和 WeakReference（软引用和弱引用）。但是走到了 Android 2.3（Level 9）时代，垃圾回收机制更倾向于回收 SoftReference 或 WeakReference 的对象。后来，又来到了 Android3.0，图片缓存在内容中，因为不知道要在是什么时候释放内存，没有策略，没用一种可以预见的场合去将其释放。这就造成了内存溢出。
2. 使用方法

当成一个 Map 用就可以了，只不过实现了 LRU 缓存策略。

使用的时候记住几点即可：

**1.（必填）**你需要提供一个缓存容量作为构造参数。
2.（必填） 覆写 sizeOf 方法 ，自定义设计一条数据放进来的容量计算，如果不覆写就无法预知数据的容量，不能保证缓存容量限定在最大容量以内。
3.（选填） 覆写 entryRemoved 方法 ，你可以知道最少使用的缓存被清除时的数据（ evicted, key, oldValue, newVaule ）。
**4.（记住）**LruCache是线程安全的，在内部的 get、put、remove 包括 trimToSize 都是安全的（因为都上锁了）。
5.（选填） 还有就是覆写 create 方法 。
一般做到 1、2、3、4就足够了，5可以无视 。

以下是 一个 LruCache 实现 Bitmap 小缓存的案例, entryRemoved 里的自定义逻辑可以无视，想看的可以去到我的我的展示 demo 里的看自定义 entryRemoved 逻辑。

private static final float ONE_MIB = 1024 * 1024;
// 7MB
private static final int CACHE_SIZE = (int) (7 * ONE_MIB);
private LruCache<String, Bitmap> bitmapCache;
this.bitmapCache = new LruCache<String, Bitmap>(CACHE_SIZE) {
    protected int sizeOf(String key, Bitmap value) {
        return value.getByteCount();
    }

    @Override
    protected void entryRemoved(boolean evicted, String key, Bitmap oldValue, Bitmap newValue) {
        ...
    }
};
3. 效果展示

LruCache 效果展示

4. 源码分析

4.1 LruCache 原理概要解析

LruCache 就是 利用 LinkedHashMap 的一个特性（ accessOrder＝true 基于访问顺序 ）再加上对 LinkedHashMap 的数据操作上锁实现的缓存策略。

LruCache 的数据缓存是内存中的。

1.首先设置了内部 LinkedHashMap 构造参数 accessOrder=true， 实现了数据排序按照访问顺序。

2.然后在每次 LruCache.get(K key) 方法里都会调用 LinkedHashMap.get(Object key)。

3.如上述设置了 accessOrder=true 后，每次 LinkedHashMap.get(Object key) 都会进行 LinkedHashMap.makeTail(LinkedEntry<K, V> e)。

4.LinkedHashMap 是双向循环链表，然后每次 LruCache.get -> LinkedHashMap.get 的数据就被放到最末尾了。

5.在 put 和 trimToSize 的方法执行下，如果发成数据量移除了，会优先移除掉最前面的数据（因为最新访问的数据在尾部）。

具体解析在： 4.2、4.3、4.4、4.5 。

4.2 LruCache 的唯一构造方法

/**
 * LruCache的构造方法：需要传入最大缓存个数
 */
public LruCache(int maxSize) {

    ...
    
    this.maxSize = maxSize;
    /*
     * 初始化LinkedHashMap
     * 第一个参数：initialCapacity，初始大小
     * 第二个参数：loadFactor，负载因子=0.75f
     * 第三个参数：accessOrder=true，基于访问顺序；accessOrder=false，基于插入顺序
     */
    this.map = new LinkedHashMap<K, V>(0, 0.75f, true);
}
第一个参数 initialCapacity 用于初始化该 LinkedHashMap 的大小。

先简单介绍一下 第二个参数 loadFactor，这个其实的 HashMap 里的构造参数，涉及到扩容问题，比如 HashMap 的最大容量是100，那么这里设置0.75f的话，到75容量的时候就会扩容。

主要是第三个参数 accessOrder=true ，这样的话 LinkedHashMap 数据排序就会基于数据的访问顺序，从而实现了 LruCache 核心工作原理。

4.3 LruCache.get(K key)

/**
 * 根据 key 查询缓存，如果存在于缓存或者被 create 方法创建了。
 * 如果值返回了，那么它将被移动到双向循环链表的的尾部。
 * 如果如果没有缓存的值，则返回 null。
 */
public final V get(K key) {
    
    ...  
      
    V mapValue;
    synchronized (this) {
        // 关键点：LinkedHashMap每次get都会基于访问顺序来重整数据顺序
        mapValue = map.get(key);
        // 计算 命中次数
        if (mapValue != null) {
            hitCount++;
            return mapValue;
        }
        // 计算 丢失次数
        missCount++;
    }

    /*
     * 官方解释：
     * 尝试创建一个值，这可能需要很长时间，并且Map可能在create()返回的值时有所不同。如果在create()执行的时
     * 候，一个冲突的值被添加到Map，我们在Map中删除这个值，释放被创造的值。
     */
    V createdValue = create(key);
    if (createdValue == null) {
        return null;
    }

    /***************************
     * 不覆写create方法走不到下面 *
     ***************************/

    /*
     * 正常情况走不到这里
     * 走到这里的话 说明 实现了自定义的 create(K key) 逻辑
     * 因为默认的 create(K key) 逻辑为null
     */
    synchronized (this) {
        // 记录 create 的次数
        createCount++;
        // 将自定义create创建的值，放入LinkedHashMap中，如果key已经存在，会返回 之前相同key 的值
        mapValue = map.put(key, createdValue);

        // 如果之前存在相同key的value，即有冲突。
        if (mapValue != null) {
            /*
             * 有冲突
             * 所以 撤销 刚才的 操作
             * 将 之前相同key 的值 重新放回去
             */
            map.put(key, mapValue);
        } else {
            // 拿到键值对，计算出在容量中的相对长度，然后加上
            size += safeSizeOf(key, createdValue);
        }
    }

    // 如果上面 判断出了 将要放入的值发生冲突
    if (mapValue != null) {
        /*
         * 刚才create的值被删除了，原来的 之前相同key 的值被重新添加回去了
         * 告诉 自定义 的 entryRemoved 方法
         */
        entryRemoved(false, key, createdValue, mapValue);
        return mapValue;
    } else {
        // 上面 进行了 size += 操作 所以这里要重整长度
        trimToSize(maxSize);
        return createdValue;
    }
}
上述的 get 方法表面并没有看出哪里有实现了 LRU 的缓存策略。主要在 mapValue = map.get(key);里，调用了 LinkedHashMap 的 get 方法，再加上 LruCache 构造里默认设置 LinkedHashMap 的 accessOrder=true。

4.4 LinkedHashMap.get(Object key)

/**
 * Returns the value of the mapping with the specified key.
 *
 * @param key
 *            the key.
 * @return the value of the mapping with the specified key, or {@code null}
 *         if no mapping for the specified key is found.
 */
@Override public V get(Object key) {
    /*
     * This method is overridden to eliminate the need for a polymorphic
     * invocation in superclass at the expense of code duplication.
     */
    if (key == null) {
        HashMapEntry<K, V> e = entryForNullKey;
        if (e == null)
            return null;
        if (accessOrder)
            makeTail((LinkedEntry<K, V>) e);
        return e.value;
    }

    int hash = Collections.secondaryHash(key);
    HashMapEntry<K, V>[] tab = table;
    for (HashMapEntry<K, V> e = tab[hash & (tab.length - 1)];
         e != null; e = e.next) {
        K eKey = e.key;
        if (eKey == key || (e.hash == hash && key.equals(eKey))) {
            if (accessOrder)
                makeTail((LinkedEntry<K, V>) e);
            return e.value;
        }
    }
    return null;
}
其实仔细看 if (accessOrder) 的逻辑即可，如果 accessOrder=true 那么每次 get 都会执行 N 次 makeTail(LinkedEntry<K, V> e) 。

接下来看看：

4.5 LinkedHashMap.makeTail(LinkedEntry<K, V> e)

/**
 * Relinks the given entry to the tail of the list. Under access ordering,
 * this method is invoked whenever the value of a  pre-existing entry is
 * read by Map.get or modified by Map.put.
 */
private void makeTail(LinkedEntry<K, V> e) {
    // Unlink e
    e.prv.nxt = e.nxt;
    e.nxt.prv = e.prv;

    // Relink e as tail
    LinkedEntry<K, V> header = this.header;
    LinkedEntry<K, V> oldTail = header.prv;
    e.nxt = header;
    e.prv = oldTail;
    oldTail.nxt = header.prv = e;
    modCount++;
}
// Unlink e


// Relink e as tail


LinkedHashMap 是双向循环链表，然后此次 LruCache.get -> LinkedHashMap.get 的数据就被放到最末尾了。

以上就是 LruCache 核心工作原理。

接下来介绍 LruCache 的容量溢出策略。

4.6 LruCache.put(K key, V value)

public final V put(K key, V value) {
    ...
    synchronized (this) {
        ...
        // 拿到键值对，计算出在容量中的相对长度，然后加上
        size += safeSizeOf(key, value);
        ...
    }
	...
    trimToSize(maxSize);
    return previous;
}
记住几点：

**1.**put 开始的时候确实是把值放入 LinkedHashMap 了，不管超不超过你设定的缓存容量。
**2.**然后根据 safeSizeOf 方法计算 此次添加数据的容量是多少，并且加到 size 里 。
**3.**说到 safeSizeOf 就要讲到 sizeOf(K key, V value) 会计算出此次添加数据的大小 。
**4.**直到 put 要结束时，进行了 trimToSize 才判断 size 是否 大于 maxSize 然后进行最近很少访问数据的移除。
4.7 LruCache.trimToSize(int maxSize)

public void trimToSize(int maxSize) {
    /*
     * 这是一个死循环，
     * 1.只有 扩容 的情况下能立即跳出
     * 2.非扩容的情况下，map的数据会一个一个删除，直到map里没有值了，就会跳出
     */
    while (true) {
        K key;
        V value;
        synchronized (this) {
            // 在重新调整容量大小前，本身容量就为空的话，会出异常的。
            if (size < 0 || (map.isEmpty() && size != 0)) {
                throw new IllegalStateException(
                        getClass().getName() + ".sizeOf() is reporting inconsistent results!");
            }
            // 如果是 扩容 或者 map为空了，就会中断，因为扩容不会涉及到丢弃数据的情况
            if (size <= maxSize || map.isEmpty()) {
                break;
            }

            Map.Entry<K, V> toEvict = map.entrySet().iterator().next();
            key = toEvict.getKey();
            value = toEvict.getValue();
            map.remove(key);
            // 拿到键值对，计算出在容量中的相对长度，然后减去。
            size -= safeSizeOf(key, value);
            // 添加一次收回次数
            evictionCount++;
        }
        /*
         * 将最后一次删除的最少访问数据回调出去
         */
        entryRemoved(true, key, value, null);
    }
}
简单描述：会判断之前 size 是否大于 maxSize 。是的话，直接跳出后什么也不做。不是的话，证明已经溢出容量了。由 makeTail 图已知，最近经常访问的数据在最末尾。拿到一个存放 key 的 Set，然后一直一直从头开始删除，删一个判断是否溢出，直到没有溢出。

最后看看：

4.8 覆写 entryRemoved 的作用

entryRemoved被LruCache调用的场景：

1.（put） put 发生 key 冲突时被调用，evicted=false，key=此次 put 的 key，oldValue=被覆盖的冲突 value，newValue=此次 put 的 value。
2.（trimToSize） trimToSize 的时候，只会被调用一次，就是最后一次被删除的最少访问数据带回来。evicted=true，key=最后一次被删除的 key，oldValue=最后一次被删除的 value，newValue=null（此次没有冲突，只是 remove）。
3.（remove） remove的时候，存在对应 key，并且被成功删除后被调用。evicted=false，key=此次 put的 key，oldValue=此次删除的 value，newValue=null（此次没有冲突，只是 remove）。
4.（get后半段，查询丢失后处理情景，不过建议忽略） get 的时候，正常的话不实现自定义 create 的话，代码上看 get 方法只会走一半，如果你实现了自定义的 create(K key) 方法，并且在 你 create 后的值放入 LruCache 中发生 key 冲突时被调用，evicted=false，key=此次 get 的 key，oldValue=被你自定义 create(key)后的 value，newValue=原本存在 map 里的 key-value。
解释一下第四点吧：**<1>.第四点是这样的，先 get(key)，然后没拿到，丢失。<2>.如果你提供了 自定义的 create(key) 方法，那么 LruCache 会根据你的逻辑自造一个 value，但是当放入的时候发现冲突了，但是已经放入了。<3>.**此时，会将那个冲突的值再让回去覆盖，此时调用上述4.的 entryRemoved。

因为 HashMap 在数据量大情况下，拿数据可能造成丢失，导致前半段查不到，你自定义的 create(key) 放入的时候发现又查到了**（有冲突）**。然后又急忙把原来的值放回去，此时你就白白create一趟，无所作为，还要走一遍entryRemoved。

综上就如同注释写的一样：

/**
 * 1.当被回收或者删掉时调用。该方法当value被回收释放存储空间时被remove调用
 * 或者替换条目值时put调用，默认实现什么都没做。
 * 2.该方法没用同步调用，如果其他线程访问缓存时，该方法也会执行。
 * 3.evicted=true：如果该条目被删除空间 （表示 进行了trimToSize or remove）  evicted=false：put冲突后 或 get里成功create后
 * 导致
 * 4.newValue!=null，那么则被put()或get()调用。
 */
protected void entryRemoved(boolean evicted, K key, V oldValue, V newValue) {
}
可以参考我的 demo 里的 entryRemoved 。

4.9 LruCache 局部同步锁

在 get, put, trimToSize, remove 四个方法里的 entryRemoved 方法都不在同步块里。因为 entryRemoved 回调的参数都属于方法域参数，不会线程不安全。

本地方法栈和程序计数器是线程隔离的数据区

LruCache重要的几点：

1.LruCache 是通过 LinkedHashMap 构造方法的第三个参数的 accessOrder=true 实现了 LinkedHashMap 的数据排序基于访问顺序 （最近访问的数据会在链表尾部），在容量溢出的时候，将链表头部的数据移除。从而，实现了 LRU 数据缓存机制。

**2.**LruCache 在内部的get、put、remove包括 trimToSize 都是安全的（因为都上锁了）。

**3.**LruCache 自身并没有释放内存，将 LinkedHashMap 的数据移除了，如果数据还在别的地方被引用了，还是有泄漏问题，还需要手动释放内存。

**4.**覆写 entryRemoved 方法能知道 LruCache 数据移除是是否发生了冲突，也可以去手动释放资源。

5.maxSize 和 sizeOf(K key, V value) 方法的覆写息息相关，必须相同单位。（ 比如 maxSize 是7MB，自定义的 sizeOf 计算每个数据大小的时候必须能算出与MB之间有联系的单位 ）

##ListView源码分析

ListView 和 GridView 都继承自 AbsListView ，AbsListView 继承自 AdapterView，再往上就是 ViewGroup 和 View 了

在使用 ListView 时，我们需要 Adpater 来适配 ListView 的工作，这样用到了适配器模式，ListView 只负责显示数据和处理交互， 至于显示什么数据，这个工作就交给了 Adapter，这样在 ListView 需要数据时就跟 Adapter 来要，减轻了 ListView 的工作量，并为 ListView 可以展示不同的数据提供了实现。

我们学习过 ListView 的时候，都知道 ListView 不但以列表形式展示，并且 item 还能实现复用，这样即使显示上千条数据，内存中 ListView 的 item 的 View 对象也就那么几个，至于 ListView 的 item 复用是如何实现的，今天的解析就是为了搞清楚这个。

RecycleBin

首先我们看一个 AbsListView 的内部类，RecycleBin ，这个类在 LisView 工作中起到了关键性作用。主要解析 RecycleBin 的 5 个方法

RecycleBin 中的代码很多，我只选了重要的 5 个方法，这五个方法都加了注释，可以先稍微了解一下，接下来分析 ListView 的时候再回头看会更加清晰。

```
class AbListView {

class RecycleBin {

/**
* 在 ListView 的 item 只有一种布局时，存储废弃 View 的集合
/
private ArrayList<View> mCurrentScrap;
/*
* 在 ListViwe 有多种布局时，这个数组中的每一项都是一种布局的集合
*/
private ArrayList<View>[] mScrapViews;

/**
* 当前 ListView 中显示的 View
*/
private View[] mActiveViews = new View[0];

/**
* 将 ListView 中的 View 都放入 mActivieViews 数组中
* childCount 要存入的 View 的数量
* firstActivePosition 第一个可见的 View 的 position
*/
void fillActiveViews(int childCount, int firstActivePosition) {

if (mActiveViews.length < childCount) {
mActiveViews = new View[childCount]; 
}

mFirstActivePosition = firstActivePosition;

final View[] activeViews = mActiveViews;
for (int i = 0; i < childCount; i++) {// 遍历 AbListView 中的子 View ，把每一个都放入 mActiveViews 中
View child = getChildAt(i); 
AbsListView.LayoutParams lp = (AbsListView.LayoutParams) child.getLayoutParams();

if (lp != null && lp.viewType != ITEM_VIEW_TYPE_HEADER_OR_FOOTER) {

activeViews[i] = child;
// Remember the position so that setupChild() doesn't reset state.
lp.scrappedFromPosition = firstActivePosition + i;
}
}
}
```

/**
* 与 fillActiveViews 相对，getActiveView 方法作用为从 mActiveViews 中取出相应位置的 View，并移除 mActiveViews 中 position 位置的 View
* 说明 mActiveViews 中每个位置的 View 只能使用一次，除非重新赋值
*/
```
View getActiveView(int position) {
int index = position - mFirstActivePosition;
final View[] activeViews = mActiveViews;
if (index >=0 && index < activeViews.length) {
final View match = activeViews[index];
activeViews[index] = null;
return match;
}
return null;
}
```
/**
* 根据 View 在 AbListView 中的 Type ，存储废弃的 View，例如滚动出屏幕的 View
*/

```
void addScrapView(View scrap, int position) {
...
final AbsListView.LayoutParams lp = (AbsListView.LayoutParams) scrap.getLayoutParams();
final int viewType = lp.viewType; // 获取 Type 的值 
...
if (mViewTypeCount == 1) {
mCurrentScrap.add(scrap); // 如果只有一种则添加到 mCurrentScrap 中
} else {
mScrapViews[viewType].add(scrap); // 如果有多种，则放入对应的集合中
}
...
}
```
/**
* @return A view from the ScrapViews collection. 找到废弃 View 的数组中 itemId 跟 position 处 itemId 相同的 View 从数组中移除，并将 View 返回
*/

```
View getScrapView(int position) {
final int whichScrap = mAdapter.getItemViewType(position); // 需要获取的 Viwe 的 Type
if (whichScrap < 0) {
return null;
}
if (mViewTypeCount == 1) {
return retrieveFromScrap(mCurrentScrap, position); // 只有一种时，返回 mCurrentScrap 中对应位置 View 
} else if (whichScrap < mScrapViews.length) {
return retrieveFromScrap(mScrapViews[whichScrap], position); // 多种情况是，返回对应种类集合中的 View
}
return null;
}
```

/**
* 设置 ListView 中 item 的种类，并初始化存储废弃 View 的集合
*/

```
public void setViewTypeCount(int viewTypeCount) {
if (viewTypeCount < 1) {
throw new IllegalArgumentException("Can't have a viewTypeCount < 1");
}
//noinspection unchecked
ArrayList<View>[] scrapViews = new ArrayList[viewTypeCount];
for (int i = 0; i < viewTypeCount; i++) {
scrapViews[i] = new ArrayList<View>();
}
mViewTypeCount = viewTypeCount;
mCurrentScrap = scrapViews[0];
mScrapViews = scrapViews;
}
}
}
```

ListView 的工作流程

ListView 在显示时，也是通过 onMeasure，onLayout，onDraw 的流程，onMeasure 过程很简单，因为 ListView 一般都是固定宽高或者宽高最大；onDraw 更简单，绘制的过程都是每个子 View 自己绘制自己，ListViwe 不用管绘制阶段；只有 onLayout 过程是有实际工作意义的，所以我们直接来看 onLayout 方法，onLayout 方法的实现是在 AbListView 中的，onLayout 方法中主要就是调用 layoutChildren 来布局子 View,这个方法就切换到 ListView 中了

```
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
super.onLayout(changed, l, t, r, b);

...

layoutChildren(); // 布局 子 view

...
}
```

```
@Override
protected void layoutChildren() {
...
final int childCount = getChildCount();
if (dataChanged) { // 判断 数据集 是否变化
for (int i = 0; i < childCount; i++) {
recycleBin.addScrapView(getChildAt(i), firstPosition+i);
}
} else { // ListView 刚进来时，进去当然是没变化的，所以执行这里，recycleBin 就是刚开始提到的 RecycleBin 的实例，fillActiveViews 将子 View 添加到 RcycleBin 的响应集合里，由于第一次进来是没有子 View 的，所以这行代码无效果
recycleBin.fillActiveViews(childCount, firstPosition);
}

...
switch (mLayoutMode) { // 布局模式，一般都是默认
default:
if (childCount == 0) { // 第一次 onLayout 时 子 View 个数为 0
if (!mStackFromBottom) { // 从上往下加载
final int position = lookForSelectablePosition(0, true);
setSelectedPositionInt(position);
sel = fillFromTop(childrenTop);
} else { // 从下往上加载
final int position = lookForSelectablePosition(mItemCount - 1, false);
setSelectedPositionInt(position);
sel = fillUp(mItemCount - 1, childrenBottom);
}
} else {
...
}
break;

}
}
```

我们来看 layoutChildre 方法，数据没有变化时会调用 recycleBin.fillActiveViews 的方法，这是子 View 的个数为 0 ，所以这行代码无效，接着到了根据布局模式填充数据的阶段，一般都是默认模式。接下来判断从上往下填充活着从下往上填充 ListView，这个是由 ListView 使用者设置的，默认从上往下。

```
// 从上往下填充
private View fillFromTop(int nextTop) {
// 检查第一个位置的有效性
mFirstPosition = Math.min(mFirstPosition, mSelectedPosition);
mFirstPosition = Math.min(mFirstPosition, mItemCount - 1);
if (mFirstPosition < 0) {
mFirstPosition = 0;
}
return fillDown(mFirstPosition, nextTop); // 填充数据
}

private View fillDown(int pos, int nextTop) {
View selectedView = null;

...

// 循环填充数据
while (nextTop < end && pos < mItemCount) { // 下一个 Viwe 的 top 位置还不到 listView 的底部，并且当前 position 小于数据的数量时继续填充
// is this the selected item?
boolean selected = pos == mSelectedPosition;

// 由 位置，需要添加的 Viwe 的顶部在 ListView 中的位置，padding 值的构造子 View 并添加到 ListView 中
View child = makeAndAddView(pos, nextTop, true, mListPadding.left, selected);

nextTop = child.getBottom() + mDividerHeight; // 计算下一个子 View 的顶部位置
if (selected) {
selectedView = child;
}
pos++;
}

setVisibleRangeHint(mFirstPosition, mFirstPosition + getChildCount() - 1);
return selectedView;
}
```

```
// 从下往上填充的情况
private View fillUp(int pos, int nextBottom) {
View selectedView = null;

int end = 0;
if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
end = mListPadding.top;
}

// 循环填充
while (nextBottom > end && pos >= 0) { // 从下往上填充，判断的是下一个 View 的底部是否已经出了屏幕位置，如果出了屏幕则不用添加
// is this the selected item?
boolean selected = pos == mSelectedPosition;
View child = makeAndAddView(pos, nextBottom, false, mListPadding.left, selected); // 构造 View 并填充到 ListView
nextBottom = child.getTop() - mDividerHeight; // 下一个 Viwe 的底部位置
if (selected) {
selectedView = child;
}
pos--;
}

mFirstPosition = pos + 1;
setVisibleRangeHint(mFirstPosition, mFirstPosition + getChildCount() - 1);
return selectedView;
}
```

由代码我们看到不管是从下往上填充还是从上往下填充，都会调用一个 makeAndAddView 方法来构造 View 并将其填充到 ListView 中

```
// 构造 View 并填充 ListView
private View makeAndAddView(int position, int y, boolean flow, int childrenLeft,
boolean selected) {

if (!mDataChanged) { // 数据集没变化时
final View activeView = mRecycler.getActiveView(position); // 从 RecycleBin 中获取当前位置的 View ，由于第一次加载界面，该 View 为空
if (activeView != null) {
setupChild(activeView, position, y, flow, childrenLeft, selected, true); 
return activeView;
}
}

final View child = obtainView(position, mIsScrap); // 在 RecycleBin 中没有需要的 View 时，调用 obtainView 方法来构造 View
// This needs to be positioned and measured.
setupChild(child, position, y, flow, childrenLeft, selected, mIsScrap[0]); // 将 View 填充到 ListView

return child;
}
```

第一次加载时在 RecycleBin 中不能获取到对象 view, 则通过 obtainView 方法来构造 Viwe，该方法在 AbListView 中.

```
View obtainView(int position, boolean[] outMetadata) {

...

final View scrapView = mRecycler.getScrapView(position); // 从 RecycleBin 中的废弃 View 中取出需要的 View ，由于是第一次加载，返回为空

final View child = mAdapter.getView(position, scrapView, this); // 由 Adapter 的 getView 方法来获取 View

if (scrapView != null) {
if (child != scrapView) {
mRecycler.addScrapView(scrapView, position);
} else if (child.isTemporarilyDetached()) {
...
}
}

setItemViewLayoutParams(child, position);

return child; // 返回 Viwe
}
```

obtainView 方法中会首先从 RecycleBin 中的废弃 View 的集合中来获取 View，第一次加载时是获取不到的，所以同 mAdapter.getView 方法来获取 View，getView 方法就是我们定义 Adapter 时重写的那个方法。在 scrapView 为空时，我们会通过 LayoutInflater 来加载一个 View 赋值给 child，最后将child 返回。makeAndAddView 方法中调用 obtainView 获取到 View 之后，在调用 setupChild 方法，其中调用 attachViewToParent 将 Viwe 填充到 ListView 中。加载时的第一次 onLayout 过程结束

由于 Android 界面加载机制，onLayout 方法会调用两次，第一次 onLayout 时我们上面已经分析过了，如果再加载一次，难道数据会添加两次？ 我们接着回头看第二次 onLayout 的过程。其中 onLayout 还是会调用 layoutChildren 方法，不过执行的逻辑确跟第一次不同了

```
@Override
protected void layoutChildren() {
...
final int childCount = getChildCount();
if (dataChanged) { // 判断 数据集 是否变化
for (int i = 0; i < childCount; i++) {
recycleBin.addScrapView(getChildAt(i), firstPosition+i);
}
} else { // 这已经是第二季加载，ListView 中已经是有数据了，这时候将 ListView 中的所有 View 都添加到 recycleBind 中的存储 ListView 子 View 的集合中
recycleBin.fillActiveViews(childCount, firstPosition);
}

...

detachAllViewsFromParent(); // 移除 ListView 中所有的 Viwe

...
switch (mLayoutMode) { // 布局模式，一般都是默认
default:
if (childCount == 0) { // 第一次 onLayout 时 子 View 个数为 0 时的加载
...
} else { // 第二次加载时 childCount 已经不是 0
if (mSelectedPosition >= 0 && mSelectedPosition < mItemCount) {
sel = fillSpecific(mSelectedPosition,
oldSel == null ? childrenTop : oldSel.getTop());
} else if (mFirstPosition < mItemCount) {
sel = fillSpecific(mFirstPosition,
oldFirst == null ? childrenTop : oldFirst.getTop());
} else {
sel = fillSpecific(0, childrenTop);
}
}
break;

}
}
```

第二次 onLayout 时，layoutChildren 方法的执行已经跟第一次不一样，这是会将 ListView 中所有的 View 添加到 RecycleBin 中，然后移除 ListView 中所有的 View，再调用 fillSpecific 方法来填充 ListView 。

```
private View fillSpecific(int position, int top) {
boolean tempIsSelected = position == mSelectedPosition;
View temp = makeAndAddView(position, top, true, mListPadding.left, tempIsSelected);
// Possibly changed again in fillUp if we add rows above this one.
mFirstPosition = position;

View above;
View below;

final int dividerHeight = mDividerHeight;
if (!mStackFromBottom) { // 从上往下加载
above = fillUp(position - 1, temp.getTop() - dividerHeight);
// This will correct for the top of the first view not touching the top of the list
adjustViewsUpOrDown();
below = fillDown(position + 1, temp.getBottom() + dividerHeight);
int childCount = getChildCount();
if (childCount > 0) {
correctTooHigh(childCount);
}
} else {// 从下往加载
below = fillDown(position + 1, temp.getBottom() + dividerHeight);
// This will correct for the bottom of the last view not touching the bottom of the list
adjustViewsUpOrDown();
above = fillUp(position - 1, temp.getTop() - dividerHeight);
int childCount = getChildCount();
if (childCount > 0) {
correctTooLow(childCount);
}
}

...
}
```

fillSpecific 方法里面还是会根据 ListView 的填充模式从下往上或者从上往下来填充，不过由于之前已经加载过，被选中的 position 的值可能不为 0 ，这时候会从被选中的位置向两头加载。同样是调用 fillUp 和 fillDown 方法，这两个方法的执行跟第一次 onLayout 没什么区别，都是根据当前视图中最后一个 View 的位置和数据的数量来判断是否需要继续添加 View，不过其内部构造 View 时调用的 makeAndAddView 确有了一点不同。由于 从 RecycleBin 中取到了数据，就不需要调用 obtainView 方法来获取 View ，直接将 RecycleBin 中取到的 view 填充到 ListView 中，通过 setupChild 方法中的 attachViewToParent 方法完成填充。

```
private View makeAndAddView(int position, int y, boolean flow, int childrenLeft,
boolean selected) {
if (!mDataChanged) {

final View activeView = mRecycler.getActiveView(position); // 由于第二次 onLayout 开始就把 ListView 中原来的 view 添加到 RecycleBin 中了，所有 activeView 有值

if (activeView != null) {
// Found it. We're reusing an existing child, so it just needs
// to be positioned like a scrap view.
setupChild(activeView, position, y, flow, childrenLeft, selected, true); // 将从 RecycleBin 中取到的值填充到 ListView 中
return activeView; // 返回 RecycleBin 中得到的 View
}
}

// Make a new view for this position, or convert an unused view if
// possible.
final View child = obtainView(position, mIsScrap);

// This needs to be positioned and measured.
setupChild(child, position, y, flow, childrenLeft, selected, mIsScrap[0]);

return child;
}
```

这样第一次 onLayout 和第二次 onLayout 方法就调用完成了，ListView 显示到界面上时也就可以展示数据了。只有在第一次加载时才从 Adapter 中获取View，第二次加载时首先为 RecycleBin 添加数据，ListView 再需要数据时直接从 RecycleBin 中获取数据即可。

ListView 滑动过程中的视图填充

在 ListView 滑动过程中 ListView 中的视图是会随时变化的，所以我们必须要分析滑动过程中 ListView 的工作过程，滑动中处理主要在 onTouchEvent 方法中。由于滑动过程中的触摸事件是 EVENT_MOVE 所以我们只看移动过程中的代码，接着追踪到了 onTouchMove 方法中

```
@Override
public boolean onTouchEvent(MotionEvent ev) {

initVelocityTrackerIfNotExists();
final MotionEvent vtev = MotionEvent.obtain(ev);
final int actionMasked = ev.getActionMasked();
...
switch (actionMasked) {
...
case MotionEvent.ACTION_MOVE: {
onTouchMove(ev, vtev);
break;
}
...
}

if (mVelocityTracker != null) {
mVelocityTracker.addMovement(vtev);
}
vtev.recycle();
return true;
}
```

ListView 的滑动过程中，在 onTouchMove 中对应的情况为 TOUCH_MODE_SCROLL ，接着追踪到了 scrollFNeeded 方法中，该方法中通过调用 trackMotionScroll 方法来完成滑动过程中的填充任务，trackMotionScroll 方法的两个参数为 deltaY 为从 ACTION_DOWN 到当前位置移动的距离，incrementalDelaY 当前距离上一个事件时发送的距离变化

```
boolean trackMotionScroll(int deltaY, int incrementalDeltaY) {
...

// 如果 incrementalDeltaY 大于 0 ，说明手指下滑，屏幕中 ListView 向上滚动，向下拉状态，要显示的 View 从顶部出现

// 如果 incrementalDeltaY 小于 0 ，说明手指上滑，屏幕中 ListView 向下滚动，向上拉状态，要显示的 View 从底部出现

final boolean down = incrementalDeltaY < 0;

int start = 0;
int count = 0;

if (down) { // 上拉状态，View 从底部填充，同时 View 从顶部滑出 ListView 的区域
int top = -incrementalDeltaY; // 手指滑动的长度
if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
top += listPadding.top; // 手指滑动的长度加上 ListViwe 的顶部的 padding 值
}
for (int i = 0; i < childCount; i++) {
final View child = getChildAt(i);
if (child.getBottom() >= top) { // View 的底部加上手指滑动的距离如果比 top 值大，说明该 View 还有部分或整体都在 ListView 中，不能回收
break;
} else { // 如果 View 的底部的值加滑动的距离比 top 值小，说明该 View 经过滑动之后会滑出 ListView 的区域，需要回收
count++; // 需要回收的数量加 1 
int position = firstPosition + i;
if (position >= headerViewsCount && position < footerViewsStart) {
...
mRecycler.addScrapView(child, position); // 将需要回收的 view 放入 RecycleBin 中的回收集合中
}
}
}
} else { // 下拉状态，数据从顶部填充，同时 View 从底部移出
int bottom = getHeight() - incrementalDeltaY;
if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
bottom -= listPadding.bottom;
}
for (int i = childCount - 1; i >= 0; i--) {
final View child = getChildAt(i);
if (child.getTop() <= bottom) {
break;
} else {
start = i;
count++;
int position = firstPosition + i;
if (position >= headerViewsCount && position < footerViewsStart) {
// The view will be rebound to new data, clear any
// system-managed transient state.
child.clearAccessibilityFocus();
mRecycler.addScrapView(child, position);
}
}
}
}

if (count > 0) {
detachViewsFromParent(start, count); // 需要移除的 view 移出 ListView
mRecycler.removeSkippedScrap();
}

offsetChildrenTopAndBottom(incrementalDeltaY); // 将还存在的 view 根据滑动距离做出偏移

// 判断原来最顶部的 View 的在 ListView 中的高度小于滑动的高度，或者最底部的 View 在 ListView 中的高度小于滑动的高度，这时候必然有 View 滑出屏幕
if (spaceAbove < absIncrementalDeltaY || spaceBelow < absIncrementalDeltaY) {
fillGap(down); // 有 View 滑出屏幕是需要新的 View 填充屏幕
}

}
```

trackMotionScroll 方法中，就根据滑动的距离，筛选出经过滑动之后移除屏幕的 View 并添加到 RecycleBin 中存储废弃 View 的集合中，接着会调用 detachViewFromParent 将需要移除的 view 移出 ListView，还会将仍在布局中的 View 根据滑动距离做出移动，最后如果有 View 移出屏幕，则通过 fillGap 方法充新的 View

```
void fillGap(boolean down) {
final int count = getChildCount();
if (down) { // 向上拉，从底部填充 view
int paddingTop = 0;
if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
paddingTop = getListPaddingTop();
}
final int startOffset = count > 0 ? getChildAt(count - 1).getBottom() + mDividerHeight :
paddingTop;
fillDown(mFirstPosition + count, startOffset);
correctTooHigh(getChildCount());
} else { // 向下拉，从顶部填充 view
int paddingBottom = 0;
if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
paddingBottom = getListPaddingBottom();
}
final int startOffset = count > 0 ? getChildAt(0).getTop() - mDividerHeight :
getHeight() - paddingBottom;
fillUp(mFirstPosition - 1, startOffset);
correctTooLow(getChildCount());
}
}
```
又到了，fillDown 和 fillUp 方法，还记得吗，上面 onLayout 过程中填充 ListView 的方法，其中会调用 makeAndAddView 方法来获取 View

```
private View makeAndAddView(int position, int y, boolean flow, int childrenLeft,
boolean selected) {
if (!mDataChanged) {
// Try to use an existing view for this position.
final View activeView = mRecycler.getActiveView(position); // 由于在 ListView 的第二次 onLayout 过程中将 RecycleBin.getActiveView 中存储的 View 已经全部获取，所以这里会取得 null
if (activeView != null) {
// Found it. We're reusing an existing child, so it just needs
// to be positioned like a scrap view.
setupChild(activeView, position, y, flow, childrenLeft, selected, true);
return activeView;
}
}

// Make a new view for this position, or convert an unused view if
// possible.
final View child = obtainView(position, mIsScrap); // 从而接着从 obtainView 方法中获取 View

// This needs to be positioned and measured.
setupChild(child, position, y, flow, childrenLeft, selected, mIsScrap[0]);

return child;
}
```
这次 obtainView 中的执行跟第一次 onLayout 时就不一样了，从 RecycleBin 中的废弃 View 中取出需要的 View ，由于滑出屏幕的 View 已经添加到RecycleBin 中，所以这里不为 null，在 Adapter.getView 时就可以重用被回收的 View ，实现了 ListView 虽然有千百条数据，但是真正的 item View 确只有很少的几个。

```
View obtainView(int position, boolean[] outMetadata) {

...

final View scrapView = mRecycler.getScrapView(position); // 从 RecycleBin 中的废弃 View 中取出需要的 View ，由于滑出屏幕的 View 已经添加到了 RecycleBin 中，所以这里不为 null

final View child = mAdapter.getView(position, scrapView, this); // 由 Adapter 的 getView 方法来获取 View，将 scrapView 传入该方法，也就是 AdatperView 的 getView 的 convertView 是有值的，这时我们不会重新构造 View，而是复用 convertView ，这样就实现了 ListView 滑动是的 View 重用

if (scrapView != null) {
if (child != scrapView) {
mRecycler.addScrapView(scrapView, position);
} else if (child.isTemporarilyDetached()) {
...
}
}

setItemViewLayoutParams(child, position);

return child; // 返回 Viwe
}
```
到这里，整个 ListView 源码的分析就要结束了，主要的内容是两次 onLayout 过程中的处理，和 ListView 在滑动过程中 View 的回收和复用的过程。

在 ListView 滑动之后，还会调用 invalidate 方法来申请重新绘制，这个过程又会调用依次 onLayout 这时的 onLayout 执行过程就如果显示到屏幕上时第二次调用 onLayout 的过程，会将 View 添加到 RecycleBin ，移除所有 View ，再从 RecycleBin 中获取 View 添加到 ListView 中。整个过程执行结束。

##Android Parcelable和Serializable的区别

1、作用

Serializable的作用是为了保存对象的属性到本地文件、数据库、网络流、rmi以方便数据传输，当然这种传输可以是程序内的也可以是两个程序间的。而Android的Parcelable的设计初衷是因为Serializable效率过慢，为了在程序内不同组件间以及不同Android程序间(AIDL)高效的传输数据而设计，这些数据仅在内存中存在，Parcelable是通过IBinder通信的消息的载体。

从上面的设计上我们就可以看出优劣了。

 

2、效率及选择

Parcelable的性能比Serializable好，在内存开销方面较小，所以在内存间数据传输时推荐使用Parcelable，如activity间传输数据，而Serializable可将数据持久化方便保存，所以在需要保存或网络传输数据时选择Serializable，因为android不同版本Parcelable可能不同，所以不推荐使用Parcelable进行数据持久化

 

3、编程实现

对于Serializable，类只需要实现Serializable接口，并提供一个序列化版本id(serialVersionUID)即可。而Parcelable则需要实现writeToParcel、describeContents函数以及静态的CREATOR变量，实际上就是将如何打包和解包的工作自己来定义，而序列化的这些操作完全由底层实现。

Parcelable的一个实现例子如下

```
public class MyParcelable implements Parcelable {
     private int mData;
     private String mStr;

     public int describeContents() {
         return 0;
     }

     // 写数据进行保存
     public void writeToParcel(Parcel out, int flags) {
         out.writeInt(mData);
         out.writeString(mStr);
     }

     // 用来创建自定义的Parcelable的对象
     public static final Parcelable.Creator<MyParcelable> CREATOR
             = new Parcelable.Creator<MyParcelable>() {
         public MyParcelable createFromParcel(Parcel in) {
             return new MyParcelable(in);
         }

         public MyParcelable[] newArray(int size) {
             return new MyParcelable[size];
         }
     };
     
     // 读数据进行恢复
     private MyParcelable(Parcel in) {
         mData = in.readInt();
         mStr = in.readString();
     }
 }
```

从上面我们可以看出Parcel的写入和读出顺序是一致的。如果元素是list读出时需要先new一个ArrayList传入，否则会报空指针异常。如下：

```
list = new ArrayList<String>();
in.readStringList(list);
```
 PS: 在自己使用时，read数据时误将前面int数据当作long读出，结果后面的顺序错乱，报如下异常，当类字段较多时务必保持写入和读取的类型及顺序一致。

```
11-21 20:14:10.317: E/AndroidRuntime(21114): Caused by: java.lang.RuntimeException: Parcel android.os.Parcel@4126ed60: Unmarshalling unknown type code 3014773 at offset 164
```

4、高级功能上

Serializable序列化不保存静态变量，可以使用Transient关键字对部分字段不进行序列化，也可以覆盖writeObject、readObject方法以实现序列化过程自定义

##Bitmap

在Android 2.2 (API level 8)以及之前，当垃圾回收发生时，应用的线程是会被暂停的，这会导致一个延迟滞后，并降低系统效率。 从Android 2.3开始，添加了并发垃圾回收的机制， 这意味着在一个Bitmap不再被引用之后，它所占用的内存会被立即回收。
在Android 2.3.3 (API level 10)以及之前, 一个Bitmap的像素级数据（pixel data）是存放在Native内存空间中的。 这些数据与Bitmap本身是隔离的，Bitmap本身被存放在Dalvik堆中。我们无法预测在Native内存中的像素级数据何时会被释放，这意味着程序容易超过它的内存限制并且崩溃。 自Android 3.0 (API Level 11)开始， 像素级数据则是与Bitmap本身一起存放在Dalvik堆中。

从Android 3.0 (API Level 11)开始，引进了BitmapFactory.Options.inBitmap字段。 如果使用了这个设置字段，decode方法会在加载Bitmap数据的时候去重用已经存在的Bitmap。这意味着Bitmap的内存是被重新利用的，这样可以提升性能，并且减少了内存的分配与回收。然而，使用inBitmap有一些限制，特别是在Android 4.4 (API level 19)之前，只有同等大小的位图才可以被重用。


##Fragment生命周期
onAttach -> onCreate() -> onCreateView -> onActivityCreated() -> onStart() -> onResume -> onPause -> onStop -> onDestroyView -> onDestroy-> onDetach

##进程的五个等级
+ 前台进程 Activity onResume Service bind startForground onCreate onStart onDestroy BroadcastReceiver onReceive
+ 可见进程 Activity onPause
+ 服务进程 Service 后台播放
+ 后台进程 Activity onStop
+ 空进程




