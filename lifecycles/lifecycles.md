# lifecycles

## 1.  定义
　　lifecycles用来创建生命周期感知组件，它可以执行操作以响应Activity或者Fragment生命周期状态的改变

## 2. 作用
　　生命周期感知组件用来将Activity或者Fragment生命周期方法中的组件代码移动到组件本身的实现中，从而避免在Activity或者Fragment生命周期方法中加入过多的实现

## 3. 实现原理
### Lifecycles
　　lifecycles类的实现基于观察者模式。lifecycles类会保存Activity或者Fragment生命周期状态的信息，并且通过观察者模式允许其他对象观察这个状态信息

　　lifecycles使用两个枚举类型Event和State来跟踪Activity或者Fragment生命周期的状态

+ Event

　　用来表示lifecycles事件，这些事件映射到Activity或者Fragment中的回调事件
+ State

　　用来表示lifecycles跟踪的Activity或者Fragment的当前状态
   
   
   ![](https://github.com/rczh/AndroidJetpackGuide/blob/master/lifecycles/lifecycle-states.svg) 

### LifecycleObserver
　　Activity或者Fragment的生命周期观察者对象需要实现LifecycleObserver接口，并且通过OnLifecycleEvent注解方法来响应Activity或者Fragment的生命周期状态

### LifecycleOwner
　　Activity或者Fragment生命周期被观察者对象需要实现LifecycleOwner接口，该接口只有一个getLifecycle()方法返回Lifecycles对象。生命周期观察者对象需要调用Lifecycles对象的addObserver进行注册

　　自从26.1.0开始，Support Library中实现的Activity或者Fragment已经实现了LifecycleOwner接口，而对于标准库中的Activity或者Fragment则需要自己实现LifecycleOwner接口

## 4. 实例
* ### 使用AppCompatActivity

```kotlin
class MyActivity : AppCompatActivity() {
    private lateinit var myLocationListener: MyLocationListener

    override fun onCreate(...) {
        myLocationListener = MyLocationListener(this, lifecycle) { location ->
            // update UI
        }
        lifecycle.addObserver(myLocationListener)
        Util.checkUserStatus { result ->
            if (result) {
                myLocationListener.enable()
            }
        }
    }
}
```

　　在MyActivity的onCreate方法中初始化MyLocationListener对象，这里使用checkUserStatus方法模拟耗时操作。


```kotlin
internal class MyLocationListener(
        private val context: Context,
        private val lifecycle: Lifecycle,
        private val callback: (Location) -> Unit
) : LifecycleObserver{

    private var enabled = false

    @OnLifecycleEvent(Lifecycle.Event.ON_START)
    fun start() {
        if (enabled) {
            // connect
        }
    }

    fun enable() {
        enabled = true
        if (lifecycle.currentState.isAtLeast(Lifecycle.State.STARTED)) {
            // connect if not connected
        }
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_STOP)
    fun stop() {
        // disconnect if connected
    }
}
```

　　MyLocationListener是生命周期感知的，当被观察者的生命周期状态发生变化时MyLocationListener能够自动做出响应，从而避免在被观察者的生命周期方法中加入MyLocationListener方法调用

　　MyLocationListener类中可以使用lifecycle.currentState来查询被观察者的当前状态。如果checkUserStatus耗时方法在start方法之后执行完成，在enable方法中查询如果被观察者当前状态已经是STARTED，可以重新执行connect操作

* ### 自定义LifecycleOwner

```kotlin
class MyActivity : Activity(), LifecycleOwner {

    private lateinit var lifecycleRegistry: LifecycleRegistry

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        lifecycleRegistry = LifecycleRegistry(this)
        lifecycleRegistry.markState(Lifecycle.State.CREATED)
    }

    public override fun onStart() {
        super.onStart()
        lifecycleRegistry.markState(Lifecycle.State.STARTED)
    }

    override fun getLifecycle(): Lifecycle {
        return lifecycleRegistry
    }
}
```

　　使用LifecycleRegistry实现被观察者生命周期时需要手动设置生命周期状态
