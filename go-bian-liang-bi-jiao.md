# Go 变量比较

## 基本类型

### 定义

包括整型、浮点类型、复数类型、字符串类型、布尔类型

### 比较方式

由于golang不会做隐式转换，"a==b" 会先判断两种类型是否严格一致，然后比较值是否一致

```
var a uint8 = 1
var b uint32 = 1
// a == b is invalid operation, because mismatched types uint8 and uint32
// (uint32)a == b is true
```

在类型判断中，再定义类型与原类型不相等，别名类型与原类型相等

```
type myType int
var a int = 1
var b myType = 1
// a == b is invalid operation

type myType = int
var a int = 1
var b myType = 1
// a == b is true
```

## 复合类型

### 定义

包括数组和结构体

### 比较方式

#### 数组

数组的长度也是类型的一部分，只有数组类型一致且数组元素是可比较类型，数组才可以比较。比较时，会逐个元素进行比较，即深度比较

#### struct

只有struct类型一致才可以进行比较。比较时，会逐个成员进行比较，即深度比较

## 引用类型

### 定义

包括slice、map、channel和基础类型对象的引用（类似\&val，即变量指针）

### 比较方式

#### channel和基础类型对象的引用

只比较引用类型本身的值，即指针值，也就是浅层比较

```
type myStruct struct{
    a int
    b string
}

var a = &myStruct{a=1, b="test"}
var b = &myStruct{a=1, b="test"}
// a == b is false

var c = a
// a == c is true
```

#### slice

slice不可比较，只能和nil比较

slice的零值是nil，而空slice为长度为0的slice，不是nil

```
var s []int
// s == nil is true
var s = []int{}
// s == nil is false, because s is empty slice
var s = make([]int, 0, 2)
// s == nil is false, beacuse s is empty slice
```

#### map

map不可比较，只能和nil比较

map的零值也是nil，注意和空map的区别

### 为什么slice和map不可比较

slice和map如果按照引用类型的比较规则，应该只需要比较它们本身的值，即浅层比较。但种做法与正常使用不符，我们大部分场景都需要比较它们的底层值是否一致

但目前可以底层比较slice和map的做法都不高效，golang的开发者就干脆不提供这个特性

### 如何底层比较slice和map

特别的，\[]byte可以使用byte.Equal来进行底层比较

#### reflect.DeepEqual

该函数可以用来比较两个任意类型的变量，包括map、slice、struct。函数内部会做类型判断和遍历登记，防止比较对象自我引用的问题。但因为用了反射，性能比自己循环遍历要差

```
m1 := map[string]int{"foo": 1, "bar": 2}
m2 := map[string]int{"foo": 1, "bar": 2}
// reflect.DeepEqual(m1, m2) is true

m2 = map[string]int{"foo": 1, "bar": 3}
// reflect.DeepEqual(m1, m2) is false
```

但注意，该函数没有实现slice和map与nil比较的特性，如下

```
var m5 map[float64]string
// reflect.DeepEqual(m5, nil) is false
// m5 == nil is true
```

## 接口类型

接口类型包括动态类型和动态值，只有两者都相等，比较才相等

```
type Person interface {
    getName() string
}

type Student struct {
    Name string
}

type Teacher struct {
    Name string
}

func (s Student) getName() string {
    return s.Name
}

func (t Teacher) getName() string {
    return t.Name
}

var p1 Person = Student{"frank"}
var p2 Person = Teacher{"frank"}
// p1 p2 的接口类型是Person，它们动态类型是Student和Teacher，虽然它们动态值是相等的，但p1不等于p2
```

## 函数类型

函数类型不可比较
