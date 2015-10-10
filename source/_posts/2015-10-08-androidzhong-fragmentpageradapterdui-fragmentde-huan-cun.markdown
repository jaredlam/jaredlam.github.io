---
layout: post
title: "Android中FragmentPagerAdapter对Fragment的缓存"
date: 2015-10-08 21:02:43 +0800
comments: true
categories: 
---

> ViewPager + FragmentPagerAdapter，时我们经常使用的一对搭档，其实际应用的代码也非常简单，但是也有一些容易被忽略的地方，这次我们就来讨论下FragmentPagerAdapter对Fragment的缓存应用。

&emsp;我们可以先看看最简单的实现，自定义Adapter如下：
```java
public class CustomPagerAdapter extends FragmentPagerAdapter{

    private List<Fragment> mFragments;

    public CustomPagerAdapter(FragmentManager fm, List<Fragment> fragments) {
        super(fm);
        this.mFragments = fragments;
        fm.beginTransaction().commitAllowingStateLoss();
    }

    @Override
    public Fragment getItem(int position) {
        return this.mFragments.get(position);
    }

    @Override
    public int getCount() {
        return this.mFragments.size();
    }

    @Override
    public long getItemId(int position) {
        return position;
    }
}
```
&emsp;代码比较简单，就不解释了，接着在Activity中使用这个Adapter:
```java
ViewPager pager = (ViewPager) findViewById(R.id.view_pager);

List<Fragment> fragmentList = new ArrayList<>()
TestFragment fragmentOne = new TestFragment();
fragmentOne.setText("One");
TestFragment fragmentTwo = new TestFragment();
fragmentTwo.setText("Two");
TestFragment fragmentThree = new TestFragment();
fragmentThree.setText("Three");

fragmentList.add(fragmentOne);
fragmentList.add(fragmentTwo);
fragmentList.add(fragmentThree);

CustomPagerAdapter adapter = new CustomPagerAdapter(getSupportFragmentManager(), fragmentList);
pager.setAdapter(adapter);
```
&emsp;这样就完成了一个FragmentPagerAdapter最基本的应用。现在，看上去一切都如我们所愿，但是真的没有任何问题了吗？

{% img /images/20151008_02.png %}


&emsp;接下来，我们来模拟一下程序运行在后台时，Android系统由于内存紧张，杀掉我们程序进程的情况：

>1. 首先运行程序至前台
2. 接下来，点击Home键，返回桌面，同时我们的程序退回至后台运行。
3. 进入Android Studio中，点击Android Monitor这个tab，并选择当前Device，并选择我们程序的进程名。
4. 点击Terminal Application这个小红叉按钮，如下图：{% img /images/20151008_01.png %}
5. 这个时候后台进程已经被杀掉了，但是应用程序历史里我们的应用还在，所以长按Home键，并选择我们的程序，让其恢复到前台。
6. 这时会看到，程序的确恢复到之前的页面。但奇怪的是，页面上却只有Hello，我们之前传入的Two到哪里去了？

{% img /images/20151008_03.png %}

&emsp;Fragment代码也比较简单，通过日志，我们发现恢复时，mText字段为空。所以页面上对应的TextView无法显示。

```java
public class TestFragment extends Fragment {

    private String mText;

    @Nullable
    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
        View view = inflater.inflate(R.layout.fragment_test, container, false);
        TextView textView = (TextView) view.findViewById(R.id.center_text_view);
        textView.setText(mText);
        return view;
    }

    public void setText(String text) {
        this.mText = text;
    }
}
```

&emsp;我们知道，以上是模拟Android系统内存紧张时，杀掉后台应用的流程。另外，当用户安装了类似360安全管家等应用，选择清理内存时，也会触发以上情况。

&emsp;那么当上面的流程发生时，Activity的onSaveInstanceState会被调用，以便我们可以保存当前的用户数据/页面状态等。当恢复时，在onCreate时，我们通过savedInstanceState参数，可以取到之前存储的数据，然后重新绑定到View上。

&emsp;这个过程都可以理解，可是回到我们的Activity代码当中:

```java
TestFragment fragmentOne = new TestFragment();
fragmentOne.setText("One");
TestFragment fragmentTwo = new TestFragment();
fragmentTwo.setText("Two");
TestFragment fragmentThree = new TestFragment();
fragmentThree.setText("Three");
fragmentList.add(fragmentOne);
fragmentList.add(fragmentTwo);
fragmentList.add(fragmentThree);

CustomPagerAdapter adapter = new CustomPagerAdapter(getSupportFragmentManager(), fragmentList);
        pager.setAdapter(adapter);
```
&emsp;这段代码，是在onCreate方法中调用的。应用从后台恢复的时候，这段代码是被完整的执行过的。既然这样，三个Fragment都被重新创建过，并设置过对应的Text值，那么为什么Fragment中mText字段仍然为空呢？

&emsp;难道说，呈现在屏幕上的Fragment，和我们在onCreate中实例化的Fragment，已然不是同一个实例？

&emsp;为了验证这个想法，在OnCreate中加入下面的日志：
```java
 TestFragment fragmentOne = new TestFragment();
 fragmentOne.setText("One");
 Log.i("test", "++++fragmentOne++++:" + fragmentOne.toString());
```
&emsp;同时在TestFragment的onCreateView方法中也记下日志：
```java
Log.i("test", "++++current fragment++++:" + this.toString());
```
&emsp;第一次运行，恩，没有问题。创建和运行的都是同一个实例 **534ed278**
```java
I/test: ++++fragmentOne++++:TestFragment{534ed278}
I/test: ++++current fragment++++:TestFragment{534ed278 #0 id=0x7f0c0066 android:switcher:2131492966:0}
```
&emsp;接下来，我们再次进行杀进程并恢复的过程。日志输出为：
```java
I/test: ++++fragmentOne++++:TestFragment{534c5c30}
I/test: ++++current fragment++++:TestFragment{534d10d4 #0 id=0x7f0c0066 android:switcher:2131492966:0}
```
&emsp;额。。果然，这次我们创建的Fragment，和实际经过onCreateView的Fragment。并不是同一个（**534c5c30/534d10d4**）。

&emsp;看来，还是要从源码中寻求真相，打开FragmentPagerAdapter的源码，在instantiateItem方法中发现了下面这一段：
```java
// Do we already have this fragment?
String name = makeFragmentName(container.getId(), itemId);
Fragment fragment = mFragmentManager.findFragmentByTag(name);

if (fragment != null) {
    if (DEBUG) Log.v(TAG, "Attaching item #" + itemId + ": f=" + fragment);
    mCurTransaction.attach(fragment);
} else {
    fragment = getItem(position);
    if (DEBUG) Log.v(TAG, "Adding item #" + itemId + ": f=" + fragment);
    mCurTransaction.add(container.getId(), fragment,
            makeFragmentName(container.getId(), itemId));
}
```
&emsp;makeFragmentName方法如下：
```java
private static String makeFragmentName(int viewId, long id) {
    return "android:switcher:" + viewId + ":" + id;
}
```
&emsp;原来，在实例化Fragment的时候，FragmentPagerAdapter会先通过makeFragmentName返回的tag，到FragmentManager当中进行查找是否有当前Fragment的缓存，如果有的话，就直接将之前的Fragment恢复回来并使用；反之，才会使用我们传入的新实例。

&emsp;而makeFragmentName产生的tag，只受我们重写的getItemId()方法返回值，和当前容器View的Id，container.getId()的影响。

&emsp;到这里，问题就清楚了，由于FragmentPagerAdapter会主动的去取缓存当中的Fragment，所以导致恢复回来之后，Fragment的实例不一样的问题。

&emsp;至于为什么mText字段为空，以及怎样解决这个情况，我们下一篇再来讨论^_^。


