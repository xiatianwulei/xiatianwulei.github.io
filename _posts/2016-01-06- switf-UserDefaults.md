---
layout:     post
title:      UserDefaults 
subtitle:   
date:       2018-01-06
author:     夏天无泪
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Switf
    - 学习整理
---

#  switf 使用UserDefaults来进行本地数据存储 学习整理

#### 注意点
* 一般来说本地存储数据我们还可以是用 SQlite 数据库，或者使用自己建立的 plist 文件什么的，但这还得自己显示创建文件，读取文件，很麻烦，而是用 UserDefaults 则不用管这些东西，就像读字符串一样，直接读取就可以了。
* UserDefaults 支持的数据格式也很多，有：Int，Float，Double，BOOL，Array，Dictionary，甚至 Any 类型。
* UserDefaults 里面的数据最终是保存到 .plist 文件中，理论上 UserDefaults 存放好几个 G 的数据是可以实现的（取决于设备剩余的空间）。
* 但 UserDefaults 设计就不是用来存储大数据的。因为当应用启动时会自动加载 Userdefault 里所有的数据，如果太大的话就会造成启动缓慢，影响性能
* 所以大文件还是建议使用 Core Data、 CloudKit、SQLite 等结构化存储方式。

#### 使用
 1、 对原生数据类型的储存和读取
 
```
let userDefault = UserDefaults.standard
 
//Any
userDefault.set("gge.com", forKey: "Object")
let objectValue:Any? = userDefault.object(forKey: "Object")
 
//Int类型
userDefault.set(12345, forKey: "Int")
let intValue = userDefault.integer(forKey: "Int")
 
//Float类型
userDefault.set(3.2, forKey: "Float")
let floatValue = userDefault.float(forKey: "Float")
 
//Double类型
userDefault.set(5.2240, forKey: "Double")
let doubleValue = userDefault.double(forKey: "Double")
 
//Bool类型
userDefault.set(true, forKey: "Bool")
let boolValue = userDefault.bool(forKey: "Bool")
 
//URL类型
userDefault.set(URL(string:"http://hangge.com")!, forKey: "URL")
let urlValue = userDefault.url(forKey: "URL")
 
//String类型
userDefault.set("hangge.com", forKey: "String")
let stringValue = userDefault.string(forKey: "String")
 
//NSNumber类型
var number = NSNumber(value:22)
userDefault.set(number, forKey: "NSNumber")
number = userDefault.object(forKey: "NSNumber") as! NSNumber
 
//Array类型
var array:Array = ["123","456"]
userDefault.set(array, forKey: "Array")
array = userDefault.array(forKey: "Array") as! [String]
 
//Dictionary类型
var dictionary = ["1":"hangge.com"]
userDefault.set(dictionary, forKey: "Dictionary")
dictionary = userDefault.dictionary(forKey: "Dictionary") as! [String : String]
```

2、系统对象的存储与读取
* 系统对象实现存储，需要通过 archivedData 方法转换成 Data 为载体，才可以存储。下面以 UILabel 对象为例：

```
let userDefault = UserDefaults.standard
 
//UILabel对象存储
//将对象转换成Data流
let label = UILabel()
label.text = "欢迎访问hangge.com"
let labelData = NSKeyedArchiver.archivedData(withRootObject: label)
//存储Data对象
userDefault.set(labelData, forKey: "labelData")
 
//UILabel对象读取
//获取Data
let objData = userDefault.data(forKey: "labelData")
//还原对象
let myLabel = NSKeyedUnarchiver.unarchiveObject(with: objData!) as? UILabel
print(myLabel)
```
补充： 
对于 UIImage 对象的存储比较特殊。注意下方高亮部分，如果我们过直接把 image1 存储起来，再取出转换回 UIImage 就变成了 nil。必须先转成 image2 再存储。

```
let userDefault = UserDefaults.standard
 
//UIImage对象存储
//将对象转换成Data流
let image1 = UIImage(named: "apple.png")!
let image2 = UIImage(cgImage: image1.cgImage!, scale: image1.scale,
                     orientation: image1.imageOrientation)
let imageData = NSKeyedArchiver.archivedData(withRootObject: image2)
//存储Data对象
userDefault.set(imageData, forKey: "imageData")
 
//UIImage对象读取
//获取Data
let objData = userDefault.data(forKey: "imageData")
//还原对象
let myImage = NSKeyedUnarchiver.unarchiveObject(with: objData!) as? UIImage
print(myImage)
```
3、自定义对象的存储和读取

*如果想要存储自己定义的类，首先需要对该类实现 NSCoding 协议来进行归档和反归档（序列化和反序列化）。即该类内添加 func encode(with coder: NSCoder) 方法和 init(coder decoder: NSCoder) 方法，将属性进行转换。*

```
let userDefault = UserDefaults.standard
 
//自定义对象存储
let model = UserInfo(name: "航歌", phone: "3525")
//实例对象转换成Data
let modelData = NSKeyedArchiver.archivedData(withRootObject: model)
//存储Data对象
userDefault.set(modelData, forKey: "myModel")
 
//自定义对象读取
let myModelData = userDefault.data(forKey: "myModel")
let myModel = NSKeyedUnarchiver.unarchiveObject(with: myModelData!) as! UserInfo
print(myModel)
 
//----- 自定义对象类 -----
class UserInfo: NSObject, NSCoding {
    var name:String
    var phone:String
     
    //构造方法
    required init(name:String="", phone:String="") {
        self.name = name
        self.phone = phone
    }
     
    //从object解析回来
    required init(coder decoder: NSCoder) {
        self.name = decoder.decodeObject(forKey: "Name") as? String ?? ""
        self.phone = decoder.decodeObject(forKey: "Phone") as? String ?? ""
    }
     
    //编码成object
    func encode(with coder: NSCoder) {
        coder.encode(name, forKey:"Name")
        coder.encode(phone, forKey:"Phone")
    }
}
```

4、 删除对象 和 数据同步
```
UserDefaults.standard.removeObject(forKey: "ge")
 UserDefaults.standard.synchronize() /// 同步
```