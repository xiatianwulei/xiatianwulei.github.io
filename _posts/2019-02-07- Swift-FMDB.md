---
layout:     post
title:      FMDB 安装和使用 
subtitle:   
date:       2017-01-06
author:     夏天无泪
header-img: img/post-bg-ios9-web.jpg?raw=true
catalog: true
tags:
    - Swift
---


# swift FMDB安装和使用 - 学习整理

### FMDB

**因为最近在开发一个公司自用品控项目，数据要做本地数据，所以搭建SQLite数据库框架。**

#### 1.FMDB简介：

* FMDB 是 iOS 平台的 SQLite 数据库框架。
* FMDB 以 OC 的方式封装了 SQLite 的 C 语言的 API。
* FMDB 对比苹果自带的 Core Data 框架，更加轻量级和灵活。使用起来更加面向对象，省去了很多麻烦、冗余的 C 语言代码
* FMDB 还提供了多线程安全的数据库操作方法，能有效地防止数据混乱

#### 2.FMDB主要工作类：

1）**FMDatabase**：一个 FMDatabase 表示一个 sqlite 数据库，所有对数据库的操作都是通过这个类（线程不安全）

* executeStatements：执行多条 sql。
* executeQuery：执行查询语句。
* executeUpdate：执行除查询以外的语句，如create、drop、insert、delete、update。
   

2）**FMDatabaseQueue**：内部封装 FMDatabase 和串行 queue，用于多线程操作数据库，并且提供事务（线程安全）。  

* inDatabase: 参数是一个闭包,在闭包里面可以获得
* FMDatabase 对象。
* inTransaction: 使用事务。
 
3） **FMResultSet**：查询的结果集。可以通过字段名称获取字段值。

### FMDB 使用
#### 1.FMDB 的安装配置
1）GiThub 地址：[https://github.com/ccgus/fmdb]()  
2) 安装cocopods 这就不做解释了 cd：项目目录 pod init 创建podfile文件，然后执行pod instal 文件

 ![-w642](https://github.com/xiatianwulei/xiatianwulei.github.io/blob/master/img/media/15759433308350/15759479989438.jpg?raw=true)
 
**当然你也可以把项目下载下来直接导入。
注意：fmdb 文件夹中的 info.plist 要删除掉** 

1） FMDB已经导入进来了
![-w272](https://github.com/xiatianwulei/xiatianwulei.github.io/blob/master/img/media/15759433308350/15759483148299.jpg?raw=true)

2） 因为我搭建的项目是swift ，所以要做个桥接文件。
**补充下 创建桥接文件随便创建个oc 类然后选择'Create Bridging Header'**  

![-w771](https://github.com/xiatianwulei/xiatianwulei.github.io/blob/master/img/media/15759433308350/15759589317188.jpg?raw=true)

 **创建完桥接文件并把FMDB头文件类引进来**  
 
![-w1058](https://github.com/xiatianwulei/xiatianwulei.github.io/blob/master/img/media/15759433308350/15760290225713.jpg?raw=true)

3）在 Build Phases -> Link Binary With Libraries 中点击加号，添加 libsqlite3.0.tbd 到项目中来  

![-w821](https://github.com/xiatianwulei/xiatianwulei.github.io/blob/master/img/media/15759433308350/15759593278464.jpg?raw=true)


#### 2.FMDB 封装工具类
为了方便使用，我需要自己封装一个工具类

```
// 数据库管理类
class SQLiteManager: NSObject {
     
    // 创建单例
    private static let manger: SQLiteManager = SQLiteManager()
    class func shareManger() -> SQLiteManager {
        return manger
    }
     
    // 数据库名称 .db 是数据库文件格式类型
    private let dbName = "test.db"
     
    // 数据库地址
    lazy var dbURL: URL = {
        // 根据传入的数据库名称拼接数据库的路径
        let fileURL = try! FileManager.default
            .url(for: .applicationSupportDirectory, in: .userDomainMask,
                 appropriateFor: nil, create: true)
            .appendingPathComponent(dbName)
        print("数据库地址：", fileURL)
        return fileURL
    }()
     
    // FMDatabase对象（用于对数据库进行操作）
    lazy var db: FMDatabase = {
        let database = FMDatabase(url: dbURL)
        return database
    }()
     
    // FMDatabaseQueue对象（用于多线程事务处理）
    lazy var dbQueue: FMDatabaseQueue? = {
        // 根据路径返回数据库
        let databaseQueue = FMDatabaseQueue(url: dbURL)
        return databaseQueue
    }()
}
  private init() {} // 私有化init方法
```

```
class ViewController: UIViewController {

    override func viewDidLoad() {
        super.viewDidLoad()
        // Do any additional setup after loading the view.
        
        let db = SQLiteManager.shareManger().db
        if db.open() {
            print("💣💣💣💣💣💣💣💣💣💣💣数据库打开成功!")
        } else {
            print("🛫🛫🛫🛫🛫🛫🛫🛫🛫🛫🛫数据库打开失败: \(db.lastErrorMessage())")
        }
        db.close()
    }
}

```

**数据库成功打开了**

![-w473](https://github.com/xiatianwulei/xiatianwulei.github.io/blob/master/img/media/15759433308350/15759600817293.jpg?raw=true)
**本地文件**

![-w777](https://github.com/xiatianwulei/xiatianwulei.github.io/blob/master/img/media/15759433308350/15759614116344.jpg?raw=true)

**介绍个简单的数据管理工具 **

下载地址：https://xclient.info/s/datum.html#versions

#### 3.FMDB 基础使用
**运行下面代码我创建一个user 类 里面包含性别 ID 年龄 名字 性别**

SQLite3 ：关系型数据库，不能直接存储对象，要将对象拆开存不能直接存储对象，要储，所以要把对象给拆出来

```
         // 编写SQL语句（id: 主键  name和age sex是字段名）
         let sql = "CREATE TABLE IF NOT EXISTS User(userId INTEGER PRIMARY KEY AUTOINCREMENT,name TEXT,sex TEXT,age INTEGER); "
          
         // 执行SQL语句（注意点: 在FMDB中除了查询意外, 都称之为更新）
         let db = SQLiteManager.shareManger().db
         if db.open() {
             if db.executeUpdate(sql, withArgumentsIn: []){
                 print("创建表成功")
             }else{
                 print("创建表失败")
             }
         }
         db.close()
```

这个数据库创建成功了
![-w951](https://github.com/xiatianwulei/xiatianwulei.github.io/blob/master/img/media/15759433308350/15759721239000.jpg?raw=true)
     
#### 4.数据库的操作使用
**数据库创建完后，对数据库进行 增 删 改 查操作**
1) 增加一条数据

```
        /**
         insert into 表名 (字段1, 字段2, …) values (字段1的值, 字段2的值, …) ;
         let sql = "INSERT INTO User (name, age,sex) VALUES ('name', 100,'男人');"
         注意： 数据库中的字符串内容应该用一对单引号 ' 包含
         **/
        // 编写SQL语句 我采用的是预编译方式 效果是一样的
        let sql = "INSERT INTO User (name, age, sex) VALUES (?,?,?);"
         
        // 执行SQL语句
        let db = SQLiteManager.shareManger().db
        let age:Int = Int(arc4random() % 100)
        if db.open() {
            if db.executeUpdate(sql, withArgumentsIn: ["xiatianwulei", age,"男人"]){
                print("插入成功")
            }else{
                print("插入失败")
            }
        }
        db.close()

```

* 给表添加一个新字段 let sql = "ALTER TABLE User ADD phone TEXT;"

![-w900](https://github.com/xiatianwulei/xiatianwulei.github.io/blob/master/img/media/15759433308350/15760340814326.jpg?raw=true)

2) 删

```
    // 删除特定1号userID 信息
    let sql = "DELETE FROM User WHERE userID = ?;"
            // 执行SQL语句
    let db = SQLiteManager.shareManger().db
        if db.open() {
                if db.executeUpdate(sql, withArgumentsIn: [1]){
                print("删除成功")
        }else{
                print("删除失败")
            }
        }
        db.close()
```

* 删除整个表格数据 let sql = "delete from User"
* 删除整个表 let sql = "drop table User"


3）改

```
/*
 update 表名 set 字段1 = 字段1的值, 字段2 = 字段2的值, … ;
 示例
 update t_student set name = ‘jack’, age = 20 ;
 */
// 编写SQL语句
let sql = "UPDATE User set name = ? WHERE userId = ?;"

// 执行SQL语句
let db = SQLiteManager.shareManger().db
if db.open() {
    if db.executeUpdate(sql, withArgumentsIn: ["wudile", 40]){
        print("更新成功")
    }else{
        print("更新失败")
    }
}
db.close()  
```
 
4) 查

```
// 编写SQL语句
let sql = "SELECT * FROM User WHERE userId < ?"
// 执行SQL语句
let db = SQLiteManager.shareManger().db
if db.open() {
    if let res = db.executeQuery(sql, withArgumentsIn: [46]){
        // 遍历输出结果
        while res.next() {
            let userID = res.int(forColumn: "userID")
            let name = res.string(forColumn: "name")!
            let age = res.int(forColumn: "age")
            let sex = res.string(forColumn: "age")
            print(userID, name, age, sex)
        }
    }else{
        print("查询失败")
    }
}
db.close()
```

5)  **FMDatabaseQueue** 使用  
（1）有时我们要对大批量的数据进行操作。比如同时插入大量的数据，如果每插入一条就提交一次的话就会比较耗时。  
（2）FMDB 提供的 FMDatabaseQueue 对象可以使用事务进行批量操作，大大缩短了插入时间。同时它还支持回滚，只要中间有一条语句执行错误，就会自动全部恢复成最初的状态。  

```
if let queue = SQLiteManager.shareManger().dbQueue {
    queue.inTransaction { db, rollback in
        do {
            for i in 0..<10 {
                try db.executeUpdate("INSERT INTO User (name, age) VALUES (?,?);",
                                     values: ["xiatianwulei", i])
            }
            print("插入成功!")
        } catch {
            print("插入失败，进行回滚!")
            rollback.pointee = true
        }
    }
}
```

