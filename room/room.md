# Room
## 定义
Room对sqlite进行了封装，用来简化数据库的操作

## 基本组件
* Database：用来表示数据库类。使用@Database标记

* Entity：用来表示实体类，它对应数据库中的一张表。使用@Entity标记

* Dao：包含一系列访问数据库的方法。使用@Dao标记

组件之间的对应关系:

![](https://github.com/rczh/JetpackGuide/blob/master/room/v2-6cd8950561fafc6cb5f3307175b1513f_720w.jpg) 

## 注解
### @Database
用来标记数据库类，该类需要满足以下条件：

* 必须继承于RoomDatabase抽象类

* 包含返回Dao类的抽象方法

* 注解中需要定义与数据库相关联的实体类列表

```kotlin
@Database(entities = [User::class], version = 1)
abstract class AppDatabase : RoomDatabase() {

    abstract fun userDao(): UserDao

    companion object {
        private var instance: AppDatabase? = null
        fun getInstance(context: Context): AppDatabase {
            if (instance == null) {
                instance = Room.databaseBuilder(
                    context.applicationContext,
                    AppDatabase::class.java,
                    "user.db" //数据库名称
                ).allowMainThreadQueries().build()
            }
            return instance as AppDatabase
        }
    }
}
```

使用Room.databaseBuilder构造数据库实例。由于构造过程非常耗时，需要使用单例模式

### @Entity
使用Entity定义的类会被映射为数据库中的一张表

```kotlin
@Entity(tableName = "USERS")
data class User(
    @PrimaryKey(autoGenerate = true) var userId: Long = 0,
    @ColumnInfo(name = "user_name")var userName: String = "",
    @ColumnInfo(name = "user_phone")var userPhone: String = "",
    @ColumnInfo(defaultValue = "china") var address: String = "",
    @Ignore var sex: String = ""
)
```

### @Dao
Dao类是一个接口，定义了一系列访问数据库的方法

```kotlin
@Dao
interface UserDao {
    @Query("select * from USERS where userId = :id")
    fun getUserById(id: Long): User
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    fun addUser(user: User)
    @Update
    fun updateUserByUser(user: User)
    @Delete
    fun deleteUserByUser(user: User)
}
```

## 实现原理
Room会为AppDatabase生成实现类AppDatabase_Impl，其中包含UserDao和SupportSQLiteOpenHelper对象

```java
public final class AppDatabase_Impl extends AppDatabase {
  private volatile UserDao _userDao;

  @Override
  protected SupportSQLiteOpenHelper createOpenHelper(DatabaseConfiguration configuration) {...}
  ...
  @Override
  public UserDao userDao() {
    if (_userDao != null) {
      return _userDao;
    } else {
      synchronized(this) {
        if(_userDao == null) {
          _userDao = new UserDao_Impl(this);
        }
        return _userDao;
      }
    }
  }
}
```

通过建造者模式实例化AppDatabase_Impl对象，并且执行init方法创建SupportSQLiteOpenHelper对象

```java
public abstract class RoomDatabase {
    private SupportSQLiteOpenHelper mOpenHelper;
    public T build() {
                ...
                T db = Room.getGeneratedImplementation(mDatabaseClass, DB_IMPL_SUFFIX);
                db.init(configuration);
                return db;
            }
            
    @CallSuper
        public void init(@NonNull DatabaseConfiguration configuration) {
            mOpenHelper = createOpenHelper(configuration);
            ...
        }
    }

```

Room为UserDao生成实现类UserDao_Impl，该类中实现了addUser方法

```java
public final class UserDao_Impl implements UserDao {
  private final RoomDatabase __db;
  private final EntityInsertionAdapter<User> __insertionAdapterOfUser;

  public UserDao_Impl(RoomDatabase __db) {
    this.__db = __db;
    this.__insertionAdapterOfUser = new EntityInsertionAdapter<User>(__db) {...}
  ...
  @Override
  public void addUser(final User user) {
    __db.assertNotSuspendingTransaction();
    __db.beginTransaction();
    try {
      __insertionAdapterOfUser.insert(user);
      __db.setTransactionSuccessful();
    } finally {
      __db.endTransaction();
    }
  }

}
```

addUser方法中使用RoomDatabase.mOpenHelper对象实现事务处理

```java
public abstract class RoomDatabase {
    public void beginTransaction() {
        assertNotMainThread();
        SupportSQLiteDatabase database = mOpenHelper.getWritableDatabase();
        mInvalidationTracker.syncTriggers(database);
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN
                && database.isWriteAheadLoggingEnabled()) {
            database.beginTransactionNonExclusive();
        } else {
            database.beginTransaction();
        }
    }
}

```

addUser方法中使用EntityInsertionAdapter.insert方法实现插入操作

```java
public abstract class EntityInsertionAdapter<T> extends SharedSQLiteStatement {
    public final void insert(T entity) {
        final SupportSQLiteStatement stmt = acquire();
        try {
            bind(stmt, entity);
            stmt.executeInsert();
        } finally {
            release(stmt);
        }
    }
}
```

bind方法用来将参数保存到mBindArgs数组中

```java
public final class UserDao_Impl implements UserDao {
  public UserDao_Impl(RoomDatabase __db) {
    this.__insertionAdapterOfUser = new EntityInsertionAdapter<User>(__db) {

      @Override
      public void bind(SupportSQLiteStatement stmt, User value) {
        stmt.bindLong(1, value.getUserId());
        if (value.getUserName() == null) {
          stmt.bindNull(2);
        } else {
          stmt.bindString(2, value.getUserName());
        }
      }
    }
}

public abstract class SQLiteProgram extends SQLiteClosable {
    private void bind(int index, Object value) {
            if (index < 1 || index > mNumParameters) {
                throw new IllegalArgumentException("Cannot bind argument at index "
                        + index + " because the index is out of range.  "
                        + "The statement has " + mNumParameters + " parameters.");
            }
            mBindArgs[index - 1] = value;
        }
｝
```

executeInsert方法用来执行具体sql语句

```java
public final class SQLiteStatement extends SQLiteProgram {
    public long executeInsert() {
        acquireReference();
        try {
            return getSession().executeForLastInsertedRowId(
                    getSql(), getBindArgs(), getConnectionFlags(), null);
        } catch (SQLiteDatabaseCorruptException ex) {
            onCorruption();
            throw ex;
        } finally {
            releaseReference();
        }
    }
}
```

