<!--
author: vlean
date: 2016-05-06
title: PHP数组内存占用分析
tags: php,array,源码
category: php
status: publish
summary: 通过源码和实例分析php数组内存占用
-->

下面的做法会占用多大的内存？
```php
list($appid,$openid) = ["testcontent","test"];
```


##　0x00 测试

```php
$m0 = memory_get_usage();
$k = range(1,200000);
$m1 = memory_get_usage();
echo round(($m1-$m0)/pow(1024,2),4) ."MB\n";
foreach ($k as $i){
    $n1 = "kk$i";
    $n2 = "tt$i";
    list($$n1,$$n2) = [$i,$i*3];
}
$m2 = memory_get_usage();
echo round(($m2-$m1)/pow(1024,2),4) ."MB\n";

$m1 = memory_get_usage();
foreach ($k as $i){
    $n1 = "kk$i";
    $n2 = "tt$i";
    $$n1 = $i+time();
    $$n2 = 2*time();
}
$m2 = memory_get_usage();
echo round(($m2-$m1)/pow(1024,2),4) ."MB\n";
```
上面运行输出的结果如下：
```bash
27.9404MB
51.3041MB
9.1553MB
```
可见数组占用的内存远大于正常分配的内容

## 0x01 原理

在PHP中都使用long类型来代表数字，没有使用int类型。大家都明白PHP是一种弱类型的语言，它不会去区分变量的类型，没有int float char *之类的概念。我们看看php在zend里面存储的变量，PHP中每个变量都有对应的 zval，Zval结构体定义在Zend/zend.h里面，其结构:
```c
typedef struct _zval_struct zval;  
struct _zval_struct {  
    /* Variable information */  
    zvalue_value value;     /* The value 1 12字节(32位机是12，64位机需要8+4+4=16) */  
    zend_uint refcount__gc; /* The number of references to this value (for GC) 4字节 */  
    zend_uchar type;        /* The active type 1字节*/  
    zend_uchar is_ref__gc;  /* Whether this value is a reference (&) 1字节*/  
}; 
```
PHP使用一种UNION结构来存储变量的值,即zvalue_value 是一个union，UNION变量所占用的内存是由最大成员数据空间决定。
```c
typedef union _zvalue_value {  
    long lval;                  /* long value */  
    double dval;                /* double value */  
    struct {                    /* string value */  
        char *val;  
        int len;  
    } str;   
    HashTable *ht;              /* hash table value */  
    zend_object_value obj;      /*object value */  
} zvalue_value;  
```
最大成员数据空间是struct str，指针占*val用4字节，INT占用4字节，共8字节。
struct zval占用的空间为8+4+1+1 = 14字节，其实呢，在zval中数组，字符串和对象还需要另外的存储结构，数组则是一个 HashTable:

HashTable结构体定义在Zend/zend_hash.h.
```c
typedef struct _hashtable {  
    uint nTableSize;//4  
    uint nTableMask;//4  
    uint nNumOfElements;//4  
    ulong nNextFreeElement;//4  
    Bucket *pInternalPointer;   /* Used for element traversal 4*/  
    Bucket *pListHead;//4  
    Bucket *pListTail;//4  
    Bucket **arBuckets;//4  
    dtor_func_t pDestructor;//4  
    zend_bool persistent;//1  
    unsigned char nApplyCount;//1  
    zend_bool bApplyProtection;//1  
#if ZEND_DEBUG  
    int inconsistent;//4  
#endif  
} HashTable; 
```
HashTable 结构需要 39 个字节，每个数组元素存储在 Bucket 结构中:
```c
typedef struct bucket {  
    ulong h;    /* Used for numeric indexing                4字节 */  
    uint nKeyLength;    /* The length of the key (for string keys)  4字节 */  
    void *pData;        /* 4字节*/  
    void *pDataPtr;         /* 4字节*/  
    struct bucket *pListNext;  /* PHP arrays are ordered. This gives the next element in that order4字节*/  
    struct bucket *pListLast;  /* and this gives the previous element           4字节 */  
    struct bucket *pNext;      /* The next element in this (doubly) linked list     4字节*/  
    struct bucket *pLast;      /* The previous element in this (doubly) linked list     4字节*/  
    char arKey[1];            /* Must be last element   1字节*/  
} Bucket;  
```
Bucket 结构需要 33 个字节，键长超过四个字节的部分附加在 Bucket 后面，而元素值很可能是一个 zval 结构，另外每个数组会分配一个由 arBuckets 指向的 Bucket 指针数组， 虽然不能说每增加一个元素就需要一个指针，但是实际情况可能更糟。这么算来一个数组元素就会占用 54 个字节，与上面的估算几乎一样。

一个空数组至少会占用 14(zval) + 39(HashTable) + 33(arBuckets) = 86 个字节，作为一个变量应该在符号表中有个位置，也是一个数组元素，因此一个空数组变量需要 118 个字节来描述和存储。从空间的角度来看，小型数组平均代价较大，当然一个脚本中不会充斥数量很大的小型数组，可以以较小的空间代价来获取编程上的快捷。但如果将数组当作容器来使用就是另一番景象了，实际应用经常会遇到多维数组，而且元素居多。比如10k个元素的一维数组大概消耗540k内存，而10k x 10 的二维数组理论上只需要 6M 左右的空间，但是按照 memory_get_usage 的结果则两倍于此，[10k,5,2]的三维数组居然消耗了23M，小型数组果然是划不来的。

## 0xff 参考

> [php数组占用内存大小分析](http://blog.csdn.net/hguisu/article/details/7376705)