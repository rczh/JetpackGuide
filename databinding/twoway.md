# two-way binding
## 1. 定义
双向绑定能够同时响应数据对象值的更新并且监听页面上的用户操作

## 2. 作用
用来移除单向绑定中的事件处理方法

## 3. 实现原理
双向绑定的绑定表达式需要使用@={}形式，绑定类Impl会对使用双向绑定表达式的UI组件注册更新事件处理方法，数据对象必须实现Observable接口，当用户进行页面操作时事件处理方法回调数据对象属性值的set方法，set方法调用notifyPropertyChanged通知databinding执行executeBindings更新绑定状态

```xml
        <CheckBox
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:id="@+id/rememberMeCheckBox"
            android:checked="@={viewmodel.rememberMe}"/>
```

注册UI组件更新的事件处理方法

```java
private androidx.databinding.InverseBindingListener rememberMeCheckBoxandroidCheckedAttrChanged = new androidx.databinding.InverseBindingListener() {
        @Override
        public void onChange() {
            boolean callbackArg_0 = rememberMeCheckBox.isChecked();
            boolean viewmodelJavaLangObjectNull = false;
            boolean viewmodelRememberMe = false;
            com.example.databinding.twoway.LoginViewModel viewmodel = mViewmodel;
            viewmodelJavaLangObjectNull = (viewmodel) != (null);
            if (viewmodelJavaLangObjectNull) {
                //回调数据对象属性值的set方法
                viewmodel.setRememberMe(((boolean) (callbackArg_0)));
            }
        }
    };
```

set方法调用notifyPropertyChanged触发databinding执行executeBindings更新绑定状态

```kotlin
class LoginViewModel : BaseObservable() {
     val data = Data()
    @Bindable
    fun getRememberMe(): Boolean {
        return data.myrememberMe
    }
    fun setRememberMe(value: Boolean) {
        //避免无限循环
        if (data.myrememberMe != value) {
            data.myrememberMe = value
            notifyPropertyChanged(BR.rememberMe)
        }
    }
}
```

双向绑定的实现就是调用被观察者数据对象的set方法

## 4.实例
* ###  自定义属性使用双向绑定

如何避免无限循环

* ### 双向绑定表达式中使用Converters类型转换
