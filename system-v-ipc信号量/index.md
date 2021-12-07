# System V IPC信号量

&emsp;&emsp;因我一度混淆IPC信号量与内核同步机制信号量，在此总结IPC信号量，加以区分
> ## 什么是IPC信号量？
&emsp;&emsp;这里注意与**内核同步机制中的信号量**区分  
&emsp;&emsp;二者均为计数器，为多个进程共享的数据结构提供受控访问。但是IPC信号量比内核同步机制中的信号量更为复杂，主要区别有下:  
+ 每个IPC信号量是一个或多个信号量值的集合，而不像内核信号量只有一个值，当然可以请求只有一个信号量的信号量集合，并定义函数模拟内核信号量的简单操作
+ IPC信号量提供一种失效安全机制，用于在进程死亡前自动恢复其控制的信号量
> ## 如何使用IPC信号量
&emsp;&emsp;这里给出一段《深入Linux内核架构》的代码
```C
#include<stdio.h> 
#include<sys/types.h> 
#include<sys/ipc.h> 
#include<sys/sem.h>
#define SEMKEY 1234L /* 标识符 */ 
#define PERMS 0666 /* 访问权限： rwrwrw */ 

struct sembuf op_down[1] = { 0, -1 , 0 }; 
struct sembuf op_up[1] = { 0, 1 , 0 }; 
int semid = -1; /* 信号量 ID */ 
int res; /* 信号量操作的结果 */ 
void init_sem() { 
    /* 测试信号量是否已经存在 */ 
    semid = semget(SEMKEY, 0, IPC_CREAT | PERMS); 
    if (semid < 0) { 
        printf("Create the semaphore\n"); 
        semid = semget(SEMKEY, 1, IPC_CREAT | PERMS); 
        if (semid < 0) { 
            printf("Couldn't create semaphore!\n"); 
            exit(-1);
        }
        /* 初始化为1 */ 
        res = semctl(semid, 0, SETVAL, 1); 
    } 
} 
void down() { 
    /* 执行down操作 */ 
    res = semop(semid, &op_down[0], 1); 
} 
void up() { 
    /* 执行up操作 */ 
    res = semop(semid, &op_up[0], 1); 
} 
int main(){ 
    init_sem(); 
    /* 正常的程序代码 */ 
    printf("Before critical code\n"); 
    down(); 
    /* 临界区代码 */ 
    printf("In critical code\n"); 
    sleep(10); 
    up(); 
    /* 其余代码 */ 
    return 0; 
}
```
> ## 数据结构
&emsp;&emsp;内核使用了几个数据结构来描述所有注册信号量的当前状态，并建立了一种网状结构。它们不仅负责管理信号量及其特征（值、读写权限，等等），还负责通过等待列表将信号量与等待进程关联起来。  
&emsp;&emsp;给定的进程属于task_struct->nsproxy->ipc_ns指向的命名空间，初始的默认命名空间通过ipc_namespace的静态实例init_ipc_ns实现。每个命名空间都包含如下信息:    
```C
<ipc.h> 
struct ipc_namespace { 
    ...
    struct ipc_ids *ids[3]; 
    /* 资源限制 */ 
    ... 
}
```
&emsp;&emsp;我省去了与监视资源消耗和设置资源限制相关的很多结构成员。举例来说，内核会限制共享内存页的最大数目、共享内存段的最大长度、消息队列的最大数目，等等。数组ids中每个数组元素对应于一种IPC机制：共享内存、信号量、消息队列。每个数组项指向一个struct ipc_ids的实例，该结构用于跟踪各类别现存的IPC对象。索引0对应的是信号量，其后是消息队列，最后是共享内存。  
&emsp;&emsp;struct ipc_ids定义如下：
```C
struct ipc_ids { 
    int in_use; 
    unsigned short seq; 
    unsigned short seq_max; 
    struct rw_semaphore rw_mutex; 
    struct idr ipcs_idr; 
};
```
&emsp;&emsp;前几个成员保存了有关IPC对象状态的一般信息:
+ in_use保存了当前使用中IPC对象的数目
+ seq和seq_max用于连续产生用户空间IPC ID。但要注意，ID不等同于序号。内核通过ID来标识IPC对象，ID按资源类型管理，即一个ID用于消息队列，一个用于信号量，一个用于共享内存对象。每次创建新的IPC对象时，序号加1（自动进行回绕，即到达最大值自动变为0）用户层可见的ID由s * SEQ_MULTIPLIER + i给出，其中s是当前序号，i是内核内部的ID。SEQ_MULTIPLIER设置为IPC对象的上限。如果重用了内部ID，仍然会产生不同的用户空间ID，因为序号不会重用。在用户层传递了一个陈旧的ID时，这种做法最小化了使用错误资源的风险
+ _rw_mutex是一个**内核信号量**。它用于实现信号量操作，避免用户空间中的竞态条件。该互斥量有效地保护了包含信号量值的数据结构_      

&emsp;&emsp;每个IPC对象都由kern_ipc_perm的一个实例表示，每个对象都有一个内核内部ID，ipcs_idr用于将ID关联到指向对应的kern_ipc_perm实例的指针。kern_ipc_perm的成员保存了有关信号量“所有者”和访问权限的有关信息:
```C
<ipc.h> 
struct kern_ipc_perm 
{ 
    int id; 
    key_t key; 
    uid_t uid;
    gid_t gid; 
    uid_t cuid; 
    gid_t cgid; 
    mode_t mode; 
    unsigned long seq;
};
```
&emsp;&emsp;该结构不仅可用于信号量，还可以用于其他的IPC机制。其中主要字段说明如下:      
+ key保存了用户程序用来标识信号量的魔数，id是内核内部的ID       
+ uid和gid分别指定了所有者的用户ID和组ID。cuid和cgid保存了产生信号量的进程的用户ID和组ID     
+ mode保存了位掩码，指定了所有者、组、其他用户的访问权限     

&emsp;&emsp;上述数据结构不足以保存信号量所需的所有信息。各进程的task_struct实例中有一个与IPC相关的成员，主要用于撤销信号量。       
&emsp;&emsp;sem_queue是另一个数据结构，用于将信号量与睡眠进程关联起来，该进程想要执行信号量操作，但目前不允许执行。其结构类似双端队列，在此不描述。   
&emsp;&emsp;总体来讲，IPC信号量的内核数据结构层次如下图：  
![IPC](/posts/SystemVIPC信号量/IPC.jpg "ICP信号量结构层次")     
> ## 实现系统调用     
&emsp;&emsp;所有对信号量的操作都使用一个名为ipc的系统调用执行。该调用不仅用于信号量，也用于操作消息队列和共享内存。其第一个参数用于将实际工作委托给其他函数。    
&emsp;&emsp;总之一张图搞定:     
![IPC](/posts/SystemVIPC信号量/IPC2.jpg "ICP信号量系统调用")
