# viewmodel
## 1.定义
　　viewmodel用来存储数据，它是生命周期感知的

## 2.作用
　　viewmodel用来将数据从activity中分离出来。viewmodel能够保证当旋转屏幕时activity触发重建，viewmodel里面的数据保持不变
  
　　通常将livedata存放在viewmodel中使用

## 3.实现原理
　　系统提供ViewModelProvider类用来获取viewmodel对象，ViewModelProvider的初始化方法需要传入activity对象，activity中会保存一个ViewModelStore对象，viewmodel保存在ViewModelStore对象的集合中

　　ComponentActivity构造函数中会创建LifecycleEventObserver对象来观察activity的生命周期状态，当activity由于屏幕旋转触发销毁时系统不会执行ViewModelStore的clear方法，从而实现activity转屏重建时仍然能够保留viewmodel的功能

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

　　转屏时activity会被重新创建，这时从rc中获取转屏之前activity中的viewModelStore对象，从而使新创建的activity和之前activity中的viewModelStore对象建立对应关系

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

　　当转屏重建activity时不会执行clear方法清理ViewModelStore对象中的viewmodel集合，从而使得ViewModelStore中的viewmodel得以保留

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

## 4. 生命周期
　　viewmodel的生命周期取决于初始化viewmodel时传入的activity生命周期，viewmodel会一直保存在内存中直到activity被销毁，这时系统调用ViewModelStore对象中clear方法清理当前activity下面的所有viewmodel对象

　　当转屏重建activity时，系统不会清理掉activity下面的viewmodel对象

　　注意：viewmodel必须不能引用activity对象！如果viewmodel引用activity对象会造成activity在转屏重建后不能被垃圾回收，导致内存泄露

   ![](https://github.com/rczh/JetpackGuide/blob/master/viewmodel/viewmodel-lifecycle.png) 
   
 ## 5.实例
* ### Activity中获取viewmodel
   
```kotlin
   class MyViewModel : ViewModel() {
    private val users: MutableLiveData<List<User>> by lazy {
        MutableLiveData<List<User>>().also {
            it.value = loadUsers()
        }
    }

    fun getUsers(): LiveData<List<User>> {
        return users
    }

    private fun loadUsers() : List<User>{
        return listOf(User("name1"), User("name2"))
    }
}
```
   
　　多次调用ViewModelProviders.of方法将获取同一个ViewModel对象
   
```kotlin
   class MyActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // Create a ViewModel the first time the system calls an activity's onCreate() method.
        // Re-created activities receive the same MyViewModel instance created by the first activity.
        val model = ViewModelProviders.of(this)[MyViewModel::class.java]
        model.getUsers().observe(this, Observer<List<User>>{ users ->
            // update UI
        })
    }
}
```

* ### fragment之间共享viewmodel对象

```kotlin
class SharedViewModel : ViewModel() {
    val selected = MutableLiveData<Item>()

    fun select(item: Item) {
        selected.value = item
    }
}

class MasterFragment : Fragment() {

    private lateinit var itemSelector: Selector
    private lateinit var model: SharedViewModel

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        model = activity?.run {
            ViewModelProviders.of(this)[SharedViewModel::class.java]
        } ?: throw Exception("Invalid Activity")
        itemSelector.setOnClickListener { item ->
            // Update the UI
            model.select(item)
        }
    }
}

class DetailFragment : Fragment() {

    private lateinit var model: SharedViewModel

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        model = activity?.run {
            ViewModelProviders.of(this)[SharedViewModel::class.java]
        } ?: throw Exception("Invalid Activity")
        model.selected.observe(this, Observer<Item> { item ->
            // Update the UI
            println(item)
        })
    }
}
```
  
　　activity不需要负责在多个fragment之间传递数据，每个fragment获取同一个viewmodel对象，多个fragment之间不需要相互引用
