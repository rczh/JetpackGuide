# lifecycles

## 1. 定义
lifecycles用来创建生命周期感知组件，它可以执行操作以响应Activity或者Fragment生命周期状态的改变

## 2.作用
生命周期感知组件用来将Activity或者Fragment生命周期方法中的组件代码移动到组件本身的实现中，从而避免在Activity或者Fragment生命周期方法中加入过多的实现

## 3. 实现原理
### lifecycles
lifecycles类的实现基于观察者模式。lifecycles类会保存Activity或者Fragment生命周期状态的信息，并且通过观察者模式允许其他对象观察这个状态信息。

lifecycles使用两个枚举类型Event和State来跟踪Activity或者Fragment生命周期的状态

+ Event

   用来表示lifecycles事件，这些事件映射到Activity或者Fragment中的回调事件
+ State

   用来表示lifecycles跟踪的Activity或者Fragment的当前状态
   
   
   ![](https://github.com/rczh/AndroidJetpackGuide/blob/master/lifecycles/lifecycle-states.svg) 

### LifecycleObserver
### LifecycleOwner
