## GO语音之切片
#### 1. 切片简介  
切片（slice）是Golang中数组之上的抽象，可按需自动增长与缩小。切片的底层是连续的内存块。  
切片本身是一个三个字段的数据结构：
- Data：指向底层数组的指针；
- Len：切片中元素的个数；len(s)获取;
- Cap：切片的容量（不需重新分配内存前，可容纳的元素数量）；使用方法cap(s)获取；

```
type SliceHeader struct {
    Data uintptr
    Len  int
    Cap  int
}
```

#### 2. 创建与初始化  
- make函数创建  

```
// 创建长度为0，容量为10的切片
s := make([]int, 0, 10)
 
```

- 初始化创建  
根据给定的数据，自动创建并初始化切

```
// 长度、容量都为3的切片
s := []int{1,2,3}
 
// 长度、容量都为10，最后一个元素为99，其他为默认值0
s2 := []int{9: 99}

```

#### 3. 空切片与nil切片  
声明时不做任何初始化动作会创建一个nil切片；而空切片是底层数组包含0个元素，且没有分配存储空间的切片。nil切片与空切片都可直接获取长度（len）或追加元素（append）。  

```
// nil切片
var s []int
 
// 空切片
s1 := make([]int, 0)
s2 := []int{}

```
nil空切片引用数组指针地址为0（无指向任何实际地址）。空切片的引用数组指针地址是有的，且固定为一个值，即所有的空切片指向的数组引用地址都是一样的。

#### 4. 共享底层数组的切片  
在已有数组/切片上切出一部分可以生成新的切片，他们共享底层的数组；因此修改一个切片的内容（在不扩容的情况下），会影响到其他共享底层数组的切片。

```
// 元素范围为[i,j)，k限制容量为（k-i）
s := src[i:j:k]
 
// 从i到尾部
s := src[i:]
 
// 从头到j（不包括j）
s := src[:j]
 
// 从头到尾（复制整个切片），但底层数组还是公用的
s := src[:]


```

#### 5. 迭代  
切片是一个集合，可以通过range迭代其中的元素：

```
for index, value := range myS{
    fmt.Printf("index: %d, value: %d\n", index, value)
}


```
range返回的第二个值是对应元素的一份副本，不能用于修改；若要修改则需要通过索引：  

```
for index, _ := range myS{
    myS[index] += 1
}

```

#### 6. 不定参数传递  
切片可直接给方法传递‘不定参数’

```
func func1() {
  var s []string
  func2(s...)
}
 
func func2(prams ...string) {
  log.Println(prams)
}

```

#### 7. Append  
通过函数append可在切片尾部追加元素；通过copy(dest,src)可复制切片，复制的长度是minimum of len(src) and len(dst)。

可对nil切片追加元素，以及求长度：

```
var sl []int            // nil
sl = append(sl, 1)      // [1]

// 尾部插入
a = append(a, 1, 2, 3)
a = append(a, []int{1,2,3}...) // 追加切片时，需要使用...解包

// 头部插入,会引起内存的分配与复制操作
a = append([]int{1}, a...)
a = append([]int{1,2,3}, a...)

```

#### 8. 扩容  
Slice依托数组实现，底层数组容量不足时可自动重新分配。追加数据通过append（返回值一定要再赋值给原slice）；

```
    var s []int // s == nil
    fmt.Println(s) // []
    fmt.Println(len(s)) // 0
    for i := 0; i < 5; i++ {
        s = append(s, i+1)
    }
    // 容量：0，1，2，4，8


```
#### 9. 扩容原理  
在Go语言中使用append()函数向Slice添加元素，扩容也是发生在append的调用中，当切片内部的容量，不足以容纳新增元素时就会触发Slice的扩容。
数组本质是不可扩容的，数组的扩容实际上就是创建新的数组，分配新的内存，然后执行数组的拷贝，所以slice实际上就需要数组新的内存地址的返回，指针指向新的内存地址。  
并非每次调用append()函数都会触发扩容，因为扩容涉及到内存分配，会减缓append的速度。  

```
func SliceExpansion() {
  sliceA := make([]int, 3, 4)
  sliceA = append(sliceA, 1) // 未扩容
  fmt.Printf("SliceA len: %v, cap: %v\n", len(sliceA), cap(sliceA))
  sliceB := make([]int, 3, 3)
  sliceB = append(sliceB, 1) // 触发扩容
  fmt.Printf("SliceB len: %v, cap: %v\n", len(sliceB), cap(sliceB))
}


```
可见SliceA 初始len=3，初始cap=4；SliceB 初始len=3，初始cap=3，向SliceA 追加一个元素时，由于SliceA追加元素后len<cap并未发生扩容，而向SliceB 追加一个元素时，由于SliceB追加元素后len>=cap发生扩容，且扩容后容量为6，看上去好像扩容就是在原来Slice长度的基础上增加一倍容量，事实上也是如此吗？  
策略：  
- 策略一：如果新申请的容量(cap)大于2倍的旧容量(old.cap),则最终容量(nwecap)是新申请的容量。
- 策略二：如果不满足策略一
如果旧切片len<1024,则最终容量是就容量的2倍，即newcap = doublecap。
如果旧切片len>=1024,则最终容量循环增加1.25倍，直至newcap > cap 为止。
- 策略三：在进行循环1.25倍计算时，最终容量计算值发生溢出，即超过了int的最大范围，则最终容量就是新申请的容量。