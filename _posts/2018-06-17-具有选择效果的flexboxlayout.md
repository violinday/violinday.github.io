---
layout: post
title: 具有选择效果的flexablelayout
categories: [Android]
description: 具有选择效果的flexablelayout
keywords: 具有选择效果的flexablelayout
---


## 具有选择效果的flexablelayout

最近项目用到标签选择效果，类似
![效果图](/images/6-17-flexboxlayout.png)
不同的是，要带有选中效果，即，点击选中，并回调选中项。
回调代码如下：

``` Java
public interface OnSelectChangedListener {
    void onItemSelected(int position, View view);
}


final String[] labels = new String[]{"程序员", "影视天堂", "美食", "漫画.手绘", "广告图", "旅行.在路上","娱乐八卦", "青春", "谈写作", "短篇小说", "散文", "摄影"};

houseFlexboxlayoutVideolabels.setData(labels);

houseFlexboxlayoutVideolabels.setOnSelectChangedListener(new SelectableFlexboxLayout.OnSelectChangedListener() {
    @Override
    public void onItemSelected(int position, View view) {
        Toast.makeText(getApplication(),labels[position], Toast.LENGTH_LONG).show;
    }
});
```

下面直接将封装代码奉上，有问题可以留言联系哈。

selectable_flexboxlayout_container.xml
``` xml
<?xml version="1.0" encoding="utf-8"?>
<com.google.android.flexbox.FlexboxLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:link="http://schemas.android.com/tools"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/house_video_label_flexbox_container"
    android:layout_width="match_parent"
    android:layout_height="wrap_content">

</com.google.android.flexbox.FlexboxLayout>
```

label_item.xml
``` xml
<?xml version="1.0" encoding="utf-8"?>
<TextView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/tv_video_label"
    android:layout_width="wrap_content"
    android:layout_height="@dimen/dimen_32dp"
    android:paddingLeft="@dimen/margin_12"
    android:paddingRight="@dimen/margin_12"
    android:gravity="center"
    android:textSize="@dimen/textsize_13"
    android:textColor="@color/color_4a4e59"
    android:background="@drawable/selector_bg_label" /> 
```

点击效果 selector_bg_label.xml
``` xml
<?xml version="1.0" encoding="utf-8"?>
<selector xmlns:android="http://schemas.android.com/apk/res/android">
  <item android:drawable="@drawable/label_selected" android:state_selected="true" />
  <item android:drawable="@drawable/label_selected"
  android:state_checked="true" />
  <item android:drawable="@drawable/label_normal" />
</selector> 
```

label_selected,label_normal即为自己实现的选中效果

``` Java
public class SelectableFlexboxLayout extends FlexboxLayout implements View.OnClickListener {

  private FlexboxLayout container;
  private LayoutInflater inflater;
  private ViewHolder mSelectedItem;
  private OnSelectChangedListener mOnSelectChangedListener;
  private List<ViewHolder> holders = new ArrayList<>();

  public SelectableFlexboxLayout(Context context) {
    super(context);
  }

  public SelectableFlexboxLayout(Context context, AttributeSet attrs) {
    super(context, attrs);
    inflater = LayoutInflater.from(getContext());
    inflate(context, R.layout.selectable_flexboxlayout_container, this);
    container = (FlexboxLayout)findViewById(R.id.house_video_label_flexbox_container);
    container.setFlexWrap(FLEX_WRAP_WRAP);
    container.setAlignItems(ALIGN_ITEMS_FLEX_START);
    container.setDividerDrawable(getResources().getDrawable(R.drawable.house_video_label_divider_placeholder));
    container.setShowDivider(SHOW_DIVIDER_MIDDLE);
  }

  public SelectableFlexboxLayout(Context context, AttributeSet attrs, int defStyleAttr) {
    super(context, attrs, defStyleAttr);
  }

  private void addItems(CharSequence[] titles, int initSelected) {
    if (titles == null || titles.length == 0) return;

    for (int i = 0; i < titles.length; i++) {
      View item = inflater.inflate(R.layout.label_item, container, false);
      ViewHolder holder = new ViewHolder(item, i, titles[i]);
      item.setTag(holder);
      item.setOnClickListener(this);
      holders.add(holder);
      if (i == initSelected) {
        mSelectedItem = holder;
        holder.setItemStatus(true);
      } else {
        holder.setItemStatus(false);
      }
      container.addView(item);
    }
  }

  @Override public void onClick(View v) {
    ViewHolder holder = (ViewHolder) v.getTag();
    if (holder == mSelectedItem) return;

    mSelectedItem.setItemStatus(false);
    mSelectedItem = holder;
    holder.setItemStatus(true);
    if (mOnSelectChangedListener != null) {
      mOnSelectChangedListener.onItemSelected(holder.mPosition, v);
    }
  }

  public static class ViewHolder {
    private Context context;

    private View mRoot;
    private TextView mTitle;
    private int mPosition;

    ViewHolder(final View root, int position, CharSequence title) {
      this.context = root.getContext();
      mRoot = root;
      mTitle = (TextView) root.findViewById(R.id.tv_video_label);
      mTitle.setText(title);
      mPosition = position;
    }

    public void setItemStatus(boolean isSelected) {
      mTitle.setTextColor(isSelected ? Color.WHITE : ContextCompat.getColor(context, R.color.color_4A4E59));
      mRoot.setSelected(isSelected);
    }
  }

  public void setOnSelectChangedListener(OnSelectChangedListener listener) {
    mOnSelectChangedListener = listener;
  }

  public void setSelectPosition(int position) {
    if (mSelectedItem != null && mSelectedItem.mPosition == position) return;
    if (position >= this.getChildCount()) return;

    View view = this.getChildAt(position);
    if (view != null) {
      mSelectedItem.setItemStatus(false);
      mSelectedItem = (ViewHolder) view.getTag();
      mSelectedItem.setItemStatus(true);
    }
  }

  public void setData(String[] titles) {
    setData(titles, 0);
  }

  public void setData(String[] titles, int initIndex) {
    addItems(titles, initIndex);
  }

  public List<ViewHolder> getViewHolders() {
    return holders != null ? Collections.unmodifiableList(holders)
            : Collections.<ViewHolder>emptyList();
  }

  public interface OnSelectChangedListener {
    void onItemSelected(int position, View view);
  }
}

```



如有缺漏不足之处，欢迎创建issue，PR 😄

