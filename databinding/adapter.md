# binding adapter
## 1.定义
　　binding adapter用来为布局文件中引用表达式语言的UI组件属性匹配相对应的方法

## 2.作用
　　binding adapter允许为布局文件中引用表达式语言的UI组件属性指定自定义方法名，或者指定自定义方法实现

## 3.使用方式
* ### 自动匹配方法名
　　对于一个android:text="@{user.name}"属性，如果user.name类型为String，databinding将自动匹配setText(arg: String)方法

```xml
 <TextView android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="@{user.name}"/>
```

　　DrawerLayout组件不提供任何属性，但是组件本身包含了一系列的set方法，比如: setScrimColor(int)。通过自定义属性scrimColor，可以使databinding自动匹配setScrimColor方法，不需要为自定义属性添加任何实现

```xml
<android.support.v4.widget.DrawerLayout
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    app:scrimColor="@{@color/scrim}">
```

　　注意：databinding在匹配属性方法时不考虑命名空间，也就是说上面的例子中app:scrimColor也可以替换成android:scrimColor

* ### 指定自定义方法名
　　某些属性的实现方法不能够满足自动匹配方法名的规则，在这种情况下可以使用BindingMethods注解为属性关联自定义方法名

　　对于android:tint="@{color}"属性，系统默认的方法实现为setImageTintList，可以通过BindingMethods注解将android:tint属性的方法实现指定为setMyImageTintList

```kotlin
@BindingMethods(value = [
    BindingMethod(
        type = ImageView::class,
        attribute = "android:tint",
        method = "setMyImageTintList")])
class CustomImageView(context: Context?, attrs: AttributeSet?) :
    ImageView(context, attrs) {
     fun setMyImageTintList(tint: ColorStateList?) {
        super.setImageTintList(tint)
    }
}
```

　　BindingMethods注解可以应用于程序中的任何类，可以包含多个BindingMethod注解，每一个BindingMethod注解对应一个属性

* ### 指定自定义方法实现
　　由于系统并不提供单独的paddingLeft属性，仅仅提供setPadding方法。可以通过自定义属性app:paddingLeft="@{left}"并且使用BindingAdapter注解将该属性的实现关联到setPadding方法来实现这个功能

```kotlin
@BindingAdapter("android:paddingLeft")
fun setPaddingLeft(view: View, padding: Int) {
    view.setPadding(padding,
        view.getPaddingTop(),
        view.getPaddingRight(),
        view.getPaddingBottom())
}
```

　　自定义实现的第一个参数为属性对应UI组件类型，第二个参数为表达式取值类型

　　当自定义的BindingAdapter实现与系统默认BindingAdapter实现存在冲突时，系统使用自定义BindingAdapter实现


　　databinding中属性的事件处理对象只能使用带有一个方法的抽象类或者接口，如果事件处理对象包含多个方法，需要将其拆分为多个对象

```kotlin
@BindingAdapter(
    "android:onViewDetachedFromWindow",
    "android:onViewAttachedToWindow",
    requireAll = false
)
fun setListener(view: View, detach: ViewBindingAdapter.OnViewDetachedFromWindow?, attach: ViewBindingAdapter.OnViewAttachedToWindow?) {
    println("setListener..........")
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.HONEYCOMB_MR1) {
        val newListener: View.OnAttachStateChangeListener?
        newListener = if (detach == null && attach == null) {
            null
        } else {
            object : View.OnAttachStateChangeListener {
                override fun onViewAttachedToWindow(v: View) {
                    attach?.onViewAttachedToWindow(v)
                }

                override fun onViewDetachedFromWindow(v: View) {
                    detach?.onViewDetachedFromWindow(v)
                }
            }
        }

        val oldListener: View.OnAttachStateChangeListener? =
            ListenerUtil.trackListener(view, newListener, R.id.onAttachStateChangeListener)
        if (oldListener != null) {
            view.removeOnAttachStateChangeListener(oldListener)
        }
        if (newListener != null) {
            view.addOnAttachStateChangeListener(newListener)
        }
    }
}
```

　　由于OnAttachStateChangeListener中包含两个方法，需要将其拆分成OnViewDetachedFromWindow和OnViewAttachedToWindow两个事件处理对象

```xml
<View android:layout_width="20dip"
            android:layout_height="20dip"
            android:onViewDetachedFromWindow="@{attachMethod::onViewAttachedToWindow}"
            android:onViewAttachedToWindow="@{detachMethod::onViewDetachedFromWindow}"/>
```

# Object conversions
## 1.定义
　　当绑定表达式返回一个Object对象时，系统自动将Object对象转换为与属性相对应的方法参数类型。当Object对象和方法参数类型不匹配时，可以通过BindingConversion注解自定义类型转换

## 2.作用
　　用来将绑定表达式返回类型转换为与属性相对应方法的参数类型

## 3.使用方式
　　对于android:background属性来说，系统默认的处理方法需要一个ColorDrawable参数，当绑定表达式返回string时，程序需要使用BindingConversion注解自定义转换函数将string类型转换成ColorDrawable

```xml
<View
            android:background="@{isColor?@string/gray:@string/blue}"
            android:layout_width="20dip"
            android:layout_height="20dip"/>
```

　　转换函数应该是一个java静态方法或者kotlin全局方法

```kotlin
@BindingConversion
fun convertColorToDrawable(color: String) : ColorDrawable {
    var tmp = Color.parseColor(color)
    return ColorDrawable(tmp)
}
```

　　注意，绑定表达式中的返回值类型必须是一致的，不能在表达式中返回两种类型，如下示例是非法的

```xml
<View
   android:background="@{isError ? @drawable/error : @color/white}"
   android:layout_width="wrap_content"
   android:layout_height="wrap_content"/>
```

