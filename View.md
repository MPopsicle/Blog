

# Android自定义View构造函数

android开发中也都遇到过自定义View，第一步肯定是添加构造函数吧，那构造函数到底负责什么哪些方面的职责呢。第一个想到的应该是自定义View中需要设置的属性，没错这也是构造函数的重要职责之一。接下来我们就一起看看自定义View构造函数的一二三吧。

```java
public class CustomView extends View {
    private final static String TAG = CustomView.class.getSimpleName();
    /**
     * 在Java代码中直接new一个CustomView实例的时候，会调用该构造函数
     * @param context
     */
    public CustomView(Context context) {
        this(context,null);
    }
    /**
     * 在xml中引用CustomView标签的时候，会调用2参数的构造函数。
     * 这种方式通常是我们需要自定义View的属性的时候，使用2参数的构造函数。
     */
    public CustomView(Context context, AttributeSet attrs) {
        this(context, attrs,0);
    }
    /**
     * 3个参数的构造函数一般是由我们主动调用的，如：上面2个参数的构造函数调用。
     */
    public CustomView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        TypedArray ta = context.obtainStyledAttributes(attrs, R.styleable.CustomView);
        if (ta!=null){
            String attr1 = ta.getString(R.styleable.CustomView_attr1);
            String attr2 = ta.getString(R.styleable.CustomView_attr2);
            String attr3 = ta.getString(R.styleable.CustomView_attr3);
            String attr4 = ta.getString(R.styleable.CustomView_attr4);

            Log.i(TAG, "attr1=" + attr1);
            Log.i(TAG, "attr2=" + attr2);
            Log.i(TAG, "attr3=" + attr3);
            Log.i(TAG, "attr4=" + attr4);
            ta.recycle();
        }
    }
}
```

主要看obtainStyledAttributes方法，跟进去看

```java
public final TypedArray obtainStyledAttributes(
            AttributeSet set, @StyleableRes int[] attrs) {
        return getTheme().obtainStyledAttributes(set, attrs, 0, 0);
    }
```

继续跟进，很快就可以找到最终实现

```java
        TypedArray obtainStyledAttributes(@NonNull Resources.Theme wrapper,
                AttributeSet set,
                @StyleableRes int[] attrs,
                @AttrRes int defStyleAttr,
                @StyleRes int defStyleRes) {
            synchronized (mKey) {
                final int len = attrs.length;
                final TypedArray array = TypedArray.obtain(wrapper.getResources(), len);

                // XXX note that for now we only work with compiled XML files.
                // To support generic XML files we will need to manually parse
                // out the attributes from the XML file (applying type information
                // contained in the resources and such).
                final XmlBlock.Parser parser = (XmlBlock.Parser) set;
                AssetManager.applyStyle(mTheme, defStyleAttr, defStyleRes,
                        parser != null ? parser.mParseState : 0,
                        attrs, array.mData, array.mIndices);
                array.mTheme = wrapper;
                array.mXml = parser;

                return array;
            }
        }
```

解释一下几个参数

`AttributeSet set：`属性值得集合；

`int[] attr：`自定义属性在`R`类中自动生成的`int`型的数组，数组中包含自定义属性的资源ID；

`int defStyleAttr:`当前Theme中包含的一个指向style的引用，当我们没有给自定义View设置`declare-styleable`值的时候，默认从该集合中查找布局文件中配置的属性值。传入`0`表示不向该`defStyleAttr`中查找默认值；

`int defStyleRes:`指向一个Style的资源ID，但只在defStyleAttr为0或者Theme中没有为defStyleAttr属性赋值的时候起作用。

##### 先说结论：

```text
View中自定义的属性值可以通过很多种方式来进行赋值，不同的赋值方式中存在一个优先级的问题；举个例子，通过A方式和B方式对属性attribute_1进行赋值，如果A的优先级高于B，则A方式赋值后再B方式赋值，B方式所赋的值无法覆盖A方式的赋值，反正却可以。
属性赋值优先级：
	xml中CustomView标签下直接赋值 > xml中CustomView标签下通过引用style进行赋值 > CustomView所在的Activity的Theme中指定style进行赋值 > 用构造函数中defStyleRes指定的默认值进行赋值
```

不明白，没关系，下面会通过现象来证明以上结论。

为了更好进行比较，我将所有的属性的类型定义为String字符型，

```xml
path:/res/values/attr_custom_view.xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <declare-styleable name="CustomView">
        <attr name="attr1" format="string"/>
        <attr name="attr2" format="string" />
        <attr name="attr3" format="string" />
        <attr name="attr4" format="string" />
    </declare-styleable>
    <attr name="attr5" format="reference" />
</resources>
```

说明一下，`declare-styleable`标签只是为了方便attr的使用，attr不依赖与`declare-styleable`;也就是说你完全可以直接在`resources`中直接使用attr。定义一个attr就会在R类中生成一个该属性对应的ID，而通过在`declare-styleable`定义一组attr，就会在R类中生成一个该组属性对应的ID属性。

```java
TypedArray ta = context.obtainStyledAttributes(attrs, R.styleable.CustomView);
或者
int[] costomAttrs = {R.attr.attr1,R.attr.attr2,R.attr.attr3,R.attr.attr4};
TypedArray ta = context.obtainStyledAttributes(attrs, costomAttrs);
```

所以说`declare-styleable`标签只是为了方便attr的使用，在obtainStyledAttributes方法中传入的参数都是一个属性的资源ID数组，所以只要能取到属性的资源ID，怎么做都是可以的。

接下来就开始赋值和取属性值了：

##### 第一个参数和第二个参数

直接在xml中赋值和在style引用中进行赋值做比较：

```xml
src/main/res/layout/activity_main.xml
<?xml version="1.0" encoding="utf-8"?>
<android.support.constraint.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools" 
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="com.itcodecook.viewconstructor.MainActivity">
    <com.itcodecook.viewconstructor.view.CustomView
        android:layout_width="match_parent"
        android:layout_height="300dp"
        app:attr1="value_attr1"
        style="@style/CustomViewStyle"/>
</android.support.constraint.ConstraintLayout>
```

```xml
src/main/res/values/styles.xml
<resources>   
    <style name="CustomViewStyle">
        <item name="attr1">value_attr1_from_style</item>
        <item name="attr2">value_attr2_from_style</item>
    </style>
</resources>
```

```java
 public CustomView(Context context, AttributeSet attrs) {
        this(context, attrs,0);}    
 public CustomView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        TypedArray ta = context.obtainStyledAttributes(attrs, R.styleable.CustomView);
        if (ta!=null){
            String attr1 = ta.getString(R.styleable.CustomView_attr1);
            String attr2 = ta.getString(R.styleable.CustomView_attr2);
            String attr3 = ta.getString(R.styleable.CustomView_attr3);
            String attr4 = ta.getString(R.styleable.CustomView_attr4);
            Log.i(TAG, "attr1=" + attr1);
            Log.i(TAG, "attr2=" + attr2);
            Log.i(TAG, "attr3=" + attr3);
            Log.i(TAG, "attr4=" + attr4);
            ta.recycle();
        }
    }
```

执行结果：

http://upload-images.jianshu.io/upload_images/7434251-f8ab3f5b25479b4b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240

![屏幕快照 2017-09-06 下午7.38.43](/Users/liumin/Desktop/屏幕快照 2017-09-06 下午7.38.43.png)

显而易见，xml中直接对属性进行赋值的优先级高于在style引用中的赋值。

##### 第三个参数defStyleAttr

上文也有说到View的三参数构造函数是手动调用的，在通过xml中标签的形式实例化View系统是调用2个参数的构造函数的。从上文的代码中也可以看到是由两参数的构造函数去调用三参数的构造函数，只不过第三个参数传0。

从字面上理解，defStyleAttr表示默认的style，对它进行赋值的优势在于可以统一样式。

1,在res/values/attr_custom_view.xml中定义一个attr5，attr5是一个引用类型；

2,找到CustomView所在的Activity或者Application的theme属性，如果Activity有就用Activity的theme，没有就用Application的theme，然后再Activity的Theme中给attr5赋值;

3,在CustomView中调三参数的构造方法中传入第三个参数：`R.attr.attr5`。

```xml
src/main/AndroidManifest.xml:
<activity android:name=".MainActivity"
          android:theme="@style/AppTheme">
  		  ···
</activity>   
src/main/res/values/styles.xml:
<style name="AppTheme" parent="android:Theme.Holo.Light.DarkActionBar">
        <!-- Customize your theme here. -->
        <item name="attr5">@style/CustomViewAttr5Value</item></style>
<style name="CustomViewAttr5Value">
        <item name="attr1">value_attr1_from_defStyleAttr</item>
        <item name="attr2">value_attr2_from_defStyleAttr</item>
        <item name="attr3">value_attr3_from_defStyleAttr</item></style>
```

```java
//2个参数的构造函数中调用3参数的构造函数时，需传入默认的取值
public CustomView(Context context, AttributeSet attrs) {
        this(context,attrs,R.attr.attr5);}
 public CustomView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        TypedArray ta = context.obtainStyledAttributes(attrs,R.styleable.CustomView,defStyleAttr,0);
   ···
 }
```

执行结果：

<image src="http://upload-images.jianshu.io/upload_images/7434251-0fc967b4adc79146.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240"/>

![屏幕快照 2017-09-06 下午8.22.03](/Users/liumin/Desktop/屏幕快照 2017-09-06 下午8.22.03.png)

##### 第四个参数defStyleAttr

前面也说过，只有当第三个参数为0的时候或者Theme没有为第三个参数赋值的时候，它才有效；

在src/main/res/values/styles.xml中添加：

```xml
src/main/res/values/styles.xml: 
<style name="CustomViewDefStyleRes">
        <item name="attr1">value_attr1_from_defStyleRes</item>
        <item name="attr2">value_attr2_from_defStyleRes</item>
        <item name="attr3">value_attr3_from_defStyleRes</item>
        <item name="attr4">value_attr4_from_defStyleRes</item></style>
```

在代码中传入：

```java
   public CustomView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        TypedArray ta =             context.obtainStyledAttributes(attrs,R.styleable.CustomView,defStyleAttr,
                               R.style.CustomViewDefStyleRes);
        ···
        }
```

执行结果：

http://upload-images.jianshu.io/upload_images/7434251-817f40fa97fa659e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240

![屏幕快照 2017-09-06 下午8.31.43](/Users/liumin/Desktop/屏幕快照 2017-09-06 下午8.31.43.png)

修改，将第三个参数传入0，即不使用默认的style。

```java
   public CustomView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        TypedArray ta =             context.obtainStyledAttributes(attrs,R.styleable.CustomView,0,
                               R.style.CustomViewDefStyleRes);
        ···
        }
```

执行结果：

http://upload-images.jianshu.io/upload_images/7434251-33d18f8aa7e66080.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240

![屏幕快照 2017-09-06 下午8.32.30](/Users/liumin/Desktop/屏幕快照 2017-09-06 下午8.32.30.png)

修改，或者将src/main/res/values/styles.xml中的attr5注释掉：

```xml
src/main/res/values/styles.xml:
<style name="AppTheme" parent="android:Theme.Holo.Light.DarkActionBar">
        <!-- Customize your theme here. -->
        <!-- <item name="attr5">@style/CustomViewAttr5Value</item> -->
</style>
```

执行结果：

http://upload-images.jianshu.io/upload_images/7434251-0ee6e3ca58f1a22c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240

![屏幕快照 2017-09-06 下午8.34.59](/Users/liumin/Desktop/屏幕快照 2017-09-06 下午8.34.59.png)

由此可见，以上结论的正确性。

