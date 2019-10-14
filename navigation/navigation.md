# navigation
## 1.定义
navigation用来在应用程序的各个fragment目的地之间进行切换

## 2.作用
navigation主要用来封装fragment切换的事物操作

navigation组件被设计用来在拥有一个activity和多个fragment的程序中使用。对于拥有多个activity的程序，每一个activity应该包含一个navigation组件

## 3.组成部分
* ### Navigation graph
Navigation graph是一个包含程序中所有fragment目的地和action路径的xml资源文件

* ### NavHost
NavHost是一个空的容器用来显示fragment目的地，navigation组件包含一个默认的NavHostFragment实现

* ### NavController
NavController用来管理NavHost中各个fragment目的地之间的导航

## 4.实现原理
* ### 创建NavHostFragment

android:name用来定义NavHost的实现类

app:navGraph用来定义navigation资源文件

app:defaultNavHost表示NavHost的实现类响应系统back键

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

NavHostFragment实现了NavHost接口，它作为一个容器用来切换各个fragment目的地

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

NavHostFragment使用ft.replace来切换fragment，并且通过ft.addToBackStack实现回退操作。navigation组件内部自己维护一个栈，用来处理回退逻辑

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
            //navigation中维护一个栈，用来判断弹出操作
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
        mFragmentManager.popBackStack(
                generateBackStackName(mBackStack.size(), mBackStack.peekLast()),
                FragmentManager.POP_BACK_STACK_INCLUSIVE);
        mBackStack.removeLast();
        return true;
    }
    
```

注意：由于NavHostFragment使用replace来切换fragment，每次执行replace方法时都会调用fragment的onCreateView方法。对于那些需要执行hide，show来保留原有fragment状态的情况，navigation组件并不适用

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
