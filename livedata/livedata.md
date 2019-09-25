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

* ### 扩展livedata

```kotlin
class StockLiveData(symbol: String) : LiveData<BigDecimal>() {
    private val stockManager: StockManager =
        StockManager(symbol)

    private val listener = { price: BigDecimal ->
        value = price
    }

    override fun onActive() {
        stockManager.requestPriceUpdates(listener)
    }

    override fun onInactive() {
        stockManager.removeUpdates(listener)
    }

    companion object {
        private lateinit var sInstance: StockLiveData
        //创建一个StockLiveData单例对象
        @MainThread
        fun get(symbol: String): StockLiveData {
            sInstance = if (Companion::sInstance.isInitialized) sInstance else StockLiveData(
                symbol
            )
            return sInstance
        }
    }
}
```

　　当livedata有一个处于活动状态的观察者对象时调用onActive方法，可以在该方法中实现初始化操作

　　当livedata没有任何处于活动状态的观察者对象时调用onInactive方法，可以在该方法中实现清理操作


```kotlin
class MyFragment : Fragment() {
    override fun onActivityCreated(savedInstanceState: Bundle?) {
        super.onActivityCreated(savedInstanceState)
        //Multiple fragments and activities can observe the livedata instance
        StockLiveData.get("symbol")
            .observe(this, Observer<BigDecimal> { price: BigDecimal? ->
            // Update the UI.
        })
    }
}
```

　　livedata是生命周期感知组件，可以在多个activity或者fragment之间共享

* ### livedata变换
#### Transformations.map()

```kotlin
        val userLiveData: MutableLiveData<User> = MutableLiveData<User>()
        val userName: LiveData<String> = Transformations.map(userLiveData) {
                user -> "${user.name} ${user.lastName}"
        }
        userName.observe(this, Observer {
            println("userName............: $it")
        })
        mapTextView.setOnClickListener {
            userLiveData.value = User("name", "last name")
        }
```

　　map变换用来将原始的livedata泛型对象转换成新的livedata泛型，当原始livedata对象数据发生变化时新的livedata对象也能够收到通知

```kotlin
    public static <X, Y> LiveData<Y> map(
            @NonNull LiveData<X> source,
            @NonNull final Function<X, Y> mapFunction) {
        final MediatorLiveData<Y> result = new MediatorLiveData<>();
        result.addSource(source, new Observer<X>() {
            @Override
            public void onChanged(@Nullable X x) {
                result.setValue(mapFunction.apply(x));
            }
        });
        return result;
    }
```

　　map变换的实现基于MediatorLiveData。MediatorLiveData继承于LiveData，它会在原始livedata对象上注册新的观察者，当原始livedata对象数据发生变化时MediatorLiveData会回调mapFunction方法，然后将返回结果作为新的MediatorLiveData数据值更新，最后通知MediatorLiveData的观察者

　　MediatorLiveData可以用来实现同时观察多个livedata数据源，当任何一个livedata数据发生变换时MediatorLiveData都可以触发通知

#### Transformations.switchMap()

```kotlin
        val userId: MutableLiveData<String> = MutableLiveData<String>()
        val user: LiveData<User> = Transformations.switchMap(userId) {
                id ->
            println("userId changed")
            val result = getUser(id)
            result.value = User("name", "lastName")
            result
        }
        user.observe(this, Observer {
            println("user............: $it")
        })
        switchMapTextView.setOnClickListener {
            userId.value = "new userId"
        }
        
        
    private fun getUser(id: String): MutableLiveData<User> {
        return MutableLiveData<User>()
    }
```

　　switchMap变化的实现基于MediatorLiveData，用来观察getUser函数返回的livedata值的变化。当userId变化时switchMap会回调并且执行getUser函数，如果getUser函数返回新的livedata对象，MediatorLiveData会重新注册观察新的livedata对象
  
  ```kotlin
      public static <X, Y> LiveData<Y> switchMap(
            @NonNull LiveData<X> source,
            @NonNull final Function<X, LiveData<Y>> switchMapFunction) {
        final MediatorLiveData<Y> result = new MediatorLiveData<>();
        result.addSource(source, new Observer<X>() {
            LiveData<Y> mSource;

            @Override
            public void onChanged(@Nullable X x) {
                LiveData<Y> newLiveData = switchMapFunction.apply(x);
                if (mSource == newLiveData) {
                    return;
                }
                if (mSource != null) {
                    result.removeSource(mSource);
                }
                mSource = newLiveData;
                if (mSource != null) {
                    result.addSource(mSource, new Observer<Y>() {
                        @Override
                        public void onChanged(@Nullable Y y) {
                            result.setValue(y);
                        }
                    });
                }
            }
        });
        return result;
    }
  ```

