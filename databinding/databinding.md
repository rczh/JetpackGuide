# databinding
## 1.定义
databinding使用声明形式的表达式语言将布局中的UI组件绑定到数据源

## 2.作用
databinding用来删除activity中的findViewById调用，使activity更加简单且易于维护

## 3.实现原理
定义一个databinding布局文件，布局文件以layout作为根节点，包括一个data节点和一个view layout节点。view layout节点中的UI组件通过表达式语言引用data节点中定义的变量

databinding库为每一个布局文件自动生成将布局中的UI组件与数据对象绑定所需的绑定类，绑定类中保存所有包含表达式语言的UI组件和data变量的引用以及它们之间的绑定关系
