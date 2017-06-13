---
layout: post
title: "Swift语法参考"
date: 2016-02-21 08:00
keywords: ha
description: swift基础语法参考
categories: [apple]
tags: [swift]
group: archive
icon: globe
---
## 类型

#### 基础类型

* Bool
* Int Int8 Int16 Int32 Int64
* UInt UInt8 UInt16 UInt32 UInt64
* Float Double
* Character  eg:"a"
* String

#### 可选类型

可以为nil的基础类型：```基础类型?```, 默认值为nil   
eg: Int? String?

<!-- more -->

#### 枚举类型

```swift
//有类型，Woman隐含为2
enum EgEnum : Int {
	case Man = 1
	case Woman
}
//也可以在一行
enum EgEnum : Int { case Man = 1, Woman }
//可以根据原始值查找特定的元素
var select = EgEnum(rawValue: 2) //Woman
//可以查看某元素的原始值
var index = EgEnum.Man.rawValue //1

//无类型，不等于隐含0，1，2和3
enum EgCompass {
	case North
	case South
	case East
	case West
}
//也可以在一行
enum Compass { case North, South, East, West }

//元素的数据类型可以各不相同
enum Shape {
	case Circle(Int)
	case Square(Int, Int)
}
```

#### 类型别名

```swift
typealias BigInt = Int32
var egBigInt : BigInt = 0
```

#### 获取数值上下限

```swift
Int.min
Int.max
```

#### 类型转换

* 数值转String

```swift
var egString = String(egInt)
```

* String转数值

```swift
var egInt = egString.toInt()
```

#### 字符串拼接

```swift
var egString1 = "hello"
var egString2 = "world"
```

* 一般拼接

```swift
print("hi," + egString1 + " " + egString2)
```

* 变量标记

```swift
print("hi,\(egString1) \(egString2)")
```

## 声明

Character需要显式声明，自动识别为String.

#### 变量声明

```swift
var eg = 10.0 //自动识别类型
var eg : Double = 10 //显式声明类型
```

#### 常量声明

```swift
let eg = 10.0 //自动识别类型
let eg : Double = 10 //显式声明类型
```

## 运算

#### 四则运算

* 运算符：\+ - * / = += -= *= /=
* 正负数：+5  -5
* 进制：二进制 0b；八进制 0o；十六进制 0x
* 科学计数：4.4321e-10 e表示以10为底的指数
* 量级：5_000_000

#### 比较运算

* 运算符：> >= < <= == != === !== (最后两个比较对象是否相同或不同)
* 返回：true false

## 元组 Tuple

有序

```swift
var egTuple = (2012, "miaoze", "cumtb")
print(egTuple.0) //输出：2012
print(egTuple.1) //输出：miaoze
print(egTuple.2) //输出：cumtb
print(egTuple.3) //输出：error: '(Int,String,String)' does not have a member named '3'
```

## 数组

有序、元素类型相同，元素可以是基本类型、可选类型、数组、字典、nil

#### 声明和初始化

```swift
//方法1
var egArray : Array<String> = []
egArray = ["beijing", "shanghai", "guangzhou"]
//方法2
var egArray = [String]()
egArray = ["beijing", "shanghai", "guangzhou"]
//方法3
var egArray : [String] = ["beijing", "shanghai", "guangzhou"]
//方法4
var egArray = ["beijing", "shanghai", "guangzhou"]
```

#### 输出元素

```swift
print(egArray[0]) //输出：beijing
print(egArray[3]) //输出：fatal error: Array index out of range
```

#### 添加元素

append()

```swift
egArray.append("shenzhen")
//egArray = ["beijing", "shanghai", "guangzhou", "shenzhen"]
print(egArray[3]) //输出：shenzhen
egArray.insert("hangzhou")
//egArray = ["beijing", "shanghai", "guangzhou", "shenzhen", "hangzhou"]
print(egArray[4]) //输出：hangzhou
```

#### 合并数组

+=

```swift
egArray += ["taiwan","aomen"]
//egArray = ["beijing", "shanghai", "guangzhou", "shenzhen", "hangzhou", "taiwan", "aomen"]
print(egArray[5]) //输出：taiwan
print(egArray[6]) //输出：aomen
```

#### 替换元素

=

```swift
egArray[4] = "xianggang"
//egArray = ["beijing", "shanghai", "guangzhou", "shenzhen", "xianggang", "taiwan", "aomen"]
print(egArray[4]) //输出：xianggang
```

#### 删除元素

removeLast() / removeAtIndex()

```swift
egArray.removeLast()
//egArray = ["beijing", "shanghai", "guangzhou", "shenzhen", "taiwan"]
print(egArray[6) //输出：fatal error: Array index out of range
egArray.removeAtIndex(0)
//egArray = ["shanghai", "guangzhou", "shenzhen", "taiwan"]
print(egArray[0]) //输出：shanghai
```

## 字典

键值对，键类型相同、值类型相同，无序，键值都不能为nil   
字典的键只能是基本类型，值可以是基本类型、可选类型，也可以是字典、数组，但建和值都不能为nil

#### 声明和初始化

```swift
//方法1
var egDict = Dictionary<String, String>()
egDict = ["item1":"value1", "item2":"value2"]
//方法2
var egDict : [String:String] = ["item1":"value1", "item2":"value2"]
//方法3
var egDict = ["item1":"value1", "item2":"value2"]
```

#### 输出元素

```swift
var item = egDict["item2"]
print(item) //输出：value2
var item = egDict["item3"]
print(item) //输出：nil
```

#### 添加元素

```swift
egDict["item3"] = "value4"
print(item) //输出：value4
egDict["item4"] = 100 //输出：error: cannot assign value of type 'Int' to type 'String?'
```

#### 更新元素

```swift
egDict["item3"] = "value3"
print(item) //输出：value3
```

#### 删除元素

```swift
egDict["item3"] = nil
print(egDict["item3"]) //输出：nil
```

## 循环

#### for-in

```swift
//[1,10]
for loopCounter in 1...10 {}

//[1,10)
for loopCounter in 1..<10 {}

//数组
for item in egArray {}

//字典
for item in egDict {}
```

#### for(;;)

```swift
//步进2，[0,10)
for (var idx = 0; idx < 10; idx = idx + 2)  {}
```

* 小括号可以省略
* 新版本中该语法已经废弃

#### while

```swift
while (condition) {}
```

* 小括号可以省略

#### repeat-while

```swift
repeat {} while (condition)
```

* 小括号可以省略
* 稍老的版本为do-while, 新版本为repeat-while

#### 循环控制

* break 提前退出循环
* continue 本次跳过循环内后面的语句，直接进行下一次循环

## 判断

#### if-else if-else

```swift
if (predicate) {}
else if (predicate) {}
else {}
```

* 小括号可以省略

#### switch-case

```swift
var egStringArray = ["beijing", "shanxi", "shannxi", "taiyuan"]
for item in egStringArray {
	switch item {
		case "beijing": print("bj")
		case "shanxi", "shannxi": print("sx")
		default: print("others")
	}
}

var egIntArray = [1,2,3,4,5,6]
for item in egIntArray {
	switch item {
		case 1: print("=1")
		case 2...6: print(">1")
		default: print("overflow")
	}
}

//可以和枚举类型结合使用
switch person {
	case .Man: print("person is man")
	case .Woman: print("person is woman")
}
```

## 函数

```swift
func funcName(paraName:type, ...) -> (returnType1,returnType2) {
	rst1 : returnType1
	rst2 : returnType2
	//...
	return (rst1,rst2)
}
```

#### 参数列表
1. 参数类型

	1. 一般参数 `paraName1:type,paraName2:type,...`
		1. 接受空参数，不写参数，或者写 _: Void

```swift
func egFunc(_: Void) -> (Void) {}
//调用
egFunc()
```

		2. 接受基础类型、可选类型、数组、元组

调用时，如果有多个参数，第一个参数*不*指明参数名，其余参数*必须*指明。

```swift
func egFunc(egParam:Int) -> Int {
	return egParam
}
func egFunc(egParam1:Int, egParam2:Int) -> Int {
	return egParam1+egParam2
}
//调用
print(egFunc(1)) //输出：1
print(egFunc(1,egParam2:2)) //输出：3
```

		3. 接受函数，比如

```swift
func egFunc(egParamI:Int, egParamF:(Int)->Int) -> (Int) {
	return egParamF(egParamI)
}
//调用
let egFuncUseAsParam = egFunc1
print(egFunc(1, egParamF: egFuncUseAsParam)) //输出：1
```

	2. 可变参数 `paraName1:type,paraNameChanged:type...`
		1. 可变参数必须在最后
		2. 可变参数中的所有参数类型相同
		3. 可变参数使用`for paraName in paraNameChanged`提取

	3. 变量参数
按照前面的方式传入的参数是常量，无法在方法内修改其值。可用使用`var`修饰参数，将其变为变量参数，变量参数的值可以在方法内修改，但是注意，*在方法内修改变量参数，不会影响方法外该变量的值。*这相当于C的值传递。

```swift
func egFunc(egParam:Int) -> (Int) {
	//egParam = 2   
	//error：cannot assign to value: 'egParam' is a 'let' constant
	return egParamF(egParamI)
}

func egFunc(var egParam:Int) -> (Int) {
	egParam = 2
	return egParam
}
var egInt = 5
print(egFunc(egInt)) //输出：2
print(egInt) //输出：5
```

	4. inout参数
使用`inout`修饰的参数，在调用时，传入的参数需要使用`&`修饰，需要传入在方法内可以修改，并且*会影响方法外该变量的值*。这相当于C的指针传递。

```swift
func egFunc(inout egParam:Int) -> (Int) {
	egParam = 2
	return egParam
}
var egInt = 5
print(egFunc(&egInt)) //输出：2
print(egInt) //输出：2
```

2. 参数默认值

调用时，第一个参数*不*指明参数名和冒号，其他参数*必须*指明。

```swift
func egFunc(egParamI:Int=1, egParamS:String="unknown") {
    print(String(egParamI) + "-" + egParamS)
}
//调用
egFunc() //输出：1-unknown
egFunc(2) //输出：2-unkown
//egFunc(egParamI:2) //错误，指定非默认参数值时不需要指明参数名
//egFunc(3, "good") //错误，指定默认参数值时必须指明参数名
egFunc(egParamS:"good") //输出：1-good
egFunc(3, egParamS:"good") //输出：3-good
```

3. 参数外部名

```swift
//普通的参数
func egFunc(egParamSrc:String, egParamDes:String) {}
//调用
egFunc("from", "to")
//使用外部名的参数
func egFunc(Src egParamSrc:String, Des egParamDes:String) {}
//调用
egFunc(Src:"from", Des:"to")
```

#### 函数返回

1. 返回类型
	1. 返回值可以为基础类型、可选类型、数组、元组
	2. 返回值可以为函数

```swift
//返回无返回值的函数
func egFunc1(egParam:Int) {
    print("egFunc1-get" + String(egParam))
}
func egFunc2(egParam:Int) {
    print("egFunc2-get" + String(egParam))
}
func egFunc(egParam:Int) -> (Int) -> (){
    if egParam == 1 {
        return egFunc1
    } else {
        return egFunc2
    }
}
egFunc(1)(5) //输出：egFunc1-get5

//返回无参数的函数
func egFunc1() -> String {
	return "egFunc1"
}
func egFunc2() -> String {
	return "egFunc2"
}
func egFunc(egParam:Int) -> () -> (String){
	if egParam == 1 {
		return egFunc1
	} else {
		return egFunc2
	}
}
print(egFunc(1)()) //输出：egFunc1

//返回带参有返回值的函数
func egFunc1(egParam:Int) -> String {
	return "egFunc1-get" + String(egParam)
}
func egFunc2(egParam:Int) -> String {
	return "egFunc2-get" + String(egParam)
}
func egFunc(egParam:Int) -> (Int) -> String {
	if egParam == 1 {
		return egFunc1
	} else {
		return egFunc2
	}
}
print(egFunc(1)(5)) //输出：egFunc1-get5
```

2. 支持返回多个参数

#### 函数嵌套

在函数中嵌套函数可以避免暴露不必要的功能。

```swift
func egFunc(egParam:Int) -> String {
    func egFuncIn(egParam:Int) -> String {
        return "egFuncIn-get" + String(egParam)
    }
    return egFuncIn(egParam)
}
print(egFunc(1)) //输出：egFuncIn-get1
```

## 类

#### 基本定义/创建对象/使用

```swift
class EgClass {
	var egProperty1 : Bool = false
	let egProperty2 : Int = 1
	
	func fetchEgProperty2() -> Int {
		egProperty1 = true
		return egProperty2
	}
}

let egClassObj = EgClass()
print(egClassObj.egProperty1) //输出 true
print(egClassObj.fetchEgProperty2()) //输出 1
egClassObj.egProperty1 = false
print(egClassObj.egProperty1) //输出 false
```

#### 对象的初始化（构造）

```swift
class EgClass {
    var egProperty1 : Bool = false
    let egProperty2 : Int = 1
    
    init(egProperty1: Bool = true) {
        self.egProperty1 = egProperty1
    }
    
    func fetchEgProperty2() -> Int {
        egProperty1 = true
        return egProperty2
    }
}

let egClassObj1 = EgClass()
print(egClassObj1.egProperty1) //输出 true
print(egClassObj1.fetchEgProperty2()) //输出 1
print(egClassObj1.egProperty1) //输出 true

let egClassObj2 = EgClass(egProperty1: false)
print(egClassObj2.egProperty1) //输出 false
print(egClassObj2.fetchEgProperty2()) //输出 1
print(egClassObj2.egProperty1) //输出 true
```


