# navigation
## 1.定义
navigation用来在应用程序的各个fragment之间进行切换

## 2.作用
navigation主要用来封装fragment切换的事物操作

navigation组件被设计用来在拥有一个activity和多个fragment的程序中使用，对于拥有多个activity的程序，每一个activity应该包含一个navigation组件

navigation组件允许将actiivty作为导航目的地，可以实现从fragment导航到activity

## 3.组成部分
* ### Navigation graph
Navigation graph是一个包含程序中所有fragment和action路径的xml资源文件

* ### NavHost
NavHost是一个空的容器用来显示fragment，navigation组件包含一个默认的NavHost接口实现NavHostFragment

* ### NavController
NavController用来管理NavHost中各个fragment之间的导航

## 4.实现原理
* ### 创建NavHost

android:name用来定义NavHost的实现类

app:navGraph用来定义navigation资源文件

app:defaultNavHost表示NavHostFragment响应系统back键

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
        android:name="androidx.navigation.fragment.NavHostFragment"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:defaultNavHost="true"
        app:navGraph="@navigation/nav_graph" />

</RelativeLayout>
```

navigation提供了NavHost接口的默认实现NavHostFragment，它作为一个容器用来切换各个fragment目的地

* ### 创建Navigation graph
app:startDestination用来定义navigation组件启动时的目的地

action用来定义fragment之间的跳转路径

```xml
<?xml version="1.0" encoding="utf-8"?>
<navigation xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/nav_graph"
    app:startDestination="@id/blankFragment">

    <fragment
        android:id="@+id/blankFragment"
        android:name="com.example.navgiation.BlankFragment"
        tools:layout="@layout/fragment_nav_blank"
        android:label="BlankFragment" >
        <action
            android:id="@+id/action_blankFragment_to_blankFragment2"
            app:destination="@id/blankFragment2" />
    </fragment>
    <fragment
        android:id="@+id/blankFragment2"
        android:name="com.example.navgiation.BlankFragment2"
        android:label="BlankFragment2" />
</navigation>
```

NavHostFragment使用ft.replace方法来切换fragment，并且通过ft.addToBackStack方法将fragment入栈来实现回退操作。与此同时，FragmentNavigator和NavController内部各自维护一个栈，用来处理回退逻辑

```java
public NavDestination navigate(@NonNull Destination destination, @Nullable Bundle args,
            @Nullable NavOptions navOptions, @Nullable Navigator.Extras navigatorExtras) {

       //......  
        final FragmentTransaction ft = mFragmentManager.beginTransaction();
        //使用replace方法切换fragment
        ft.replace(mContainerId, frag);
        ft.setPrimaryNavigationFragment(frag);

        final @IdRes int destId = destination.getId();
        final boolean initialNavigation = mBackStack.isEmpty();
        // TODO Build first class singleTop behavior for fragments
        final boolean isSingleTopReplacement = navOptions != null && !initialNavigation
                && navOptions.shouldLaunchSingleTop()
                && mBackStack.peekLast() == destId;

        boolean isAdded;
        if (initialNavigation) {
            isAdded = true;
        } else if (isSingleTopReplacement) {
            // Single Top means we only want one instance on the back stack
            if (mBackStack.size() > 1) {
                // If the Fragment to be replaced is on the FragmentManager's
                // back stack, a simple replace() isn't enough so we
                // remove it from the back stack and put our replacement
                // on the back stack in its place
                mFragmentManager.popBackStack(
                        generateBackStackName(mBackStack.size(), mBackStack.peekLast()),
                        FragmentManager.POP_BACK_STACK_INCLUSIVE);
                ft.addToBackStack(generateBackStackName(mBackStack.size(), destId));
            }
            isAdded = false;
        } else {
            //通过将fragment加入到ft的BackStack中来实现回退操作
            ft.addToBackStack(generateBackStackName(mBackStack.size() + 1, destId));
            isAdded = true;
        }
        if (navigatorExtras instanceof Extras) {
            Extras extras = (Extras) navigatorExtras;
            for (Map.Entry<View, String> sharedElement : extras.getSharedElements().entrySet()) {
                //可以添加shareElement过场动画
                ft.addSharedElement(sharedElement.getKey(), sharedElement.getValue());
            }
        }
        ft.setReorderingAllowed(true);
        ft.commit();
        // The commit succeeded, update our view of the world
        if (isAdded) {
            //FragmentNavigator中维护一个栈，用来处理回退逻辑
            mBackStack.add(destId);
            return destination;
        } else {
            return null;
        }
    }
    
    @Override
    public boolean popBackStack() {
        if (mBackStack.isEmpty()) {
            return false;
        }
        if (mFragmentManager.isStateSaved()) {
            Log.i(TAG, "Ignoring popBackStack() call: FragmentManager has already"
                    + " saved its state");
            return false;
        }
        //出栈时执行FragmentManager.popBackStack方法
        mFragmentManager.popBackStack(
                generateBackStackName(mBackStack.size(), mBackStack.peekLast()),
                FragmentManager.POP_BACK_STACK_INCLUSIVE);
        mBackStack.removeLast();
        return true;
    }
    
```

注意：由于NavHostFragment使用ft.replace方法来切换fragment，每次执行replace方法时都会调用fragment.onCreateView方法。对于那些需要执行ft.hide，ft.show方法来保留原有fragment状态的需求，navigation组件并不适用

* ### 使用NavController执行导航操作

navigation组件使用NavController.navigate方法来实现具体的导航操作

```kotlin
        textView.setOnClickListener {
            var bundle = bundleOf("myname1" to "robin")
            val extras = FragmentNavigatorExtras(textView to "name")
            it.findNavController().navigate(R.id.action_blankFragment_to_blankFragment2, bundle, null, extras)
        }
```

NavController对象的获取方式：
* Fragment.findNavController()
* View.findNavController()
* Activity.findNavController(viewId: Int)

## 5.Nested graphs和Global action

对于那些只在某些情况下对用户可见的流程，比如登录流程，可以将其单独作为一个navigation嵌套在父navigation中，或者生成一个独立的navigation然后通过include方式导入到父navigation中

对于一个拥有多条action路径入口的fragment来说，可以创建一个单独的Global action，在每一条action路径入口上的fragment可以使用这个Global action导航到目标fragment

```xml
<!-- 由多个源跳转到同一个目的地时使用global action-->
    <action android:id="@+id/action_pop_out_of_game"
        app:popUpTo="@id/in_game_nav_graph"
        app:popUpToInclusive="true"  />
```

## 6.popUpTo和popUpToInclusive属性
action支持popUpTo和popUpToInclusive属性，假设navigation回退栈中已经包含a->b->c，如果从c导航到a时使用app:popUpTo="@+id/a"，回退栈会弹出c和b直到a，然后再入栈a，最后回退栈中会包含两个a。如果使用app:popUpToInclusive="true"，回退栈会弹出c，b和a，然后再入栈a，最后回退栈中只包含一个a

```xml
<action
        android:id="@+id/action_c_to_a"
        app:destination="@id/a"
        app:popUpTo="@+id/a"
        app:popUpToInclusive="true"/>
```

## 7.两种fragment之间传递数据方式
* ### 使用bundle
在调用navigation方法时将bundle作为参数

```kotlin
var bundle = bundleOf("amount" to amount)
view.findNavController().navigate(R.id.confirmationAction, bundle)
```

目标fragment使用arguments读取bundle值

```kotlin
val tv = view.findViewById<TextView>(R.id.textViewAmount)
tv.text = arguments.getString("amount")
```

* ### 使用Safe Args
navigation组件提供Safe Args gradle插件用来生成特定类来传递参数








