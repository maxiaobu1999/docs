# Android 事件分发

## 1 事件传递

![Android事件分发机制](../sources/Android事件分发机制.png)

- 这里事件特指`MotionEvent.ACTION_DOWN`

- onTouch():是OnTouchListener接口的方法。onTouch优先级比onTouchEvent高



## 2 OnTouchListener & OnClickListener

1. 设置OnTouchListener：onTouch方法返回false时，onTouch方法及View的onTouchEvent方法依次被调用；onTouch方法返回true时，只调用onTouch方法，onTouchEvent方法不再被调用
2. 设置OnTouchListener后：onTouch方法返回false，不影响OnClickListener及OnLongClickListener的触发；onTouch方法返回true时，OnClickListener及OnLongClickListener不再触发
3. OnClickListener的触发条件是手指从触屏抬起；OnLongClickListener的触发条件是按下触屏且停留一段事件
4. onLongClick方法返回false不影响OnClickListener的触发；onLongClick方法返回true，OnClickListener不再触发



## 3滑动

#3.1 scrollTo/scrollBy