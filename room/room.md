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





