release: latest

# data structure sds in redis

sds: simple dynamic string. 类型 sds 的声明为

``` c
typedef char*sds;
```

为 sds 申请动态内存的时候，申请了一块连续内存，该内存头部包含了 sds 的头部信息，并且维护该头部信息，源码里称为 sds_hdr_var， sds的 header variable

最新版本的 redis 一共实现了五种相似的 sds 结构体，分别为 sdshdr5（不建议使用）， sdshdr8， sdshdr16， sdshdr32， sdshdr64，以 sdshdr5 和 sdshdr8 为例

``` c
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```

1. struct 声明中使用了 GNU C 编译器的结构体注释，放弃编译器对字节对齐的优化；
2. 变量 flags 中 3 lsb (低三位) 表示了该结构体变量的类型，一共五种，所以只需要低三位；
3. 变量 len 为该 string 已经使用的字符串长度，不包含 C 风格字符串的 '\000'；
4. 变量 alloc 为该 string 分配的字符串长度，包含了 C 风格字符串中的 '\000'，
5. 变量 buf 为空，实际上代表结构体 char *sds 的位置，与头部信息相连。

``` C
#define SDS_TYPE_5  0
#define SDS_TYPE_8  1
#define SDS_TYPE_16 2
#define SDS_TYPE_32 3
#define SDS_TYPE_64 4
#define SDS_TYPE_MASK 7
```

宏变量，方便进行位操作

``` c
#define SDS_TYPE_BITS 3
#define SDS_TYPE_5_LEN(f) ((f)>>SDS_TYPE_BITS)
```

宏变量，低三位，移位可以获取 type

``` c
#define SDS_HDR_VAR(T,s) struct sdshdr##T *sh = (void*)((s)-(sizeof(struct sdshdr##T)));
#define SDS_HDR(T,s) ((struct sdshdr##T *)((s)-(sizeof(struct sdshdr##T))))
```

第一个宏函数，声明一个指针 sh，指向 sdshdr##T 类型，sds 变量 s 的指针，该指针指向头部信息

第二个宏函数，同第一个，但不显示声明指针，只是获取地址

通过位操作位偏移操作，可以非常方便的获取 sds 变量的头部信息

``` c
unsigned char flags = s[-1];
switch(flags&SDS_TYPE_MASK)
```



## 优点

保存了 string 信息，二进制安全，同时使用了大量位操作，加快运算。

sds 中保存的实际上是 C 风格字符串，可以使用 C 标准库接口

# sds 模块依赖

``` makefile
sds.o: sds.c sds.h sdsalloc.h zmalloc.h \
 ../deps/jemalloc/include/jemalloc/jemalloc.h
```

依赖为 redis 实现的 zmalloc，zmalloc 是 redis 为 jemalloc 封装的自己内部模块的申请动态内存模块，jemalloc 为优秀的用于高并发用于替代 malloc 的开源库

sdsalloc.h 为 sds 模块与 zmalloc 模块的定义接口文件



