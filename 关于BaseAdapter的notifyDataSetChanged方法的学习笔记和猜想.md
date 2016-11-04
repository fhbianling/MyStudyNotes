> 在才开始学习Android时，就被告知要在重写getItemViewType()的情况下重写getViewTypeCount()方法，最近在项目中由于忘写getViewTypeCount()方法，导致调用adapter的notifyDataSetChanged()方法时，用的左滑菜单库不能重新生成对应type的菜单项，在重写getViewTypeCount()方法后解决了问题。不过还是决定借此好好学习下ListView和这一块儿相关的知识。
> <br/>在直接看别人已经分析好的源码还是自己看源码的两个选项上，最终决定还是自己看源码，主要是为了锻炼下看源码的能力。

# notifyDataSetChanged #
在ListView的setAdapter()方法中，若ListView已经有一个adapter，则会为adapter取消注册观察者（DataSetObserver类，这是一个抽象类），然后为adapter注册一个DataSetObserver
这个DataSetObserver的具体实现是来自于ListView父类AbsListView的内部类AdapterDataSetObserver。

当调用notifyDataSetChanged时，触发DataSetObserve的onChanged方法，
而在AdapterDateSetObserve的onChanged方法的具体实现中，其源码如下：

    		mDataChanged = true;
            mOldItemCount = mItemCount;
            mItemCount = getAdapter().getCount();

            // Detect the case where a cursor that was previously invalidated has
            // been repopulated with new data.
            if (AdapterView.this.getAdapter().hasStableIds() && mInstanceState != null
                    && mOldItemCount == 0 && mItemCount > 0) {
                AdapterView.this.onRestoreInstanceState(mInstanceState);
                mInstanceState = null;
            } else {
                rememberSyncState();
            }
            checkFocus();
            requestLayout();

当requestLayout被调用时，经过View树的传递，最终会使ListView的onMeasure(...)方法被调用

     @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        // Sets up mListPadding
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
		...
        mItemCount = mAdapter == null ? 0 : mAdapter.getCount();
        if (mItemCount > 0 && (widthMode == MeasureSpec.UNSPECIFIED ||
                heightMode == MeasureSpec.UNSPECIFIED)) {
            final View child = obtainView(0, mIsScrap);

            measureScrapChild(child, 0, widthMeasureSpec);

            childWidth = child.getMeasuredWidth();
            childHeight = child.getMeasuredHeight();
            childState = combineMeasuredStates(childState, child.getMeasuredState());

            if (recycleOnMeasure() && mRecycler.shouldRecycleViewType(
                    ((LayoutParams) child.getLayoutParams()).viewType)) {
                mRecycler.addScrapView(child, 0);
            }
        }
		...     
    }

注意到在这段源码中出现了和ListView缓存机制相关的变量mRecycler，同时还有一个很重要的方法obtainView(...)

mRecycler是一个RecycleBin对象，这个类是AbsListView的内部类。
在源码介绍中说明它存储了两个级别的View：ActiveViews和ScrapViews。
（关于ActiveViews和ScrapViews的概念仍有疑惑，网上大多将ActiveViews认为为在屏上的活动View,ScrapViews认为为已废弃并可复用的View，不过这点不能说明源码注释中的at the
start of a layout以及At the end of layout是什么含义）
<br/>在代码中这两级View分级的具体实现是：

    	private View[] mActiveViews = new View[0];
        private ArrayList<View>[] mScrapViews;
除此之外还有几个全局变量：

        private int mViewTypeCount;
        private ArrayList<View> mCurrentScrap;
        private ArrayList<View> mSkippedScrap;
        private SparseArray<View> mTransientStateViews;
        private LongSparseArray<View> mTransientStateViewsById;


这里的mViewTypeCount在ListView中的setAdapter()方法中通过RecycleBin的setViewTypeCount(int viewTypeCount)方法被赋值给mRecycler.
其源码如下：

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

也即是说对于其缓存机制中的ScrapView这一级的View来说，viewTypeCount决定了其总共可以包含几种类型被缓存的View。

<br/>obtainView(...)方法来自于AbsListView

    View obtainView(int position, boolean[] isScrap) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "obtainView");

        isScrap[0] = false;

        // Check whether we have a transient state view. Attempt to re-bind the
        // data and discard the view if we fail.
        final View transientView = mRecycler.getTransientStateView(position);
        if (transientView != null) {
            final LayoutParams params = (LayoutParams) transientView.getLayoutParams();

            // If the view type hasn't changed, attempt to re-bind the data.
            if (params.viewType == mAdapter.getItemViewType(position)) {
                final View updatedView = mAdapter.getView(position, transientView, this);

                // If we failed to re-bind the data, scrap the obtained view.
                if (updatedView != transientView) {
                    setItemViewLayoutParams(updatedView, position);
                    mRecycler.addScrapView(updatedView, position);
                }
            }

            // Scrap view implies temporary detachment.
            isScrap[0] = true;
            return transientView;
        }

        final View scrapView = mRecycler.getScrapView(position);
        final View child = mAdapter.getView(position, scrapView, this);
        if (scrapView != null) {
            if (child != scrapView) {
                // Failed to re-bind the data, return scrap to the heap.
                mRecycler.addScrapView(scrapView, position);
            } else {
                isScrap[0] = true;

                child.dispatchFinishTemporaryDetach();
            }
        }

        if (mCacheColorHint != 0) {
            child.setDrawingCacheBackgroundColor(mCacheColorHint);
        }

        if (child.getImportantForAccessibility() == IMPORTANT_FOR_ACCESSIBILITY_AUTO) {
            child.setImportantForAccessibility(IMPORTANT_FOR_ACCESSIBILITY_YES);
        }

        setItemViewLayoutParams(child, position);

        if (AccessibilityManager.getInstance(mContext).isEnabled()) {
            if (mAccessibilityDelegate == null) {
                mAccessibilityDelegate = new ListItemAccessibilityDelegate();
            }
            if (child.getAccessibilityDelegate() == null) {
                child.setAccessibilityDelegate(mAccessibilityDelegate);
            }
        }

        Trace.traceEnd(Trace.TRACE_TAG_VIEW);

        return child;
    }

adapter的getView(...)方法就是在obtainView方法中被调用的，它首先会判断mRecycle中是否存在一个getTransientStateView(position)，
<br/>为null的情况下，则会调用adapter的getView()方法，同时调用mRecycle的getScrapView(position)方法，如果这两个view的引用不指向同一个View对象，则说明需要将position位置的view替换掉.

//TODO 2016/11/4 23:10:12 obtainView(...)的逻辑分析

<br/>getScrapView()方法

  	/**
    * @return A view from the ScrapViews collection. These are unordered.
    */
    View getScrapView(int position) {
            if (mViewTypeCount == 1) {
                return retrieveFromScrap(mCurrentScrap, position);
            } else {
                final int whichScrap = mAdapter.getItemViewType(position);
                if (whichScrap >= 0 && whichScrap < mScrapViews.length) {
                    return retrieveFromScrap(mScrapViews[whichScrap], position);
                }
            }
            return null;
        }

假如不重写ListView的getViewTypeCount()方法，会导致mViewTypeCount为默认值1（BaseAdapter该方法默认返回1），也就是说进入了retrieveFromScrap(mCurrentScrap, position)方法

	private View retrieveFromScrap(ArrayList<View> scrapViews, int position) {
            final int size = scrapViews.size();
            if (size > 0) {
                // See if we still have a view for this position or ID.
                for (int i = 0; i < size; i++) {
                    final View view = scrapViews.get(i);
                    final AbsListView.LayoutParams params =
                            (AbsListView.LayoutParams) view.getLayoutParams();

                    if (mAdapterHasStableIds) {
                        final long id = mAdapter.getItemId(position);
                        if (id == params.itemId) {
                            return scrapViews.remove(i);
                        }
                    } else if (params.scrappedFromPosition == position) {
                        final View scrap = scrapViews.remove(i);
                        clearAccessibilityFromScrap(scrap);
                        return scrap;
                    }
                }
                final View scrap = scrapViews.remove(size - 1);
                clearAccessibilityFromScrap(scrap);
                return scrap;
            } else {
                return null;
            }
        }

mCurrentScrap的引用指向mScrapViews数组的第一个元素的那个ArrayList<View>
<br/>它内部View变化只会在mViewTypeCount==1的情况下来自于addScrapView(...)方法，
<br/>//TODO 2016/11/4 23:10:18 换句话说，其size就是在mViewTypeCount=1的情况下scrapView这一级view的数量?
