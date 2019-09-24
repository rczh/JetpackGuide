# livedata

## 1.定义
livedata是一个可以被观察的数据持有化类。livedata是生命周期感知的，它只通知处于活动生命周期状态的观察者组件更新

## 2. 作用
livedata用来持有数据，当数据发生变化时livedata通知处于活动生命周期状态的观察者。与此同时，livedata也会监控观察者的生命周期状态，当它处于DESTROYED状态时livedata会自动移除观察者，从而避免内存溢出的情况

## 3.实现原理
livedata类的实现基于观察者模式。观察者组件需要实现Observer接口，然后通过observe方法在livedata类中进行注册，observe方法需要传入一个LifecycleOwner对象参数，livedata会监控这个LifecycleOwner对象的生命周期状态，当LifecycleOwner对象的生命周期状态变成DESTROYED时livedata自动移除该观察者

```java
    @MainThread
    public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
        assertMainThread("observe");
        if (owner.getLifecycle().getCurrentState() == DESTROYED) {
            return;
        }
        //LifecycleBoundObserver类实现了LifecycleObserver接口，用来观察LifecycleOwner对象的生命周期
        LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
        //mObservers集合中持有wrapper对象，当livedata中数据发生变化时会遍历mObservers集合并且执行observer.onChange方法
        ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
        if (existing != null && !existing.isAttachedTo(owner)) {
            throw new IllegalArgumentException("Cannot add the same observer"
                    + " with different lifecycles");
        }
        if (existing != null) {
            return;
        }
        //注册观察者
        owner.getLifecycle().addObserver(wrapper);
    }
```

LifecycleBoundObserver类实现了LifecycleObserver接口，用来观察LifecycleOwner对象的生命周期，当LifecycleOwner对象的生命周期状态变成DESTROYED时自动移除该观察者

```java
        @Override
        public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
            if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
                removeObserver(mObserver);
                return;
            }
            activeStateChanged(shouldBeActive());
        }
```

当更新livedata中的值时遍历mObservers集合，并且执行observer.onChange方法

```java
    @MainThread
    protected void setValue(T value) {
        assertMainThread("setValue");
        mVersion++;
        mData = value;
        dispatchingValue(null);
    }
```

## 4.实例
* ### 使用Activity作为LifecycleOwner

```kotlin
class LiveDataActivity : AppCompatActivity() {

    private var mCustomLiveData: CustomLiveData = CustomLiveData()
    private lateinit var mNameTextView: TextView
    private lateinit var mButton : Button

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        setContentView(R.layout.activity_livedata_main)
        mNameTextView = findViewById(R.id.text_view)
        mButton = findViewById(R.id.button)
        mButton.setOnClickListener {
            //setValue方法必须在主线程中执行
            mCustomLiveData.currentNameLiveData.value = "new value"
        }

        val nameObserver = Observer<String> { newName ->
            mNameTextView.text = newName
        }

        // Observe the LiveData, passing in this activity as the LifecycleOwner and the observer.
        mCustomLiveData.currentNameLiveData.observe(this, nameObserver)
    }
}
```

livedata中的setVaule方法必须在主线程中执行。如果需要在工作线程中更新值应该使用postValue方法

```kotlin
class CustomLiveData {

    val currentNameLiveData: MutableLiveData<String> by lazy {
        MutableLiveData<String>().also {
            it.value = "init"
        }
    }

}
```

通过懒加载方式初始化livedata对象，在使用时初始化





