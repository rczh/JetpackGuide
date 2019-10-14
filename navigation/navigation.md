# navigation
## 1.定义
navigation用来在应用程序的各个fragment目的地之间进行切换

## 2.作用
navigation主要用来封装fragment切换的事物操作

navigation组件被设计用来在拥有一个activity和多个fragment的程序中使用。对于拥有多个activity的程序，每一个activity应该包含一个navigation组件

## 3.主要组成部分
* ### Navigation graph
Navigation graph是一个包含程序中所有fragment目的地和action路径的xml资源文件

* ### NavHost
NavHost是一个空的容器用来显示fragment目的地，navigation组件包含一个默认的NavHostFragment实现

* ### NavController
NavController用来管理NavHost中各个fragment目的地之间的导航

## 4.实现原理
* ### 创建NavHostFragment

```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <fragment
        android:id="@+id/nav_host_fragment"
        <!--定义NavHost实现类-->      
        android:name="androidx.navigation.fragment.NavHostFragment"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        <!--defaultNavHost定义NavHostFragment类拦截系统back键-->      
        app:defaultNavHost="true"
         <!--定义navigation资源文件--> 
        app:navGraph="@navigation/nav_graph" />

</RelativeLayout>
```

NavHostFragment实现了NavHost接口，它作为一个容器用来切换各个fragment目的地

* ### 创建Navigation graph


