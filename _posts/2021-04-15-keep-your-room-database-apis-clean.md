---
layout: post
title: Keep your Room database APIs clean
tags: android room database apis
---

I use [Room][room] in this post to illustrate how you can configure your internal APIs for database management in clean way, so that you get more stable, easy to use and easy to update framework. 

If you're not familiar with [Room][room], it's a popular library for Android that provides an abstraction layer over SQLite to allow fluent database access while harnessing the full power of SQLite. This post uses it to construct examples, but the topic itself is a bit broader and can be extended to other things like Firebase operations, or SQL operations using some other library, among other examples.

Also, this is just my approach. You might disagree or have something better, and that's fine.

## Room components

As you probably already know, in order to use Room, you need database, some models and DAOs (Data Access Objects) for them. So I'll go ahead and create some simple ones.

> Note: you'll notice I mark some classes as `internal`. It's because in my usual project setup, the database files are in separate module, and I mark the things I don't want to expose to other modules as `internal`.  We'll come back to this later.

### Database configuration

I like to keep my database configuration variables in a separate file so it's easier to see the whole structure. I call this class `RoomDatabaseConfig` and make it a singleton object:

```kotlin
internal object RoomDatabaseConfig {
  const val DB_NAME = "MyDatabase"
  const val DB_TEST_NAME = "MyTestDatabase"
  const val DB_VERSION = 1
	
  const val TABLE_USERS = "users"
}
```

### Model

For this example, let's use some stupid-simple model, e.g. `User` (like in the official documentation):

```kotlin
@Entity(tableName = RoomDatabaseConfig.TABLE_USERS)
data class User {
  @PrimaryKey(autogenerate = true) val id: Int,
  val firstName: String?,
  val lastName: String?
}
```

### DAO

In order to access this data, we need to create the DAO (Data Access Object) that will be able to perform defined database operations.

```kotlin
@Dao
internal interface UserDao {

  // Read.
	
  @Query("SELECT * FROM ${TABLE_USERS} WHERE id = :id LIMIT 1")
  suspend fun get(id: Int): User?
	
  @Query("SELECT * FROM ${TABLE_USERS}")
  suspend fun getAll(): List<User>
	
  // Write
	
  @Insert(onConflict = OnConflictStrategy.REPLACE)
  suspend fun insertOrUpdate(user: User)
	
  @Insert(onConflict = OnConflictStrategy.REPLACE)
  suspend fun inserOrUpdate(users: List<User>)
	
  // Delete
	
  @Delete
  suspend fun delete(user: User)

  @Query("DELETE FROM ${TABLE_USERS}")
  suspend fun deleteAll()
	
  // Observe changes.
	
  fun observe(): Flow<List<User>>

}
```

All the functions are marked with `suspend` because we'll use them in Kotlin coroutines.

> Small hint on naming: I usually like to keep function naming simple and precise, e.g. `insertOrUpdate()` is a bit more precise than something like `set()`. Also, I let the variable describe the data we're manipulating, so `get(): User` instead of `getUser(): User`. I will use full names later in higher-level APIs, but in the context of each DAO, this naming is shorter and more consistent. 

### Database

When initializing database, I usually provide functions for instantiating the production and test databases, along all the DAO getters.

```kotlin
@Database(
  entities = [User::class],
  version = RoomDatabaseConfig.DB_VERSION
)
abstract class MyDatabase : RoomDatabase {
  companion object {
  
    private var instance: MyDatabase? = null
    private var testInstance: MyDatabase? = null
		
    fun getInstance(ctx: Context): MyDatabase {
      instance?.let { return it }
      instance = Room
        .databaseBuilder(
          ctx, 
          RoomDatabase::class.java, 
          RoomDatabaseConfig.DB_NAME)
        .build()
      return instance!!
    }
	
    fun getTestInstance(ctx: Context): MyDatabase {
      testInstance?.let { return it }
      testInstance = Room
        .databaseBuilder(
          ctx, 
          RoomDatabase::class.java, 
          RoomDatabaseConfig.DB_TEST_NAME)
        .build()
      return testInstance!!
    }
	
  }
  
  internal abstract fun userDao(): UserDao

}
```

## Protocols for database operations

When performing database operations, you will usually want to perform similar set of actions for each of the operations. One such example: 

1. open the database
2. perform the operation
3. log the result
4. throw any possible exceptions
5. close the database

I created a class for this, that will receive a coroutine dispatcher on which to execute database operations, and that defines possible read and write operations. I named this class `RoomDatabaseOperationProtocol`, but if you're smart, you can probably come up with something a bit better :)

```kotlin
internal class RoomDatabaseOperationProtocol(
  private val dispacher: CoroutineDispatcher
) {

  companion object {
    private const val LOG_TAG = "RoomDatabase.Operation"
  }

  suspend fun executeWriteOperation(
    operationName: String,
    operation: suspend CoroutineScope.() -> Unit
  ) {
    withContext(dispatcher) {
      try {
        Log.d(LOG_TAG, "[CALL] $operationName")
        operation.invoke(this)
        Log.d(LOG_TAG, "[SUCCESS] $operationName")
      } catch (ex: Exception) {
        Log.e(LOG_TAG, "[FAILURE] $operationName", ex)
        throw ex
      }
    }
  }
  
  suspend fun <T> executeReadOperation(
    operationName: String,
    operation: suspend CoroutineScope(). -> T
  ): T {
    return withContext(dispatcher) {
      try {
        Log.d(LOG_TAG, "[CALL] $operationName")
        val result = operation.invoke(this)
        Log.d(LOG_TAG, "[SUCCESS] $operationName")
        result
      } catch (ex: Exception) {
        Log.e(LOG_TAG, "[FAILURE] $operationName", ex)
        throw ex
      }
    }
  }
  
  fun <T> executeObserveOperation(
    operationName: String,
    operation: () -> Flow<T>
  ): Flow<T> {
    return try {
      Log.d(LOG_TAG, "[CALL] $operationName")
      operation.invoke()
        .flowOn(dispatcher)
        .onEach { Log.d(LOG_TAG, "[SUCCESS] $operationName") }
    } catch (ex: Exception) {
      Log.e(LOG_TAG, "[FAILURE] $operationName", ex)
    }
  }
  
}

```

Let's see what we did here:
- We've created a protocol for each of the possible operations: read, write, observe
- Each of the protocols will make sure the operations is executed on the proper dispatcher and it will log the operation being called
- Note: in the full implementation I also log the data being read/written, but removed it here for simplicity

## Public APIs

We've marked all the components that deal with data operations (DAOs, protocols) as `internal`. This means that the access to them is restricted to this module. I use separate module for data, but you can make it package private if you don't have modules. The idea is to stop other components from accessing DAOs directly, but use the public APIs, which will make sure all the operations are performed on the correct thread following the defined protocols.

Since I use the Android's [MVVM architecture][mvvm], which uses `Repository` suffix for the classes that serve as the data source, I named the class with public APIs `RoomDatabaseRepository`, but you can of course use whatever you like.

In this class, we will define all the public APIs for database operations, and then use DAOs to execute them following the specified database protocols:

```kotlin
class RoomDatabaseRepository(
  private val db: MyRoomDatabase,
  private val dispatcher: CoroutineDispatcher
) {

  private val protocol = RoomDatabaseOperationProtocol(dispatcher)

  suspend fun getUser(id: Int): User? {
    return protocol.executeReadOperation("getUser") {
      db.userDao().get(id)
    }
  }
  
  suspend fun getAllUsers(): List<User> {
    return protocol.executeReadOperation("getAllUsers") {
      db.userDao().getAll()
    }
  }

  suspend fun insertOrUpdateUser(user: User) {
    protocol.executeWriteOperation("insertOrUpdateUser") {
      db.userDao().insertOrUpdate(user)
    }
  }
  
  suspend fun insertOrUpdateUsers(users: List<User>) {
    protocol.executeWriteOperation("insertOrUpdateUsers") {
      db.userDao().insertOrUpdate(users)
    }
  }
  
  suspend fun deleteUser(user: User) {
    protocol.executeWriteOperation("deleteUser") {
      db.userDao().delete(user)
    }
  }
  
  suspend fun deleteAllUsers() {
    protocol.executeWriteOperation("deleteAllUsers") {
      db.userDao().deleteAll()
    }
  }
  
  fun observeUsers(): Flow<List<User>> {
    return protocol.executeObserveOperation("observeUsers") {
      db.userDao().observe()
    }
  }

}
```

You can see that we've used one of 3 defined protocols to cover each of the operations in our DAO object. Operations on other DAOs will be added to this class as well. This is the class we'll be passing to the components in our app that need to perform database operations.

## Conclusion

Separation allows your components to safely invoke database operations. This way the components that use the database repository don't care if something beneath changes. E.g. if you add more logging, you automatically get more logging in every call. You change the dispatcher, all the calls are invoked on the new dispatcher. And so on.

If you're using MVVM for example, you can simply get all users by calling the repository function inside the `viewModelScope`, e.g:

```kotlin
class UsersViewModel(private val dbRepo: RoomDatabaseRepository)
: ViewModel() {
  
  init {
    viewModelScope.launch {
      val users = dbRepo.getAllUsers()
      // Do something with users.
    }
  }
  
}
```

If you notice any mistakes, fallacies, or simply have a better approach, feel free to let me know at **ivanskoric7@gmail.com**. Cheers!

[room]: https://developer.android.com/training/data-storage/room
[mvvm]: https://developer.android.com/jetpack/guide
