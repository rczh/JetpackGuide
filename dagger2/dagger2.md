# Dagger2
## 定义
* 控制反转(IOC)是面向对象编程中的一种设计原则，用来降低代码之间的耦合度
* 依赖注入是IOC的一种实现方式
* Dagger2是依赖注入的实现框架

### 依赖
如果类Ａ中包含类Ｂ的对象，则称类Ａ依赖于类Ｂ

```kotlin
class B
class A(val b: B = B())
```

缺点：如果程序中有多处使用了Ａ对象，每次初始化Ａ对象时都需要将Ｂ对象作为参数，这样增加了代码的复杂度和耦合度

### 注入
类Ａ中的Ｂ对象在运行时由框架进行实例化

```kotlin
class B
class A() {
    @Inject
    val b: B? = null
}
```

优点：每次初始化Ａ对象时不需要传任何参数，降低了代码的复杂度和耦合度

## 注解
Dagger2主要包含三个部分：依赖提供方、依赖需求方、依赖注入组件

### @Inject
Inject注解有两个作用
* 将Inject注解添加到类的构造函数上，作为依赖提供方

```kotlin
//Dagger2会使用Inject注解的构造函数来生成依赖对象
class LoginPresenter @Inject constructor(var iView: ICommonView) {
    fun login(user: User?) {
        val mContext = iView.context
        Toast.makeText(mContext, "login......", Toast.LENGTH_SHORT).show()
    }
}
```

* 将Inject注解添加到成员变量上，作为依赖需求方

```kotlin
class MainActivity : AppCompatActivity(){
    //Dagger2会为Inject注解的成员变量注入依赖对象
    @JvmField
    @Inject
    var presenter: LoginPresenter? = null
}
```

### @Module
Module注解的类作为依赖提供方，可以为构造函数提供参数

```kotlin
@Module
class CommonModule(private val iView: ICommonView) {
    //Dagger2使用Provides注解的方法为构造函数返回参数
    @Provides
    fun provideIcommonView(): ICommonView {
        return iView
    }
}
```

### @Component
Component用来注解接口，负责在依赖提供方和依赖需求方之间建立连接

```kotlin
//使用modules参数来指定相应的Module类
@Component(modules = [CommonModule::class])
interface CommonComponent {
    fun inject(activity: MainActivity?)
}
```

## 实现原理






