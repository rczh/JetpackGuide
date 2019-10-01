# observable data objects
## 1.定义
　　通常情况下，更新绑定类中数据对象的值并不会触发databinding更新UI。databinding提供observable类，用来实现当数据对象值更新后通知databinding更新UI的功能

## 2.作用
　　用来在绑定类中数据对象值更新后通知databinding更新UI

## 3.实现原理
* ### Observable接口
　　数据对象类需要实现Observable接口，当初始化数据对象时绑定类Impl会调用updateRegistration方法，该方法调用WeakPropertyListener.addListener方法，addListener方法回调addOnPropertyChangedCallback方法注册WeakPropertyListener对象作为数据对象观察者

　　当数据对象类中属性值发生变化时调用OnPropertyChangedCallback.onPropertyChanged方法，该方法调用WeakPropertyListener.onPropertyChanged方法，WeakPropertyListener.onPropertyChanged方法最终触发绑定类执行executeBindings方法更新UI

```java
      public interface Observable {
          void addOnPropertyChangedCallback(OnPropertyChangedCallback callback);
          void removeOnPropertyChangedCallback(OnPropertyChangedCallback callback);
          abstract class OnPropertyChangedCallback {
              public abstract void onPropertyChanged(Observable sender, int propertyId);
          }
      }

    public void setUser(@Nullable com.example.databinding.CustomObserableUser User) {
        //updateRegistration方法调用WeakPropertyListener.addListener方法
        updateRegistration(0, User);
        this.mUser = User;
        synchronized(this) {
            mDirtyFlags |= 0x1L;
        }
        notifyPropertyChanged(BR.user);
        super.requestRebind();
    }
    
    private static class WeakPropertyListener extends Observable.OnPropertyChangedCallback
            implements ObservableReference<Observable> {
        final WeakListener<Observable> mListener;

        public WeakPropertyListener(ViewDataBinding binder, int localFieldId) {
            mListener = new WeakListener<Observable>(binder, localFieldId, this);
        }

        //addListener方法注册数据对象观察者
        @Override
        public void addListener(Observable target) {
            target.addOnPropertyChangedCallback(this);
        }

        @Override
        public void onPropertyChanged(Observable sender, int propertyId) {
            ViewDataBinding binder = mListener.getBinder();
            if (binder == null) {
                return;
            }
            Observable obj = mListener.getTarget();
            if (obj != sender) {
                return; // notification from the wrong object?
            }
            //handleFieldChange方法调用requestRebind方法，requestRebind方法触发调用绑定类executeBindings方法更新UI
            binder.handleFieldChange(mListener.mLocalFieldId, sender, propertyId);
        }
    }
    
```

　　为了简单起见，databinding提供了BaseObservable类封装了Observable接口，数据对象类需要在属性值变化时调用notifyPropertyChanged方法通知观察者

　　Bindable注解用来通知databinding生成BR中的属性字段

　　如果数据对象类不能继承BaseObservable，可以实现Observable接口并且使用PropertyChangeRegistry类注册、通知观察者

```kotlin
class CustomObserableUser : BaseObservable() {

    @get:Bindable
    var firstName: String = ""
        set(value) {
            field = value
            notifyPropertyChanged(BR.firstName)
        }

    @get:Bindable
    var lastName: String = ""
        set(value) {
            field = value
            notifyPropertyChanged(BR.lastName)
        }
}
```

　　初始化数据对象时databinding将为数据对象注册观察者

```kotlin
class BindingCustomObservableActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        val binding: ActivityBindingCustomObservableMainBinding = DataBindingUtil.setContentView(this, R.layout.activity_binding_custom_observable_main)
        var myUser = CustomObserableUser()
        myUser.firstName = "first name aa"
        myUser.lastName = "last name"
        
        //绑定类Impl会调用updateRegistration方法注册观察者
        binding.user = myUser

        Handler().postDelayed({
            //自动通知databinding更新UI
            myUser.firstName = "new first name"
            myUser.lastName = "new last name"
        }, 2000)
    }
}
```

* ### Observable字段类型
　　当数据对象类中只有少数字段需要在值改变后更新UI时，可以将字段声明为Observable字段类型

```kotlin
class ObserableUser {
    val firstName = ObservableField<String>()
    val lastName = ObservableField<String>()
    val map = ObservableArrayMap<String, Any>().apply {
        put("okey1", "Google")
        put("okey2", "Inc.")
        put("okey3", 17)
    }
}
```

　　与Observable接口方式不同，绑定类Impl会在executeBindings方法中对数据对象字段调用updateRegistration方法

　　Observable字段类型继承于BaseObservable类，实现原理相同

```java
@Override
    protected void executeBindings() {
        long dirtyFlags = 0;
        synchronized(this) {
            dirtyFlags = mDirtyFlags;
            mDirtyFlags = 0;
        }
        java.lang.String userLastNameGet = null;
        androidx.databinding.ObservableField<java.lang.String> userFirstName = null;
        com.example.databinding.ObserableUser user = mUser;
        androidx.databinding.ObservableField<java.lang.String> userLastName = null;
        androidx.databinding.ObservableArrayMap<java.lang.String, java.lang.Object> userMap = null;
        java.lang.Object userMapOkey1 = null;
        java.lang.String userFirstNameGet = null;

        if ((dirtyFlags & 0x1fL) != 0) {
            if ((dirtyFlags & 0x19L) != 0) {
                    if (user != null) {
                        // read user.firstName
                        userFirstName = user.getFirstName();
                    }
                    updateRegistration(0, userFirstName);
                    if (userFirstName != null) {
                        // read user.firstName.get()
                        userFirstNameGet = userFirstName.get();
                    }
            }
            if ((dirtyFlags & 0x1aL) != 0) {
                    if (user != null) {
                        // read user.lastName
                        userLastName = user.getLastName();
                    }
                    updateRegistration(1, userLastName);
                    if (userLastName != null) {
                        // read user.lastName.get()
                        userLastNameGet = userLastName.get();
                    }
            }
            if ((dirtyFlags & 0x1cL) != 0) {
                    if (user != null) {
                        // read user.map
                        userMap = user.getMap();
                    }
                    updateRegistration(2, userMap);
                    if (userMap != null) {
                        // read user.map.okey1
                        userMapOkey1 = userMap.get("okey1");
                    }
            }
        }
        // batch finished
        if ((dirtyFlags & 0x19L) != 0) {
            // api target 1
            androidx.databinding.adapters.TextViewBindingAdapter.setText(this.mboundView1, userFirstNameGet);
        }
        if ((dirtyFlags & 0x1aL) != 0) {
            // api target 1
            androidx.databinding.adapters.TextViewBindingAdapter.setText(this.mboundView2, userLastNameGet);
        }
        if ((dirtyFlags & 0x1cL) != 0) {
            // api target 1
            androidx.databinding.adapters.TextViewBindingAdapter.setText(this.mboundView3, (java.lang.CharSequence) userMapOkey1);
        }
    }
```

　　对于ObservableArrayMap类型，可以直接在布局文件表达式中使用key进行引用

```xml
<TextView android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="@{user.map.okey1}"/>
```
