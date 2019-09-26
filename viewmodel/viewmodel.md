# viewmodel
## 1.定义
viewmodel用来存储数据，它是生命周期感知的

## 2.作用
viewmodel用来将数据从activity中分离出来。viewmodel能够保证当旋转屏幕activity触发重建时，viewmodel里面的数据保持不变

## 3.实现原理
系统提供ViewModelProvider类用来获取viewmodel对象，ViewModelProvider的初始化方法需要传入activity对象，activity中会保存一个ViewModelStore对象，viewmodel保存在ViewModelStore对象的集合中

ComponentActivity构造函数中会创建LifecycleEventObserver对象来观察activity的生命周期状态，当activity由于屏幕旋转触发销毁时系统不会执行ViewModelStore的clear方法，从而实现activity转屏重建但是viewmodel保留的功能



```java
    public static ViewModelProvider of(@NonNull FragmentActivity activity,
            @Nullable Factory factory) {
        Application application = checkApplication(activity);
        if (factory == null) {
            factory = ViewModelProvider.AndroidViewModelFactory.getInstance(application);
        }
        return new ViewModelProvider(activity.getViewModelStore(), factory);
    }
```

每一个activity中都会保存一个ViewModelStore对象，ViewModelStore对象中持有viewmodel集合

```java
    public ViewModelStore getViewModelStore() {
        if (getApplication() == null) {
            throw new IllegalStateException("Your activity is not yet attached to the "
                    + "Application instance. You can't request ViewModel before onCreate call.");
        }
        if (mViewModelStore == null) {
            NonConfigurationInstances nc =
                    (NonConfigurationInstances) getLastNonConfigurationInstance();
            if (nc != null) {
                //转屏时activity会重新创建，这时从rc中获取转屏之前activity中的viewModelStore对象
                mViewModelStore = nc.viewModelStore;
            }
            if (mViewModelStore == null) {
                mViewModelStore = new ViewModelStore();
            }
        }
        return mViewModelStore;
    }
```

当旋转屏幕时不会执行clear方法清理ViewModelStore对象中的viewmodel集合

```java
        getLifecycle().addObserver(new LifecycleEventObserver() {
            @Override
            public void onStateChanged(@NonNull LifecycleOwner source,
                    @NonNull Lifecycle.Event event) {
                if (event == Lifecycle.Event.ON_DESTROY) {
                    //当旋转屏幕时不会执行clear方法清理ViewModelStore对象中的viewmodel集合
                    if (!isChangingConfigurations()) {
                        getViewModelStore().clear();
                    }
                }
            }
        });
```        

## 4. viewmodel的生命周期

