# EditText 搜索优化

## 描述

设计一个 EditText 的文本监听器，停止输入 1s 后，如果文本发生变化则触发监听器。

例子：
文本内容是 111，111 -> 1111 -> 11111，连续输入都小于 1s，在输完后的 1s 触发监听器为 11111；
文本内容是 111，111 -> 1111 -> 111，连续输入都小于 1s，在输完后的 1s 不触发监听器；

类似微信的客户端搜索，不同的是微信在 111 -> 1111 -> 111 是会触发改变的。

<!-- more -->

效果如下所示，注意观察 title 文本的改变：

![demo.gif](http://ww1.sinaimg.cn/large/b75b8776ly1g6prj25wgsg206k0dwaod.gif)

## 参考答案
代码很简单，结合注释参考即可，小功能从构思到编码到完成到优化，其实也是要花不少时间的，希望可以帮到你们哈。
```java
public class SearchEditText extends EditText {

    private static final long LIMIT = 1000;

    private OnTextChangedListener mListener;
    private String                mStartText = "";// 记录开始输入前的文本内容
    private Runnable              mAction    = new Runnable() {
        @Override
        public void run() {
            if (mListener != null) {
                // 判断最终和开始前是否一致
                if (!StringUtils.equals(mStartText, getText().toString())) {
                    mStartText = getText().toString();// 更新 mStartText
                    mListener.onTextChanged(mStartText);
                }
            }
        }
    };

    public SearchEditText(Context context) {
        super(context);
    }

    public SearchEditText(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    public SearchEditText(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }

    /**
     * 在 LIMIT 时间内连续输入不触发文本变化
     */
    public void setOnTextChangedListener(OnTextChangedListener listener) {
        mListener = listener;
    }

    @Override
    protected void onTextChanged(final CharSequence text, int start, int lengthBefore, int lengthAfter) {
        super.onTextChanged(text, start, lengthBefore, lengthAfter);
        // 移除上一次的回调
        removeCallbacks(mAction);
        postDelayed(mAction, LIMIT);
    }

    @Override
    protected void onDetachedFromWindow() {
        super.onDetachedFromWindow();
        removeCallbacks(mAction);
    }

    public interface OnTextChangedListener {
        void onTextChanged(String text);
    }
}
```



## 结语

我正在打造一个帮助 Android 开发者们拿到更好 offer 的面试库————**[安卓 offer 收割基](https://github.com/Blankj/AndroidOfferKiller)**，欢迎 star，觉得不错的可以持续关注，有兴趣的可以一起加入进来和我一同打造。