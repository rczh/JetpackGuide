# livedata
## 1.定义
livedata是一个可以被观察的数据持有化类。livedata是生命周期感知的，它只通知处于活动生命周期状态的观察者组件更新
## 2. 作用
livedata用来持有数据，当数据发生变化时livedata通知处于活动生命周期状态的观察者。与此同时，livedata也会监控观察者的生命周期状态，当它处于DESTROYED状态时livedata会自动移除观察者，从而避免内存溢出的情况
