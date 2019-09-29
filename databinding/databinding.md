# databinding
## 1.定义
databinding使用声明形式的表达式语言将布局中的UI组件绑定到数据源

## 2.作用
databinding用来删除activity中的findViewById调用，使activity更加简单且易于维护

## 3.实现原理
定义一个databinding布局文件，布局文件以layout作为根节点，包括一个data节点和一个view layout节点。view layout节点中的UI组件通过表达式语言引用data节点中定义的变量

```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android">
    <data>
        <variable name="handlers" type="com.example.MyHandlers"/>
        <variable name="user" type="com.example.User"/>
        <variable name="task" type="com.example.Task" />
        <variable name="presenter" type="com.example.Presenter" />
    </data>
    <LinearLayout
        android:orientation="vertical"
        android:layout_width="match_parent"
        android:layout_height="match_parent">
        <TextView android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:id="@+id/firstNameId"
            android:textSize="18dp"
            android:onClick="@{handlers::onClickFriend}"
            android:text="@{user.firstName}"/>
        <TextView android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:id="@+id/lastNameId"
            android:textSize="18dp"
            android:onClick="@{() -> presenter.onSaveClick(task)}"
            android:text="@{user.lastName}"/>
    </LinearLayout>
</layout>
```

databinding库为每一个布局文件自动生成将布局中的UI组件与数据对象绑定所需的绑定类，绑定类中保存所有包含表达式语言的UI组件和data变量的引用以及它们之间的绑定关系

```java
public abstract class ActivityBindingMainBinding extends ViewDataBinding {
  @NonNull
  public final TextView firstNameId;
  @NonNull
  public final TextView lastNameId;
  @Bindable
  protected MyHandlers mHandlers;
  @Bindable
  protected User mUser;
  @Bindable
  protected Task mTask;
  @Bindable
  protected Presenter mPresenter;
// ..........
}
```

通常在activity中使用DataBindingUtil.setContentView方法初始化绑定类，并且通过绑定类中的set方法初始化data变量

```kotlin
data class User(var firstName: String, var lastName: String)

class BindingActivity : AppCompatActivity() {
    lateinit var mRootView: LinearLayout

override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val binding: ActivityBindingMainBinding = DataBindingUtil.setContentView(
            this, R.layout.activity_binding_main)
        mRootView = binding.root as LinearLayout
        var user = User("Test", "User")
        binding.user = user
        binding.handlers = MyHandlers()
        binding.presenter = Presenter()
        binding.task = Task("task")
    }
}
```

DataBindingUtil.setContentView方法调用bind方法，bind方法用来建立layout布局文件和绑定类的对应关系

```java
    static <T extends ViewDataBinding> T bind(DataBindingComponent bindingComponent, View root, int layoutId) {
        return (T) sMapper.getDataBinder(bindingComponent, root, layoutId);
    }

@Override
  public ViewDataBinding getDataBinder(DataBindingComponent component, View view, int layoutId) {
    int localizedLayoutId = INTERNAL_LAYOUT_ID_LOOKUP.get(layoutId);
    if(localizedLayoutId > 0) {
      final Object tag = view.getTag();
      if(tag == null) {
        throw new RuntimeException("view must have a tag");
      }
      switch(localizedLayoutId) {
        case  LAYOUT_ACTIVITYBINDINGMAIN: {
          if ("layout/activity_binding_main_0".equals(tag)) {
            return new ActivityBindingMainBindingImpl(component, view);
          }
          throw new IllegalArgumentException("The tag for activity_binding_main is invalid. Received: " + tag);
        }
      }
    }
    return null;
  }
```

绑定类Impl中的executeBindings方法用来执行具体的绑定操作，将data变量的内容赋值给对应的UI组件

```java
@Override
    protected void executeBindings() {
        long dirtyFlags = 0;
        synchronized(this) {
            dirtyFlags = mDirtyFlags;
            mDirtyFlags = 0;
        }
        com.wisetv.shell.jetpackdemo.databinding.Presenter presenter = mPresenter;
        java.lang.String userFirstName = null;
        com.wisetv.shell.jetpackdemo.databinding.User user = mUser;
        android.view.View.OnClickListener handlersOnClickFriendAndroidViewViewOnClickListener = null;
        com.wisetv.shell.jetpackdemo.databinding.MyHandlers handlers = mHandlers;
        java.lang.String userLastName = null;
        com.wisetv.shell.jetpackdemo.databinding.Task task = mTask;

        if ((dirtyFlags & 0x12L) != 0) {
                if (user != null) {
                    // read user.firstName
                    userFirstName = user.getFirstName();
                    // read user.lastName
                    userLastName = user.getLastName();
                }
        }
        if ((dirtyFlags & 0x14L) != 0) {
                if (handlers != null) {
                    // read handlers::onClickFriend
                    handlersOnClickFriendAndroidViewViewOnClickListener = (((mHandlersOnClickFriendAndroidViewViewOnClickListener == null) ? (mHandlersOnClickFriendAndroidViewViewOnClickListener = new OnClickListenerImpl()) : mHandlersOnClickFriendAndroidViewViewOnClickListener).setValue(handlers));
                }
        }
        // batch finished
        if ((dirtyFlags & 0x14L) != 0) {
            // api target 1
            this.firstNameId.setOnClickListener(handlersOnClickFriendAndroidViewViewOnClickListener);
        }
        if ((dirtyFlags & 0x12L) != 0) {
            // api target 1
            androidx.databinding.adapters.TextViewBindingAdapter.setText(this.firstNameId, userFirstName);
            androidx.databinding.adapters.TextViewBindingAdapter.setText(this.lastNameId, userLastName);
        }
        if ((dirtyFlags & 0x10L) != 0) {
            // api target 1
            this.lastNameId.setOnClickListener(mCallback3);
        }
    }
```
