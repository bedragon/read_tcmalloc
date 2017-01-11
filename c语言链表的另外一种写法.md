最近看tcmalloc 当看到PageHeadAllocator关于链表的写法很诧异，居然还能这样写（一般情况写链表肯定得有个next但它没有）。so我就研究了一番
```
#include <stdio.h>
#include <malloc.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>

void insert(void **list, void *p){
    *((void **)p) = *list; //让p的前8字节保存list地址
    *list = p;//list始终保持自己是头部 so指向了p
   
}
void *getHead(void **list){
    if(list == NULL){
        return NULL;
    }
    void *result = *list;
    *list = *((void **)result); //让list拿到result前8字节指向的next地址
    return result;

}
struct AAA{
    uint64_t pos; //占位 占8位保存下一个变量的地址，64位OS需要8字节 32位需要4字节这里可以写成通用的 void *pos;
    char str[32];

};

int main(void) {
    void *linkedList = NULL;
    struct AAA p1;
    strcpy(p1.str, "aaa");
    insert(&linkedList, (void*)&p1);
    struct AAA p2;
    strcpy(p2.str, "bbb");
    insert(&linkedList, (void*)&p2);
    struct AAA p3;
    strcpy(p3.str, "ccc");
    insert(&linkedList, (void*)&p3);
    struct AAA* tmp  = (struct AAA *)getHead(&linkedList);
    printf("%s\n", tmp->str);
    tmp  = (struct AAA *)getHead(&linkedList);
    printf("%s\n", tmp->str);
    tmp  = (struct AAA *)getHead(&linkedList);
    printf("%s\n", tmp->str);

    //另外一种非结构体写法
    void* strHead = NULL;
    char a[256];
    strcpy(a+8, "aaa"); //+8为了占8位 64位操作系统占8字节 32位占4字节，占位为了存next指向变量地址
    insert(&strHead, , (void*)&a);
    char b[256];
    strcpy(b+8, "bbb");
    insert(&strHead, , (void*)&b);
    return 0;
}
```
最终打印结果
[root@docker100204 test_c]# ./a.out
ccc
bbb
aaa

为什么会有这样的效果呢，最主要上面的写法让指针实际指向地址前8字节保存了下个指针的地址，这样就达到了链表next的目的。

对于多级指针或者数组，要掌握正确的识别方法：
void*  是说: 这是一个指针,去掉一个(*)就是它所指向的，在这里是指向放void型的地方；
void**  是说: 这也是一个指针,去掉一个(*)就是它所指向的，它指向一个放void*型的地方.

一级指针是找到地址，然后取得这个地址的值
2级指针是找到地址，然后找到地址的内容的地址，然后再取得这个地址的值
