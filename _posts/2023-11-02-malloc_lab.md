---
layout: post
title: malloc_lab
categories: csapp
description: malloc
keywords: lab
---
# malloc_lab

实验要求：

尽可能少的全局变量

隐式空闲链表：块大小+00a+有效载荷和填充

每个堆块使用边界标记法。头部大小为4字节，前29位表示块大小，后3位表示这个块是否空闲；脚部(ftr)是头部(hdr)的副本。目的是**将合并前面的堆块时的搜索时间降到常数**。

使用两种方法：

- 隐式空闲链表 + 首次适配/下一次适配。
- 显示空闲链表 + 分离的空闲链表 + 分离适配

![123](https://img1.imgtp.com/2023/10/26/VTJrNOPM.jpg)

## 隐式空闲链表

### 宏定义

```c
#define FIND_FIT_CHOOSE 2
/*1 for first_fit   首次适配
  2 for next_fit	下次适配
  3 for best_fit	最佳适配
  选择哪个算法
*/

/* single word (4) or double word (8) alignment */
#define ALIGNMENT 8
//本实验设置双字对齐，将8字节设置为双字
/* rounds up to the nearest multiple of ALIGNMENT */
#define ALIGN(size) (((size) + (ALIGNMENT - 1)) & ~0x7)
//(ALIGNMENT - 1)是指计算一个最接近ALIGNMENT的上限值 将size向上舍入下一个ALIGNMENT的倍数
//& ~0x7将其变为11111000，与上面的&按位清零
#define SIZE_T_SIZE (ALIGN(sizeof(size_t)))

#define WSIZE 4
#define DSIZE 8
#define CHUNKSIZE (1 << 12) 
//扩展堆时的默认大小
#define MINBLOCKSIZE 16

#define MAX(x, y) ((x) > (y) ? (x) : (y))
#define PACK(size, alloc) ((size) | (alloc))
//使用位操作将两个参数信息合并到单一值中，以便将其保存到内存块的头部
#define GET(p) (*(unsigned int *)(p))              
//unsigned int让其对齐 指针p指向的值
#define PUT(p, val) (*(unsigned int *)(p) = (val)) 

#define GET_SIZE(p) (GET(p) & ~0x7) 
//指针指向的内容是整个块的大小，后三位要放alloc值,包含头尾结点
#define GET_ALLOC(p) (GET(p) & 0x1) 
//用掩码提取最低位
#define HDRP(bp) ((char *)(bp)-WSIZE)                        
//从指向内存块的指针bp往前指四个字节 heap_listp往前指四个字节指向头结点
#define FTRP(bp) ((char *)(bp) + GET_SIZE(HDRP(bp)) - DSIZE) 
//结尾块往前指八个字节，指向ftr
#define NEXT_BLKP(bp) ((char *)(bp) + GET_SIZE(HDRP(bp)))         /* given block ptr bp, compute address of next blocks */
#define PREV_BLKP(bp) ((char *)(bp)-GET_SIZE((char *)(bp)-DSIZE)) /* given block ptr bp, compute address of prev blocks */
```

![image-20231026142503327](images\image-20231026142503327.png)

### 1.首次适配

从头开始搜索，找到第一个适配点

代码实现如下：

```c
static void *first_fit(size_t asize)  //首次适配
{
    for (char *bp = heap_listp; GET_SIZE(HDRP(bp)) > 0; bp = NEXT_BLKP(bp))
    {
        if (!GET_ALLOC(HDRP(bp)) && GET_SIZE(HDRP(bp)) >= asize)
        {
            return bp;
        }
    }
    return NULL;
}
```

### 2.下次适配

第一个循环是找查询后指针之后的空闲

第二个循环是从头找到查询指针

```c
static void *next_fit(size_t asize) //下次适配
{
     /* Next fit search */
    char *oldrover = rover;

    /* Search from the rover to the end of list */
    for ( ; GET_SIZE(HDRP(rover)) > 0; rover = NEXT_BLKP(rover))
        if (!GET_ALLOC(HDRP(rover)) && (asize <= GET_SIZE(HDRP(rover))))
            return rover;
	
	/* Divide and Conquer */
    /* search from start of list to old rover */
    for (rover = heap_listp; rover < oldrover; rover = NEXT_BLKP(rover))
        if (!GET_ALLOC(HDRP(rover)) && (asize <= GET_SIZE(HDRP(rover))))
            return rover;

    return NULL;  /* no fit found */
}
```

### 3.最佳适配

查找最小的块

```c
static void *best_fit(size_t asize) //最佳适配
{
    void *bp;
    void *best_bp=NULL;
    size_t min_size=0;
    for(bp=heap_listp;GET_SIZE(HDRP(bp))>0;bp=NEXT_BLKP(bp))
    {
        if((GET_SIZE(HDRP(bp))>=asize)&&(!GET_ALLOC(HDRP(bp))))
        {
            if(min_size>GET_SIZE(HDRP(bp))||min_size==0)//==0是初始情况
            {
                min_size=GET_SIZE(HDRP(bp));//找到最小块
                best_bp=bp;
            }
        }
    }
    return best_bp;
}
```

### 4.选择方法

```c
static void* find_fit(size_t asize)
{
    if(FIND_FIT_CHOOSE==1)
        return first_fit(asize);
    else if (FIND_FIT_CHOOSE==2)
        return next_fit(asize);
    else
        return best_fit(asize);
}
```

### 5.合并函数

```c
static void *coalesce(void *bp)
{
    size_t prev_alloc = GET_ALLOC(FTRP(PREV_BLKP(bp)));  
    size_t next_alloc = GET_ALLOC(HDRP(NEXT_BLKP(bp)));
    size_t size = GET_SIZE(HDRP(bp));

    if (prev_alloc && next_alloc) // 前后块均被占用 
        return bp;
    else if (prev_alloc && !next_alloc) // 前块被占用，后块空闲，向后合并
    {
        size += GET_SIZE(HDRP(NEXT_BLKP(bp))); 
        PUT(HDRP(bp), PACK(size, 0));
        PUT(FTRP(bp), PACK(size, 0));
    }
    else if (!prev_alloc && next_alloc) // 前块空闲，后块被占用，向前合并
    {
        size += GET_SIZE(HDRP(PREV_BLKP(bp)));
        PUT(FTRP(bp), PACK(size, 0));
        PUT(HDRP(PREV_BLKP(bp)), PACK(size, 0));
        bp = PREV_BLKP(bp);
    }
    else //前后均空闲，向两端合并
    {
        size += GET_SIZE(HDRP(PREV_BLKP(bp))) + GET_SIZE(FTRP(NEXT_BLKP(bp)));
        PUT(HDRP(PREV_BLKP(bp)), PACK(size, 0));
        PUT(FTRP(NEXT_BLKP(bp)), PACK(size, 0));
        bp = PREV_BLKP(bp);
    }

    if((rover>(char*)bp)&&(rover<NEXT_BLKP(bp)))
    {
        rover=bp;
    }
    return bp;
}
```

### 6.拓展堆

```c
static void *extend_heap(size_t words)
{
    char *bp; // 指向堆空间中的块的指针
    size_t size; // 要扩展的堆空间的大小

    
    size = (words % 2) ? (words + 1) * WSIZE : words * WSIZE; // 确保分配的字节数是偶数倍的字（以保持对齐）
    if ((long)(bp = mem_sbrk(size)) == -1)
        return NULL;

    /*  初始化块头和块尾 */
    PUT(HDRP(bp), PACK(size, 0));         // 设置新块的头部
    PUT(FTRP(bp), PACK(size, 0));         // 设置新块的尾部
    PUT(HDRP(NEXT_BLKP(bp)), PACK(0, 1)); // 设置结尾块的头部

    return coalesce(bp); // 调用coalesce函数对新分配的块进行合并操作
}
```

### 7.设置占用块

```c
//把块的前asize字节设成已占用
static void split_block(void *bp, size_t asize)
{
    size_t size = GET_SIZE(HDRP(bp));

    if ((size - asize) >= MINBLOCKSIZE)
    {
        PUT(HDRP(bp), PACK(asize, 1));//把个数和alloc合在一起
        PUT(FTRP(bp), PACK(asize, 1));
        bp = NEXT_BLKP(bp);
        PUT(HDRP(bp), PACK(size - asize, 0));
        PUT(FTRP(bp), PACK(size - asize, 0));
    }
}
```

### 8.初始化

```c
int mm_init(void)
{
    if ((heap_listp = mem_sbrk(4 * WSIZE)) == (void *)-1)
        return -1;
    PUT(heap_listp, 0);                            // alignment padding
    PUT(heap_listp + (1 * WSIZE), PACK(DSIZE, 1)); // prologue header
    PUT(heap_listp + (2 * WSIZE), PACK(DSIZE, 1)); // prologue footer
    PUT(heap_listp + (3 * WSIZE), PACK(0, 1));     // epilogue header
    heap_listp += (2 * WSIZE);

    rover=heap_listp; /*initialize rover*/

    if (extend_heap(CHUNKSIZE / WSIZE) == NULL)
        return -1;
    return 0;
}
```

### malloc

```c
void *mm_malloc(size_t size)
{
    size_t asize;      // 经过对齐和标记处理后的要分配的块的大小
    size_t extendsize; // 如果没有合适的空闲块可用时，要扩展的堆的大小
    char *bp;

    if (size == 0)
        return NULL;

    // 调整块大小，包括对齐和标记位
    if (size <= DSIZE)
        asize = 2 * DSIZE; // 满足最小块大小的要求，也就是16字节
    else
        asize = DSIZE * ((size + (DSIZE) + (DSIZE - 1)) / DSIZE); 
        /* size+DSIZE代表了给块补上了头部和脚部的大小，asize这行代码是为了给size+DSIZE向上取最近的8的倍数，保证双字对齐 */


    if ((bp = find_fit(asize)) != NULL) // 通过适配的算法找到最合适的空闲块
    {
        place(bp, asize); 
        return bp;
    }

    // 没有合适的空闲块，需要扩展堆空间
    extendsize = MAX(asize, CHUNKSIZE);
    if ((bp = extend_heap(extendsize / WSIZE)) == NULL)
        return NULL;
    place(bp, asize);
    return bp;
}
```

### free&realloc

```c
void mm_free(void *bp)
{
    size_t size = GET_SIZE(HDRP(bp));

    PUT(HDRP(bp), PACK(size, 0));
    PUT(FTRP(bp), PACK(size, 0));
    coalesce(bp);
}

/*
 * mm_realloc - Implemented simply in terms of mm_malloc and mm_free
 */
void *mm_realloc(void *ptr, size_t size) 
{
    void *oldptr = ptr; // oldptr作为指向原始内存块的指针
    void *newptr;  // newptr是指向重新分配后的内存块的新指针
    size_t copySize; // copySize表示需要复制的数据大小

    newptr = mm_malloc(size);
    if (newptr == NULL)
        return NULL;
    /*获取原始内存块的大小size和重新分配后内存块的大小copySize。
    然后，比较这两个大小，将较小的值赋给copySize，以确保不会超过原始内存块的大小*/
    size = GET_SIZE(HDRP(oldptr));
    copySize = GET_SIZE(HDRP(newptr));
    if (size < copySize)
        copySize = size;
    memcpy(newptr, oldptr, copySize - WSIZE);
    mm_free(oldptr);
    return newptr;
}
```

make编译后执行./mdriver -t ./traces -V

## 显式空闲链表

在宏定义方面多出了

```c
#define GET_PREV(p) (*(unsigned int *)(p))
#define SET_PREV(p, prev) (*(unsigned int *)(p) = (prev))
#define GET_NEXT(p) (*((unsigned int *)(p)+1))
#define SET_NEXT(p, val) (*((unsigned int *)(p)+1) = (val))
```



```c
static void remove_from_free_list(void *bp)
{
    if (bp == NULL || GET_ALLOC(HDRP(bp)))
        return;

    void* prev = GET_PREV(bp);
    void* next = GET_NEXT(bp);

    SET_PREV(bp, 0);
    SET_NEXT(bp, 0);

    if (prev == NULL && next == NULL)
    {
        free_list_head = NULL;
    }
    else if (prev == NULL)
    {
        SET_PREV(next, 0);
        free_list_head = next;
    }
    else if (next == NULL)
    {
        SET_NEXT(prev, 0);
    }
    else
    {
        SET_NEXT(prev, next);
        SET_PREV(next, prev);
    } 
}
```



```c
static void insert_to_free_list(void* bp)
{
    if (bp == NULL)
        return;

    if (free_list_head == NULL)
    {
        free_list_head = bp;
        return;
    }

    SET_NEXT(bp, free_list_head);
    SET_PREV(free_list_head, bp);
    free_list_head = bp;
}
```

合并

```c
static void* coalesce(void *bp)
{
    void* prev_bp = PREV_BLKP(bp);
    void* next_bp = NEXT_BLKP(bp);
    size_t prev_alloc = GET_ALLOC(FTRP(prev_bp));
    size_t next_alloc = GET_ALLOC(HDRP(next_bp));

    size_t size = GET_SIZE(HDRP(bp));

    if (prev_alloc && next_alloc)
    {

    }
    else if (prev_alloc && !next_alloc)
    {
        remove_from_free_list(next_bp);
        size += GET_SIZE(HDRP(next_bp));
        PUT(HDRP(bp), PACK(size, 0));
        PUT(FTRP(bp), PACK(size, 0));
    }
    else if (!prev_alloc && next_alloc)
    {
        remove_from_free_list(prev_bp);
        size += GET_SIZE(HDRP(prev_bp));
        PUT(FTRP(bp), PACK(size, 0));
        PUT(HDRP(prev_bp), PACK(size, 0));

        bp = prev_bp;
    }
    else
    {
        remove_from_free_list(prev_bp);
        remove_from_free_list(next_bp);
        size += GET_SIZE(HDRP(prev_bp)) + GET_SIZE(FTRP(next_bp));
        PUT(HDRP(prev_bp), PACK(size, 0));
        PUT(FTRP(next_bp), PACK(size, 0));

        bp = prev_bp;
    }
    insert_to_free_list(bp);
    return bp;
}
```
