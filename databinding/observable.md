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

* ### Observable字段类型