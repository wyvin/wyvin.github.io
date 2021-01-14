# golang的struct比较


struct可以比较，但有一些条件需要注意

## 相同类型
必须是相同类型的struct
```golang
type A struct {
  Num int
}

type B struct {
  Num int
}

a := A{}
b := B{}

fmt.Println(a==b)
```
不同类型的struct比较的话会编译不通过：`invalid operation: a == b (mismatched types A and B)`

## 不可比较的类型

当strct里有一下3种类型时，则不可比较
 - map
 - func
 - slice

```golang
type A struct {
	M map[int]int
}

a := A{}
b := A{}

fmt.Println(a==b)
```
以上代码编译时会报错：`invalid operation: a == b (struct containing map[int]int cannot be compared)`

<br>

可以到源码里找到相应的代码
`/cmd/compile/internal/gc/typecheck.go`
```golang
func typecheck1() // 检查类型
  func IncomparableField() // 返回结构体不可比较的字段，如果有的话
    func IsComparable()  // 判断字段的类型是否可以比较
      func algtype1()  // 判断类型是否可比较
        ...
        // 类型不可比较时返回
        // 每个类型都有一个flags属性，该属性包含了Noalg判断是否可以比较
        // slice在初始化的时候会SetNoalg(true)
        // map在创建hmap时会SetNoalg(true)
        // func在创建匿名函数也会SetNoalg(true)
        if t.Noalg() {
		      return ANOEQ, t
	      }

        // 以防万一下面还会有一次判断
        switch t.Etype {
          // func和map
          case TFUNC, TMAP:
		        return ANOEQ, t
          // slice
          case TSLICE:
		        return ANOEQ, t
        }
```

同理我们平时比较的时候，也不能用以上3种去比较

> 另外，若struct包含array或struct，那么array和struct也不能有以上3种类型
