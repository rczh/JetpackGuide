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
* ###  自定义view使用双向绑定

　　为自定义view定义一个自定义属性，属性值为双向绑定表达式

```xml
<com.example.databinding.twoway.MyView
            app:test="@={customViewModel.realValue}"
            android:layout_width="200dp"
            android:layout_height="wrap_content"/>
```

　　使用BindingAdapter注解为自定义属性定义set方法，使用InverseBindingAdapter注解定义get方法

```kotlin
@BindingAdapter("test")
fun setRealValue(view: MyView, value: Int) {
    //避免循环调用
    if(view.myValue != value){
        view.myValue = value
        view.setText((value).toString())
    }
}

@InverseBindingAdapter(attribute = "test")
fun getRealValue(editText: MyView): Int {
    return editText.myValue
}
```

　　databinding为使用双向绑定表达式的UI组件另外生成一个自定义属性，属性名为基础属性名加上AttrChanged后缀，使用BindingAdapter注解为这个自定义属性定义事件处理方法，该方法接收一个InverseBindingListener参数，用来当页面发生变化时通知databinding框架更新数据对象

　　绑定类Impl会在第一次执行executeBindings方法时调用该事件处理方法进行初始化

```kotlin
@BindingAdapter("app:testAttrChanged")
fun setListener(editText : MyView, listener : InverseBindingListener ) {
    if (listener != null) {
        editText.addTextChangedListener(object : TextWatcher{
            override fun afterTextChanged(s: Editable?) {
                listener.onChange()
            }

            override fun beforeTextChanged(s: CharSequence?, start: Int, count: Int, after: Int) {
            }

            override fun onTextChanged(s: CharSequence?, start: Int, before: Int, count: Int) {
            }
        })
    }
}
```

　　onChange调用数据对象的set方法更新属性值

```kotlin
    private androidx.databinding.InverseBindingListener mboundView2testAttrChanged = new androidx.databinding.InverseBindingListener() {
        @Override
        public void onChange() {
            int callbackArg_0 = com.example.databinding.twoway.TwoWayBindingAdapterKt.getRealValue(mboundView2);
            com.example.databinding.twoway.CustomAttributeViewModel customViewModel = mCustomViewModel;
            int customViewModelRealValue = 0;
            boolean customViewModelJavaLangObjectNull = false;
            customViewModelJavaLangObjectNull = (customViewModel) != (null);
            if (customViewModelJavaLangObjectNull) {
                customViewModel.setRealValue(((int) (callbackArg_0)));
            }
        }
    };
```

　　注意：页面更新会触发onChange方法，onChange方法调用数据对象的set方法，set方法调用notifyPropertyChanged触发databinding执行executeBindings更新绑定状态，executeBindings方法调用BindingAdapter注解定义的set方法，由于该方法会设置view的属性值，如果view属性值发生变化会再一次触发onChange方法造成循环调用

　　为了避免循环调用的情况，需要在数据对象的set方法或者BindingAdapter注解定义的set方法中判断属性值是否和原始值相同，如果则直接返回

* ### 双向绑定表达式中使用Converters类型转换

　　在双向绑定表达式中使用Converters类型转换时需要提供正向和逆向两个转换方法

```xml
<EditText
            android:text="@={Converter.intToString(convertViewModel.convertValue)}"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"></EditText>
```

　　使用InverseMethod注解为正向转换方法定义相对应的逆向转换方法，参数为方法名

```kotlin
object Converter {
    @InverseMethod("stringToInt")
    @JvmStatic fun intToString(value: Int): String {
        return value.toString()
    }

    @JvmStatic fun stringToInt(value: String): Int {
        return value.toInt()
    }
}
```

## 5.系统自带支持双向绑定的属性

Class|Attributes|Binding adapter
---|:--:|---:
AdapterView|android:selectedItemPosition<br>android:selection|AdapterViewBindingAdapter
CalendarView|android:date|CalendarViewBindingAdapter
CompoundButton|android:checked|	CompoundButtonBindingAdapter
DatePicker|android:year<br>android:month<br>android:day|DatePickerBindingAdapter
NumberPicker|android:value|NumberPickerBindingAdapter
RadioButton|android:checkedButton|RadioGroupBindingAdapter
RatingBar|android:rating|RatingBarBindingAdapter
SeekBar|android:progress|SeekBarBindingAdapter
TabHost|android:currentTab|TabHostBindingAdapter
TextView|android:text|TextViewBindingAdapter
TimePicker|android:hour<br>android:minute|TimePickerBindingAdapter


