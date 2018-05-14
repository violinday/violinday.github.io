---
layout: post
title: Android 自定义进度条效果
categories: [Android]
description: Android CustomView Progress
keywords: CustomView, Progress
---



## Android 自定义进度条效果

最近再看自定义View的相关代码，
比照着网上的效果做了一个，录屏有卡顿

![效果图](/images/5-13-customview-progress.gif)

下面直接放出源码：

主要分为两部分 重写的onDraw方法、自定义的setProgress(int progress)方法

下面看onDraw方法，分为三步：画背景，画进度，画文字内容。：

``` Java
@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);

    // TODO: consider storing these as member variables to reduce
    // allocations per draw cycle.
    int paddingLeft = getPaddingLeft();
    int paddingTop = getPaddingTop();
    int paddingRight = getPaddingRight();
    int paddingBottom = getPaddingBottom();

    int contentWidth = getWidth() - paddingLeft - paddingRight;
    int contentHeight = getHeight() - paddingTop - paddingBottom;

    // 2.画背景大圆弧
    int centerX = contentWidth / 2;
    int centerY = contentHeight / 2;
    // 设置圆弧画笔的宽度
    mTextPaint.setStrokeWidth(mRoundWidth);
    // 设置为 ROUND
    mTextPaint.setStrokeCap(Paint.Cap.ROUND);
    mTextPaint.setStrokeJoin(Paint.Join.ROUND);
    // 设置画笔颜色
    mTextPaint.setColor(mRoundColor);
    mTextPaint.setStyle(Paint.Style.STROKE);
    // 半径
    int radius = (int) (centerX - mRoundWidth);
    //避免频繁创建对象
    if (mOval == null) {
        mOval = new RectF(centerX - radius, centerY - radius, centerX + radius, centerY + radius);
    }
    // 画背景圆弧
    canvas.drawArc(mOval, 150, 240, false, mTextPaint);

    //画进度
    mTextPaint.setColor(Color.BLUE);
    // 根据当前百分比计算圆弧扫描的角度
    canvas.drawArc(mOval, mStartAngle, progress/100f  * mSweepAngle, false, mTextPaint);

    // 重置画笔
    mTextPaint.reset();
    mTextPaint.setAntiAlias(true);
    mTextPaint.setTextSize(getResources().getDimensionPixelSize(R.dimen.lb_action_text_size));
    mTextPaint.setColor(Color.RED);
    String mStep = progress + "%";
    // 测量文字的宽高
    mTextPaint.getTextBounds(mStep, 0, mStep.length(), textBounds);
    int dx = (getWidth() - textBounds.width()) / 2;
    // 获取画笔的FontMetrics
    Paint.FontMetrics fontMetrics = mTextPaint.getFontMetrics();
    // 计算文字的基线
    int baseLine = (int) (getHeight() / 2 + (fontMetrics.bottom - fontMetrics.top) / 2 - fontMetrics.bottom);
    // 绘制步数文字
    canvas.drawText(mStep, dx, baseLine, mTextPaint);
}
``` 

最后按照一定的效果（Interceptor插值器实现进度效果）更新progress，invalidate页面，从而重新调用onDraw方法回执界面。实现如下方法：

``` Java 
public void setProgress(int setProgress) {
    objectAnimator = ValueAnimator.ofInt(0, setProgress);
    objectAnimator.setDuration(3000);
    // 不同的拦截器实现不同的展示效果 new BounceInterpolator(); 
    // 可以实现类似速度表盘效果
    objectAnimator.setInterpolator(new AccelerateDecelerateInterpolator());
    objectAnimator.setEvaluator(new IntEvaluator());

    objectAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
        @Override
        public void onAnimationUpdate(ValueAnimator animation) {
            progress = (int)animation.getAnimatedValue();
            invalidate();
        }
    });
    objectAnimator.start();
}
```


最后，有什么可以优化的地方请多指教 (ﾟ▽ﾟ)/
