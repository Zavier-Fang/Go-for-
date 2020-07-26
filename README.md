# Go for-range之坑

## 1. 遍历元素取不到元素地址
如下面代码所示，在遍历对元素取址：

```go
arr := [2]int{1, 2}
res := []*int{}
for _, v := range arr {
    res = append(res, &v)
}
//expect: 1 2
fmt.Println(*res[0],*res[1])
//but output: 2 2
```

结果是，只拿到了最后一个元素的地址的值。 for-range 其实是语法糖，内部调用还是 for 循环，
初始化会拷贝带遍历的列表（如 array，slice，map），然后每次遍历的v都是对同一个元素的遍历赋值。
也就是说如果直接对v取地址，最终只会拿到一个地址，而对应的值就是最后遍历的那个元素所附给v的值。对应伪代码如下：

```go
 len_temp := len(range)
 range_temp := range
 for index_temp = 0; index_temp < len_temp; index_temp++ {
     value_temp = range_temp[index_temp]
     index = index_temp
     value = value_temp
     original body
 }
```

修改的方式有两种：
* 使用局部变量拷贝v

```go
for _, v := range arr {
    //局部变量v替换了v，也可用别的局部变量名
    v := v
    res = append(res, &v)
}
```

* 通过索引取址：

```go
//这种其实退化为for循环的简写
for k := range arr {
    res = append(res, &arr[k])
}
```

## 2. 遍历时同时增加元素，遍历会停止吗
如下图：

```go
v := []int{1, 2, 3}
for i := range v {
    v = append(v, i)
}
```

答案是【会】，因为遍历前对v做了拷贝，所以期间对原来v的修改不会反映到遍历中

## 3. 对大数组这样的遍历有问题吗？

```go
//假设值都为1，这里只赋值3个
var arr = [102400]int{1, 1, 1}
for i, n := range arr {
    //just ignore i and n for simplify the example
    _ = i
    _ = n
}
```

答案是【有问题】！遍历前的拷贝对内存是极大浪费啊 怎么优化？有两种

* 对数组取地址遍历for i, n := range &arr
* 对数组做切片引用for i, n := range arr[:]

反思题：对大量元素的 slice 和 map 遍历为啥不会有内存浪费问题？（提示，底层数据结构是否被拷贝）

##4. 对大数组这样重置效率高么？

```go
//假设值都为1，这里只赋值3个
var arr = [102400]int{1, 1, 1}
for i, _ := range &arr {
    arr[i] = 0
}
```

答案是【高】，这个要理解得知道 go 对这种重置元素值为默认值的遍历是有优化的:

```go
// Lower n into runtime·memclr if possible, for
// fast zeroing of slices and arrays (issue 5373).
// Look for instances of
//
// for i := range a {
//  a[i] = zero
// }
//
// in which the evaluation of a is side-effect-free.
```

##5. 对 map 遍历时删除元素能遍历到么？

```go
var m = map[int]int{1: 1, 2: 2, 3: 3}
//only del key once, and not del the current iteration key
var o sync.Once
for i := range m {
    o.Do(func() {
        for _, key := range []int{1, 2, 3} {
            if key != i {
                fmt.Printf("when iteration key %d, del key %d\n", i, key)
                delete(m, key)
                break
            }
        }
    })
    fmt.Printf("%d%d ", i, m[i])
}
```

答案是【不会】 map 内部实现是一个链式 hash 表，为保证每次无序，初始化时会随机一个遍历开始的位置, 这样，如果删除的元素开始没被遍历到（上边once.Do函数内保证第一次执行时删除未遍历的一个元素），那就后边就不会出现。

##6. 对 map 遍历时新增元素能遍历到么？

```go
var m = map[int]int{1:1, 2:2, 3:3}
for i, _ := range m {
    m[4] = 4
    fmt.Printf("%d%d ", i, m[i])
}
```

答案是【可能会】，输出中可能会有44。原因同上一个, 可以用以下代码验证。

```go
var createElemDuringIterMap = func() {
    var m = map[int]int{1: 1, 2: 2, 3: 3}
    for i := range m {
        m[4] = 4
        fmt.Printf("%d%d ", i, m[i])
    }
}
for i := 0; i < 50; i++ {
    //some line will not show 44, some line will
    createElemDuringIterMap()
    fmt.Println()
}
```

##7. 这样遍历中起 goroutine 可以么？

```go
var m = []int{1, 2, 3}
for i := range m {
    gofunc() {
        fmt.Print(i)
    }()
}
//block main 1ms to wait goroutine finished
time.Sleep(time.Millisecond)
```

答案是【不可以】。预期输出 0,1,2 的某个组合，如 012，210.. 结果是 222. 同样是拷贝的问题 怎么解决?

* 以参数方式传入
```go
for i := range m {
    gofunc(i int) {
        fmt.Print(i)
    }(i)
}
```
* 使用局部变量拷贝
```go
for i := range m {
    i := i
    gofunc() {
        fmt.Print(i)
    }()
}
```
