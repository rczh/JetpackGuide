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


* ### 指定自定义方法实现
