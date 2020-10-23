map是go语言中的内置的字典类型，他存储的是一个键值对 `key:value` 类型的数据。map也是一种容器，和数组切片不一样的是，他不是通过下标来访问数据，而是通过key来访问数据

### map 特点

- map是无序的、长度不固定、不能通过下标获取，只能通过key来访问。
- map的长度不固定 ，也是一种引用类型。可以通过内置函数 len(map)来获取map长度。
- 创建map的时候也是通过make函数创建。
- map的key不能重复，如果重复新增加的会覆盖原来的key的值。

```go
声明  变量名称 map[key的数据类型]value的数据类型
//2，使用make声明
m2:=make(map[key_data_type]value_data_type)
//3,直接声明并初始化赋值map方法
m3:=map[string]int{"语文":89,"数学":23,"英语":90}
```

### map的使用

map 是引用类型的，如果声明没有初始化值，默认是nil。**空的切片是可以直接使用的，因为他有对应的底层数组,空的map不能直接使用。需要先make之后才能使用**。

```go
var m1 map[int]string         //只是声明 nil
var m2 = make(map[int]string) //创建
m3 := map[string]int{"语文": 89, "数学": 23, "英语": 90}

fmt.Println(m1 == nil) //true
fmt.Println(m2 == nil) //false
fmt.Println(m3 == nil) //false

//map 为nil的时候不能使用 所以使用之前先判断是否为nil
if m1 == nil {
    m1 = make(map[int]string)
}

//1存储键值对到map中  语法:map[key]=value
m1[1]="小猪"
m1[2]="小猫"
```

```go
//判断key是否存在   语法：value,ok:=map[key]
val, ok := m1[1]
fmt.Println(val, ok) //结果返回两个值，一个是当前获取的key对应的val值。二是当前值否存在，会返回一个true或false。

//修改map  如果不存在则添加， 如果存在直接修改原有数据。
m1[1] = "小狗"

//删除map中key对应的键值对数据 语法: delete(map, key)
delete(m1, 1)

// 获取map中的总长度 len(map)
fmt.Println(len(m1))
```

### map的遍历

```go
//map的遍历
因为map是无序的 如果需要获取map中所有的键值对
可以使用 for range
map1 := make(map[int]string)
map1[1] = "张无忌"
map1[2] = "张三丰"
map1[3] = "常遇春"
map1[4] = "胡青牛"

//遍历map
for key, val := range map1 {
    fmt.Println(key, val)
}
```





