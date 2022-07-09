# ```intset```



整数集合```intset```是集合键值的底层实现之一, 当一个集合只包含整数值元素时, 且该集合元素数量不多, ```redis```使用整数集合作为集合键的底层实现。



```intset```在```intset.h```中被定义

```c
typedef struct intset {
    
    // 编码方式
    uint32_t encoding;

    // 集合包含的元素数量
    uint32_t length;

    // 保存元素的数组
    int8_t contents[];

} intset;
```

看上去只是一个普通的动态数组, ```contents```中保存类型为```int8_t```类型的元素.

但是实际上```contents```数组并不保存任何```int8_t```类型的值

```encoding```有```INTSET_ENC_INT16,INTSET_ENC_INT32,INTSET_ENC_INT64```三种取值, 标识着```contents```的真实类型.

注意, ```intset```中的所有属性都以大端序来存储, 所以如果需要计算还需要转化为小端序.

### 升级

没当需要加入一个新元素到整数集合中, 且新元素的类型比整数集合现有的所有元素类型都长时, 整数集合需要进行升级.

升级整数集合并添加新元素分为三部

Ⅰ. 根据新元素类型, 扩展整数集合底层数组的空间大小, 并为新元素分配空间

Ⅱ. 将底层数组现有的所有元素都转换为与新元素相同的类型, 并将类型转换后的元素放在正确的位上(维护有序性)

由此可以看出, 添加新元素的时间复杂度为```O(n)```

升级的优点当然就是, 在保持能存储任意类型数值情况下, 尽量节约内存.

而且整数结合一旦升级就不会降级, 即使原来引起升级的那个整数被删掉了.



### 源码分析

#### intsetNew

初始化一个```intset```对象

```c
#define INTSET_ENC_INT16 (sizeof(int16_t))

intset *intsetNew(void) {

    // 为整数集合结构分配空间
    intset *is = zmalloc(sizeof(intset));

    // 设置初始编码
    is->encoding = intrev32ifbe(INTSET_ENC_INT16);

    // 初始化元素数量
    is->length = 0;

    return is;
}
```

值得一提的是·```is->encoding```以大端序存储, 所以需要用```intrev32ifbe```转化一下

```c
#define intrev32ifbe(v) intrev32(v)

uint32_t intrev32(uint32_t v) {
    memrev32(&v);
    return v;
}

// 从小端转化为大端
void memrev16(void *p) {
    unsigned char *x = p, t;

    t = x[0];
    x[0] = x[1];
    x[1] = t;
}
```

#### intsetAdd

```c
/* Insert an integer in the intset 
 * 
 * 尝试将元素 value 添加到整数集合中。
 *
 * *success 的值指示添加是否成功：
 * - 如果添加成功，那么将 *success 的值设为 1 。
 * - 因为元素已存在而造成添加失败时，将 *success 的值设为 0 。
 *
 * T = O(N)
 */
intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {

    // 计算编码 value 所需的长度
    uint8_t valenc = _intsetValueEncoding(value);
    uint32_t pos;

    // 默认设置插入为成功
    if (success) *success = 1;

    /* Upgrade encoding if necessary. If we need to upgrade, we know that
     * this value should be either appended (if > 0) or prepended (if < 0),
     * because it lies outside the range of existing values. */
    // 如果 value 的编码比整数集合现在的编码要大
    // 那么表示 value 必然可以添加到整数集合中
    // 并且整数集合需要对自身进行升级，才能满足 value 所需的编码
    if (valenc > intrev32ifbe(is->encoding)) {
        /* This always succeeds, so we don't need to curry *success. */
        // T = O(N)
        return intsetUpgradeAndAdd(is,value);
    } else {
        // 运行到这里，表示整数集合现有的编码方式适用于 value

        /* Abort if the value is already present in the set.
         * This call will populate "pos" with the right position to insert
         * the value when it cannot be found. */
        // 在整数集合中查找 value ，看他是否存在：
        // - 如果存在，那么将 *success 设置为 0 ，并返回未经改动的整数集合
        // - 如果不存在，那么可以插入 value 的位置将被保存到 pos 指针中
        //   等待后续程序使用
        if (intsetSearch(is,value,&pos)) {
            if (success) *success = 0;
            return is;
        }

        // 运行到这里，表示 value 不存在于集合中
        // 程序需要将 value 添加到整数集合中
    
        // 为 value 在集合中分配空间
        is = intsetResize(is,intrev32ifbe(is->length)+1);
        // 如果新元素不是被添加到底层数组的末尾
        // 那么需要对现有元素的数据进行移动，空出 pos 上的位置，用于设置新值
        // 举个例子
        // 如果数组为：
        // | x | y | z | ? |
        //     |<----->|
        // 而新元素 n 的 pos 为 1 ，那么数组将移动 y 和 z 两个元素
        // | x | y | y | z |
        //         |<----->|
        // 这样就可以将新元素设置到 pos 上了：
        // | x | n | y | z |
        // T = O(N)
        if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1);
    }

    // 将新值设置到底层数组的指定位置中
    _intsetSet(is,pos,value);

    // 增一集合元素数量的计数器
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);

    // 返回添加新元素后的整数集合
    return is;

    /* p.s. 上面的代码可以重构成以下更简单的形式：
    
    if (valenc > intrev32ifbe(is->encoding)) {
        return intsetUpgradeAndAdd(is,value);
    }
     
    if (intsetSearch(is,value,&pos)) {
        if (success) *success = 0;
        return is;
    } else {
        is = intsetResize(is,intrev32ifbe(is->length)+1);
        if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1);
        _intsetSet(is,pos,value);

        is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
        return is;
    }
    */
}

static uint8_t _intsetValueEncoding(int64_t v) {
    if (v < INT32_MIN || v > INT32_MAX)
        return INTSET_ENC_INT64;
    else if (v < INT16_MIN || v > INT16_MAX)
        return INTSET_ENC_INT32;
    else
        return INTSET_ENC_INT16;
}
```

重点可以看一下当插入的新值需要引起升级时的操作

```c
static intset *intsetUpgradeAndAdd(intset *is, int64_t value) {
    
    // 当前的编码方式
    uint8_t curenc = intrev32ifbe(is->encoding);

    // 新值所需的编码方式
    uint8_t newenc = _intsetValueEncoding(value);

    // 当前集合的元素数量
    int length = intrev32ifbe(is->length);

    // 根据 value 的值，决定是将它添加到底层数组的最前端还是最后端
    // 注意，因为 value 的编码比集合原有的其他元素的编码都要大
    // 所以 value 要么大于集合中的所有元素，要么小于集合中的所有元素
    // 因此，value 只能添加到底层数组的最前端或最后端
    int prepend = value < 0 ? 1 : 0;

    /* First set new encoding and resize */
    // 更新集合的编码方式
    is->encoding = intrev32ifbe(newenc);
    // 根据新编码对集合（的底层数组）进行空间调整
    // T = O(N)
    is = intsetResize(is,intrev32ifbe(is->length)+1);

    // 当插入元素在最后时, 此时prepend为0, 相当于从后往前把之前所有元素往后移到正确的位置
    // 当插入元素在最前时, 此时prepend为1, 和上面同理.只不过所有元素的相对位置往后一位
    while(length--)
        _intsetSet(is,length+prepend,_intsetGetEncoded(is,length,curenc));

    /* Set the value at the beginning or the end. */
    // 设置新值，根据 prepend 的值来决定是添加到数组头还是数组尾
    if (prepend)
        _intsetSet(is,0,value);
    else
        _intsetSet(is,intrev32ifbe(is->length),value);

    // 更新整数集合的元素数量
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);

    return is;
}
```



