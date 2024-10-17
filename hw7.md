# 作业七
<center>
    蔡合森 2022K8009009004
</center>
**7.1** 设有两个优先级相同的进程 T1，T2 如下。令信号量 S1，S2 的初值为 0，已知 z=2，试

问 T1，T2 并发运行结束后 x=? y=? z=? 

线程 T1 

```
y:=1;
y:=y+2; 
V(S1);
z:=y+1;
P(S2);
y:=z+y; 
```

线程 T2 

```
x:=1; 
x:=x+1; 
p(S1); 
x:=x+y; 
V(S2); 
z:=x+z; 
```

注:请分析所有可能的情况，并给出结果与相应执行顺序。 

-----
**解**：
S1刚开始为0，所以T2的P(S1);操作被阻塞，线程 T1的V(S1);先执行。此时y的值为3，x的值为2，s1的值为1。然后才能执行P(S1);操作。
同理，先执行V(S2);然后是P(S2)操作。其余则分情况讨论。
**情况1：**

- (T1) z=y+1，即z=4
- (T2) z=x+z，即z=9
- (T1) y=z+y，即y=12

最终值：x=5，y=12，z=9

**情况2：**

- (T1) z=y+1，即z=4
- (T1) y=z+y，即y=7
- (T2) z=x+z，即z=9

最终值：x=5，y=7，z=9

**情况3：**

- (T2) z=x+z，z初始2，即z=7
- (T1) z=y+1，即z=4
- (T1) y=z+y，即y=7

最终值：x=5，y=7，z=4
**7.2**在生产者-消费者问题中，假设缓冲区大小为5，生产者和消费者在写入和读取数据时都会更新写入/读取的位置offset。现有以下两种基于信号量的实现方法，
方法一
```C++
Class BoundedBuffer {
    mutex = new Semaphore(1);
    fullBuffers = new Semaphore(0);
    emptyBuffers = new Semaphore(n);
}
BoundedBuffer::Deposit(c) {
    emptyBuffers->P(); 
    mutex->P(); 
Add c to the buffer;
offset++;
    mutex->V();
    fullBuffers->V();
}
BoundedBuffer::Remove(c) {
    fullBuffers->P();
    mutex->P();
Remove c from buffer;
offset--;
    mutex->V();
    emptyBuffers->V();
}
```

方法二：
```C++
Class BoundedBuffer {
    mutex = new Semaphore(1);
    fullBuffers = new Semaphore(0);
    emptyBuffers = new Semaphore(n);
}
BoundedBuffer::Deposit(c) {
    mutex->P();
emptyBuffers->P(); 
Add c to the buffer;
offset++;
fullBuffers->V();
mutex->V();
}
BoundedBuffer::Remove(c) {
    mutex->P();
fullBuffers->P();
Remove c from buffer;
offset--;
emptyBuffers->V();
mutex->V();
}
```
请对比分析上述方法一和方法二，哪种方法能让生产者、消费者进程正常运行，并说明分析原因。

**解**：
方法一能够让生产者消费者进程正常运行。
方法二会导致死锁。原因如下:正常流程是先检查buffer是否依然有空位，缓冲区满了，生产者会停止，等缓冲区有空间时侯，才会对缓冲区上锁；当先执行 mutex->P();操作，此时buffer不允许其他线程访问。如果此时buffer已经满了，生产者需要等待emptyBuffers->P();操作。但此时buffer已经不能被其他线程访问，而生产者在等待消费者消费，来为buffer提供空间，但此时锁还被占据；消费者无法获取锁，会导致死锁。
**7.3** 银行有n个柜员,每个顾客进入银行后先取一个号,并且等着叫号,当一个柜员空闲后,就叫下一个号.
请使用PV操作分别实现:
//顾客取号操作 Customer_Service
//柜员服务操作 Teller_Service
**解**：
仿照上一题代码，操作如下：
```C++
Class ServiceQuene {
    mutex = new Semaphore(1);
    fullQuene = new Semaphore(N);
    emptyQuene = new Semaphore(0);
}
Customer_Service::Deposit(c) {
    emptyQuene->P(); 
    mutex->P(); 
    Add the cunstomer C to the Quene;
    offset++;
    mutex->V();
    fullQuene->V();
}
Teller_Service::Remove(C){
    fullQuene->P();
    mutex->P();
    Remove the cunstomer C from Quene;
    offset--;
    mutex->V();
    emptyQuene->V();

}
```

**7.4**多个线程的规约(Reduce)操作是把每个线程的结果按照某种运算(符合交换律和结合律) 两两合并直到得到最终结果的过程。
试设计管程 monitor 实现一个8线程规约的过程，随机初始化 16 个整数，每个线程通过调用 monitor.getTask 获得2个数，相加后，返回一个数 monitor.putResult ，然后再 getTask( ) 直到全部完成退出，最后打印归约过程和结果。
要求: 为了模拟不均衡性，每个加法操作要加上随机的时间扰动，变动区间1~10ms。 
提示: 使用 pthread_系列的 cond_wait, cond_signal, mutex实现管程；使用 rand( )函数产生随机数，和随机执行时间。
**解**：
```C
#include<stdio.h>
#include<pthread.h>
#include<stdlib.h>

#define MAX_THREAD 8
#define MAX_INTS 16

typedef struct {
    int data[MAX_INTS];
    int size;//store the number of data
    int completed_tasks;//store the number of completed tasks
    int current_tasks;//store the number of current tasks
    pthread_mutex_t mutex;
    pthread_cond_t cond;

}Monitor;

void Monitor_Init(Monitor* monitor){
    for(int i=0;i<MAX_THREAD;i++){
        monitor->data[i] = rand()%100;
    }
    monitor->size = NUM_INTS;
    monitor->completed_tasks = 0;
    monitor->current_tasks = 0;
    pthread_mutex_init(&monitor->mutex,NULL);
    pthread_cond_init(&monitor->cond,NULL);
}

void Monitor_Destory(Monitor* monitor){
    pthread_mutex_destory(&monitor->mutex);
    pthread_mutex_destory(&monitor->cond);
}



int Monitor_gettask(Monitor* monitor,int* a,int* b){
    pthread_mutex_t(&monitor->mutex);

    //判断是否有任务
    if(monitor->completed_tasks>=monitor->size-1){
        pthread_mutex_unlock(&monitor->mutex);
        return -1;
    }

    *a=moniotr->data[monitor->current_tasks];
    *b=monitor->data[monitor->current_tasks+1];

    monitor->current_tasks+=2;

    pthread_mutex_unlock(&monitor->mutex);
    return 0;



}


int monitor_putresult(Monitor* monitor,int a,int b){
    pthread_mutex_lock_t(&monitor->mutex);

    int result = a+b;
    moniotr->data[moniotr->completed_tasks] = result;
    moniotr->completed_tasks++;

    pthread_cond_broadcast(&monitor->cond);
    pthread_mutex_unlock(&monitor->mutex);
}

void* reduce(void* arg){
    Monitor* monitor =(Monitor*)arg;
    int a,b;
    while(Monitor_gettask(monitor,&a,&b)==0){
        usleep(rand()%1000);
        monitor_putresult(monitor,a,b);
        printf(" Thread %lfreduce %d+%d=%d\n",pthread_self(),a,b,a+b);
    }
    return NULL;




}


int main(){
    srand(time(NULL));

    Monitor* monitor;
    Monitor_Init(monitor);
    pthread_t pthread[MAX_THREAD];
    for(int i=0;i<MAX_THREAD;i++){
        pthread_create(&pthread[i],NULL,reduce,(void*)monitor);
    }
    for(int i=0;i<MAX_THREAD;i++){
        pthread_join(pthread[i],NULL);
    }
    printf("The result is %d\n",monitor->data[0]);

    return 0;
}

```
![alt text](image.png)

