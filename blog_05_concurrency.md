# C语言并发编程：原子操作、内存屏障与无锁数据结构

## 引言

并发编程是现代系统开发的核心技术之一，而C语言提供了丰富的底层并发控制机制。本文将深入探讨C语言中的原子操作、内存屏障、线程同步和无锁数据结构等高级并发编程技术。

## 1. 并发编程基础

### 1.1 多线程编程概述

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

// 线程函数
void* thread_function(void* arg) {
    int thread_id = *(int*)arg;
    printf("Thread %d started\n", thread_id);

    // 模拟工作
    for (int i = 0; i < 5; i++) {
        printf("Thread %d: working %d\n", thread_id, i);
        usleep(100000); // 100ms
    }

    printf("Thread %d finished\n", thread_id);
    return NULL;
}

void basic_threading() {
    pthread_t threads[3];
    int thread_ids[3] = {1, 2, 3};

    // 创建线程
    for (int i = 0; i < 3; i++) {
        if (pthread_create(&threads[i], NULL, thread_function, &thread_ids[i]) != 0) {
            perror("Failed to create thread");
            exit(1);
        }
    }

    // 等待线程结束
    for (int i = 0; i < 3; i++) {
        pthread_join(threads[i], NULL);
    }

    printf("All threads finished\n");
}
```

### 1.2 竞态条件与同步问题

```c
#include <stdatomic.h>

// 竞态条件示例
int counter = 0;
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

void* race_condition_thread(void* arg) {
    for (int i = 0; i < 100000; i++) {
        counter++; // 竞态条件
    }
    return NULL;
}

void* safe_counter_thread(void* arg) {
    for (int i = 0; i < 100000; i++) {
        pthread_mutex_lock(&mutex);
        counter++; // 安全的递增
        pthread_mutex_unlock(&mutex);
    }
    return NULL;
}

void demonstrate_race_condition() {
    pthread_t t1, t2;

    // 演示竞态条件
    counter = 0;
    pthread_create(&t1, NULL, race_condition_thread, NULL);
    pthread_create(&t2, NULL, race_condition_thread, NULL);
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    printf("Race condition result: %d (expected: 200000)\n", counter);

    // 演示安全计数
    counter = 0;
    pthread_create(&t1, NULL, safe_counter_thread, NULL);
    pthread_create(&t2, NULL, safe_counter_thread, NULL);
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    printf("Safe counter result: %d (expected: 200000)\n", counter);
}
```

## 2. 原子操作深入解析

### 2.1 C11原子操作

```c
#include <stdatomic.h>

// 原子变量
atomic_int atomic_counter = ATOMIC_VAR_INIT(0);
atomic_llong atomic_long_counter = ATOMIC_VAR_INIT(0);

// 原子操作示例
void demonstrate_atomic_operations() {
    // 原子加载
    int value = atomic_load(&atomic_counter);
    printf("Atomic load: %d\n", value);

    // 原子存储
    atomic_store(&atomic_counter, 42);
    printf("After atomic store: %d\n", atomic_load(&atomic_counter));

    // 原子交换
    int old_value = atomic_exchange(&atomic_counter, 100);
    printf("Exchange: old=%d, new=%d\n", old_value, atomic_load(&atomic_counter));

    // 原子比较交换
    int expected = 100;
    int desired = 200;
    if (atomic_compare_exchange_strong(&atomic_counter, &expected, desired)) {
        printf("CAS succeeded: %d\n", atomic_load(&atomic_counter));
    } else {
        printf("CAS failed: expected=%d, actual=%d\n", expected, atomic_load(&atomic_counter));
    }

    // 原子算术操作
    atomic_fetch_add(&atomic_counter, 10);
    printf("After fetch_add: %d\n", atomic_load(&atomic_counter));

    atomic_fetch_sub(&atomic_counter, 5);
    printf("After fetch_sub: %d\n", atomic_load(&atomic_counter));

    atomic_fetch_and(&atomic_counter, 0xFF);
    printf("After fetch_and: %d\n", atomic_load(&atomic_counter));

    atomic_fetch_or(&atomic_counter, 0x0F);
    printf("After fetch_or: %d\n", atomic_load(&atomic_counter));

    atomic_fetch_xor(&atomic_counter, 0xAA);
    printf("After fetch_xor: %d\n", atomic_load(&atomic_counter));
}
```

### 2.2 内存序与同步语义

```c
// 内存序类型
typedef enum {
    memory_order_relaxed = __ATOMIC_RELAXED,
    memory_order_consume = __ATOMIC_CONSUME,
    memory_order_acquire = __ATOMIC_ACQUIRE,
    memory_order_release = __ATOMIC_RELEASE,
    memory_order_acq_rel = __ATOMIC_ACQ_REL,
    memory_order_seq_cst = __ATOMIC_SEQ_CST
} memory_order;

// 不同内存序的原子操作
void demonstrate_memory_orders() {
    atomic_int var1 = ATOMIC_VAR_INIT(0);
    atomic_int var2 = ATOMIC_VAR_INIT(0);

    // Relaxed顺序：只保证原子性，不保证顺序
    atomic_store_explicit(&var1, 1, memory_order_relaxed);
    int val1 = atomic_load_explicit(&var1, memory_order_relaxed);

    // Release-Aquire顺序：保证同步
    atomic_store_explicit(&var1, 2, memory_order_release);
    int val2 = atomic_load_explicit(&var1, memory_order_acquire);

    // Sequentially consistent顺序：最强保证
    atomic_store_explicit(&var2, 3, memory_order_seq_cst);
    int val3 = atomic_load_explicit(&var2, memory_order_seq_cst);
}

// 线程安全的队列节点
typedef struct node {
    int data;
    struct node* next;
} node_t;

typedef struct {
    atomic_ptr_t head;
    atomic_ptr_t tail;
} lock_free_queue_t;

void queue_push(lock_free_queue_t* queue, int data) {
    node_t* new_node = malloc(sizeof(node_t));
    new_node->data = data;
    new_node->next = NULL;

    node_t* tail = atomic_load_explicit(&queue->tail, memory_order_relaxed);

    // 使用release-acquire语义
    atomic_store_explicit(&tail->next, new_node, memory_order_release);
    atomic_store_explicit(&queue->tail, new_node, memory_order_release);
}
```

### 2.3 平台特定的原子操作

```c
// GCC内置原子操作
void gcc_builtin_atomics() {
    int value = 0;

    // 原子操作
    __atomic_add_fetch(&value, 1, __ATOMIC_SEQ_CST);
    __atomic_sub_fetch(&value, 1, __ATOMIC_SEQ_CST);
    __atomic_and_fetch(&value, 0xFF, __ATOMIC_SEQ_CST);
    __atomic_or_fetch(&value, 0x0F, __ATOMIC_SEQ_CST);
    __atomic_xor_fetch(&value, 0xAA, __ATOMIC_SEQ_CST);

    // 比较交换
    int expected = value;
    int desired = value + 1;
    if (__atomic_compare_exchange_n(&value, &expected, desired, 0,
                                   __ATOMIC_SEQ_CST, __ATOMIC_SEQ_CST)) {
        printf("CAS succeeded\n");
    }

    // 原子加载/存储
    int loaded = __atomic_load_n(&value, __ATOMIC_SEQ_CST);
    __atomic_store_n(&value, 42, __ATOMIC_SEQ_CST);
}

// MSVC原子操作
#ifdef _MSC_VER
#include <intrin.h>

void msvc_intrinsics() {
    long value = 0;

    // 互锁操作
    _InterlockedIncrement(&value);
    _InterlockedDecrement(&value);
    _InterlockedAdd(&value, 10);

    // 比较交换
    long old = _InterlockedCompareExchange(&value, 100, value);

    // 位操作
    _InterlockedAnd(&value, 0xFF);
    _InterlockedOr(&value, 0x0F);
    _InterlockedXor(&value, 0xAA);
}
#endif
```

## 3. 内存屏障与同步

### 3.1 编译器屏障

```c
// 编译器屏障
void compiler_barriers() {
    int a = 1, b = 2, c = 3;

    // 编译器屏障，防止编译器重排
    asm volatile("" ::: "memory");

    // 或者使用内置函数
    __asm__ __volatile__("" ::: "memory");

    // 在关键操作前后使用屏障
    a = 10;
    __asm__ __volatile__("" ::: "memory");
    b = 20;
    __asm__ __volatile__("" ::: "memory");
    c = 30;
}

// 防止编译器优化的屏障
#define OPTIMIZATION_BARRIER() asm volatile("" ::: "memory")

void prevent_optimization() {
    int flag = 0;
    int data = 0;

    // 在多线程环境中，需要防止编译器优化
    flag = 1;
    OPTIMIZATION_BARRIER();
    data = 42;  // 确保data在flag之后赋值

    OPTIMIZATION_BARRIER();
    printf("flag=%d, data=%d\n", flag, data);
}
```

### 3.2 CPU内存屏障

```c
// 全屏障
void full_memory_barriers() {
    int x = 0, y = 0;

    // 全屏障 - 确保之前的所有内存操作都完成
    __sync_synchronize();

    x = 1;
    __sync_synchronize();
    y = 2;
    __sync_synchronize();
}

// 读/写屏障
void rw_barriers() {
    int a = 0, b = 0;

    // 写屏障 - 确保之前的写操作完成
    asm volatile("sfence" ::: "memory");
    a = 10;
    asm volatile("sfence" ::: "memory");

    // 读屏障 - 确保之后的读操作能看到最新的值
    asm volatile("lfence" ::: "memory");
    b = a;
    asm volatile("lfence" ::: "memory");
}

// ARM架构的内存屏障
#ifdef __arm__
void arm_memory_barriers() {
    // 数据内存屏障
    asm volatile("dmb ish" ::: "memory");

    // 数据同步屏障
    asm volatile("dsb ish" ::: "memory");

    // 指令同步屏障
    asm volatile("isb" ::: "memory");
}
#endif
```

### 3.3 内存序与可见性

```c
// 内存可见性示例
typedef struct {
    atomic_int data_ready;
    int data;
    int padding[60]; // 缓存行对齐
} producer_consumer_t;

void* producer_thread(void* arg) {
    producer_consumer_t* pc = (producer_consumer_t*)arg;

    // 生产者线程
    pc->data = 42;
    atomic_store_explicit(&pc->data_ready, 1, memory_order_release);

    return NULL;
}

void* consumer_thread(void* arg) {
    producer_consumer_t* pc = (producer_consumer_t*)arg;

    // 消费者线程
    while (!atomic_load_explicit(&pc->data_ready, memory_order_acquire)) {
        // 等待数据准备就绪
    }

    printf("Consumer received data: %d\n", pc->data);
    return NULL;
}

void demonstrate_visibility() {
    producer_consumer_t pc = {0};
    pthread_t producer, consumer;

    pthread_create(&producer, NULL, producer_thread, &pc);
    pthread_create(&consumer, NULL, consumer_thread, &pc);

    pthread_join(producer, NULL);
    pthread_join(consumer, NULL);
}
```

## 4. 无锁数据结构

### 4.1 无锁栈

```c
typedef struct node {
    int data;
    struct node* next;
} stack_node_t;

typedef struct {
    atomic_ptr_t top;
} lock_free_stack_t;

void stack_init(lock_free_stack_t* stack) {
    atomic_init(&stack->top, NULL);
}

void stack_push(lock_free_stack_t* stack, int data) {
    stack_node_t* new_node = malloc(sizeof(stack_node_t));
    new_node->data = data;

    stack_node_t* old_top;
    do {
        old_top = atomic_load_explicit(&stack->top, memory_order_relaxed);
        new_node->next = old_top;
    } while (!atomic_compare_exchange_weak_explicit(
        &stack->top, &old_top, new_node,
        memory_order_release, memory_order_relaxed));
}

int stack_pop(lock_free_stack_t* stack) {
    stack_node_t* old_top;
    stack_node_t* new_top;

    do {
        old_top = atomic_load_explicit(&stack->top, memory_order_acquire);
        if (!old_top) {
            return -1; // 栈为空
        }
        new_top = old_top->next;
    } while (!atomic_compare_exchange_weak_explicit(
        &stack->top, &old_top, new_top,
        memory_order_release, memory_order_relaxed));

    int data = old_top->data;
    free(old_top);
    return data;
}

void demonstrate_lock_free_stack() {
    lock_free_stack_t stack;
    stack_init(&stack);

    // 推入数据
    for (int i = 0; i < 10; i++) {
        stack_push(&stack, i);
    }

    // 弹出数据
    while (1) {
        int data = stack_pop(&stack);
        if (data == -1) break;
        printf("Popped: %d\n", data);
    }
}
```

### 4.2 无锁队列

```c
typedef struct queue_node {
    int data;
    struct queue_node* next;
} queue_node_t;

typedef struct {
    atomic_ptr_t head;
    atomic_ptr_t tail;
} lock_free_queue_t;

void queue_init(lock_free_queue_t* queue) {
    queue_node_t* dummy = malloc(sizeof(queue_node_t));
    dummy->next = NULL;

    atomic_init(&queue->head, dummy);
    atomic_init(&queue->tail, dummy);
}

void queue_enqueue(lock_free_queue_t* queue, int data) {
    queue_node_t* new_node = malloc(sizeof(queue_node_t));
    new_node->data = data;
    new_node->next = NULL;

    queue_node_t* tail;
    queue_node_t* next;

    while (1) {
        tail = atomic_load_explicit(&queue->tail, memory_order_acquire);
        next = atomic_load_explicit(&tail->next, memory_order_acquire);

        if (tail == atomic_load_explicit(&queue->tail, memory_order_acquire)) {
            if (next == NULL) {
                if (atomic_compare_exchange_weak_explicit(
                    &tail->next, &next, new_node,
                    memory_order_release, memory_order_relaxed)) {
                    break;
                }
            } else {
                atomic_compare_exchange_weak_explicit(
                    &queue->tail, &tail, next,
                    memory_order_release, memory_order_relaxed);
            }
        }
    }

    atomic_compare_exchange_weak_explicit(
        &queue->tail, &tail, new_node,
        memory_order_release, memory_order_relaxed);
}

int queue_dequeue(lock_free_queue_t* queue) {
    queue_node_t* head;
    queue_node_t* tail;
    queue_node_t* next;
    int data;

    while (1) {
        head = atomic_load_explicit(&queue->head, memory_order_acquire);
        tail = atomic_load_explicit(&queue->tail, memory_order_acquire);
        next = atomic_load_explicit(&head->next, memory_order_acquire);

        if (head == atomic_load_explicit(&queue->head, memory_order_acquire)) {
            if (head == tail) {
                if (next == NULL) {
                    return -1; // 队列为空
                }
                atomic_compare_exchange_weak_explicit(
                    &queue->tail, &tail, next,
                    memory_order_release, memory_order_relaxed);
            } else {
                data = next->data;
                if (atomic_compare_exchange_weak_explicit(
                    &queue->head, &head, next,
                    memory_order_release, memory_order_relaxed)) {
                    break;
                }
            }
        }
    }

    free(head);
    return data;
}

void demonstrate_lock_free_queue() {
    lock_free_queue_t queue;
    queue_init(&queue);

    // 入队
    for (int i = 0; i < 10; i++) {
        queue_enqueue(&queue, i);
    }

    // 出队
    while (1) {
        int data = queue_dequeue(&queue);
        if (data == -1) break;
        printf("Dequeued: %d\n", data);
    }
}
```

### 4.3 无锁哈希表

```c
#include <stdint.h>

#define HASH_TABLE_SIZE 1024

typedef struct hash_entry {
    int key;
    int value;
    struct hash_entry* next;
} hash_entry_t;

typedef struct {
    atomic_ptr_t buckets[HASH_TABLE_SIZE];
} lock_free_hash_table_t;

uint32_t hash_function(int key) {
    return (uint32_t)(key * 2654435761UL) % HASH_TABLE_SIZE;
}

void hash_table_init(lock_free_hash_table_t* table) {
    for (int i = 0; i < HASH_TABLE_SIZE; i++) {
        atomic_init(&table->buckets[i], NULL);
    }
}

void hash_table_insert(lock_free_hash_table_t* table, int key, int value) {
    uint32_t hash = hash_function(key);
    hash_entry_t* new_entry = malloc(sizeof(hash_entry_t));
    new_entry->key = key;
    new_entry->value = value;

    hash_entry_t* old_head;
    do {
        old_head = atomic_load_explicit(&table->buckets[hash], memory_order_acquire);
        new_entry->next = old_head;
    } while (!atomic_compare_exchange_weak_explicit(
        &table->buckets[hash], &old_head, new_entry,
        memory_order_release, memory_order_relaxed));
}

int hash_table_lookup(lock_free_hash_table_t* table, int key) {
    uint32_t hash = hash_function(key);
    hash_entry_t* entry = atomic_load_explicit(&table->buckets[hash], memory_order_acquire);

    while (entry) {
        if (entry->key == key) {
            return entry->value;
        }
        entry = entry->next;
    }

    return -1; // 未找到
}

void hash_table_remove(lock_free_hash_table_t* table, int key) {
    uint32_t hash = hash_function(key);
    hash_entry_t* prev = NULL;
    hash_entry_t* current = atomic_load_explicit(&table->buckets[hash], memory_order_acquire);

    while (current) {
        if (current->key == key) {
            if (prev) {
                prev->next = current->next;
            } else {
                atomic_store_explicit(&table->buckets[hash], current->next, memory_order_release);
            }
            free(current);
            return;
        }
        prev = current;
        current = current->next;
    }
}

void demonstrate_lock_free_hash_table() {
    lock_free_hash_table_t table;
    hash_table_init(&table);

    // 插入数据
    for (int i = 0; i < 100; i++) {
        hash_table_insert(&table, i, i * 10);
    }

    // 查找数据
    for (int i = 0; i < 100; i += 10) {
        int value = hash_table_lookup(&table, i);
        printf("Key %d: Value %d\n", i, value);
    }

    // 删除数据
    for (int i = 0; i < 50; i++) {
        hash_table_remove(&table, i);
    }
}
```

## 5. 线程池实现

### 5.1 高性能线程池

```c
#include <stdatomic.h>
#include <pthread.h>
#include <semaphore.h>

typedef struct task {
    void (*function)(void*);
    void* argument;
    struct task* next;
} task_t;

typedef struct {
    atomic_ptr_t task_queue;
    pthread_t* threads;
    int thread_count;
    atomic_bool running;
    sem_t task_semaphore;
    pthread_mutex_t queue_mutex;
} thread_pool_t;

void* worker_thread(void* arg) {
    thread_pool_t* pool = (thread_pool_t*)arg;

    while (atomic_load_explicit(&pool->running, memory_order_acquire)) {
        // 等待任务
        sem_wait(&pool->task_semaphore);

        if (!atomic_load_explicit(&pool->running, memory_order_acquire)) {
            break;
        }

        // 获取任务
        pthread_mutex_lock(&pool->queue_mutex);
        task_t* task = atomic_load_explicit(&pool->task_queue, memory_order_relaxed);
        if (task) {
            atomic_store_explicit(&pool->task_queue, task->next, memory_order_relaxed);
        }
        pthread_mutex_unlock(&pool->queue_mutex);

        if (task) {
            task->function(task->argument);
            free(task);
        }
    }

    return NULL;
}

thread_pool_t* thread_pool_create(int thread_count) {
    thread_pool_t* pool = malloc(sizeof(thread_pool_t));
    if (!pool) return NULL;

    pool->thread_count = thread_count;
    atomic_init(&pool->running, true);
    atomic_init(&pool->task_queue, NULL);

    if (sem_init(&pool->task_semaphore, 0, 0) != 0) {
        free(pool);
        return NULL;
    }

    if (pthread_mutex_init(&pool->queue_mutex, NULL) != 0) {
        sem_destroy(&pool->task_semaphore);
        free(pool);
        return NULL;
    }

    pool->threads = malloc(thread_count * sizeof(pthread_t));
    if (!pool->threads) {
        pthread_mutex_destroy(&pool->queue_mutex);
        sem_destroy(&pool->task_semaphore);
        free(pool);
        return NULL;
    }

    // 创建工作线程
    for (int i = 0; i < thread_count; i++) {
        if (pthread_create(&pool->threads[i], NULL, worker_thread, pool) != 0) {
            // 清理已创建的线程
            for (int j = 0; j < i; j++) {
                pthread_cancel(pool->threads[j]);
            }
            free(pool->threads);
            pthread_mutex_destroy(&pool->queue_mutex);
            sem_destroy(&pool->task_semaphore);
            free(pool);
            return NULL;
        }
    }

    return pool;
}

void thread_pool_submit(thread_pool_t* pool, void (*function)(void*), void* argument) {
    task_t* task = malloc(sizeof(task_t));
    if (!task) return;

    task->function = function;
    task->argument = argument;
    task->next = NULL;

    pthread_mutex_lock(&pool->queue_mutex);
    task_t* current = atomic_load_explicit(&pool->task_queue, memory_order_relaxed);
    if (!current) {
        atomic_store_explicit(&pool->task_queue, task, memory_order_relaxed);
    } else {
        while (current->next) {
            current = current->next;
        }
        current->next = task;
    }
    pthread_mutex_unlock(&pool->queue_mutex);

    sem_post(&pool->task_semaphore);
}

void thread_pool_destroy(thread_pool_t* pool) {
    if (!pool) return;

    // 停止线程
    atomic_store_explicit(&pool->running, false, memory_order_release);

    // 唤醒所有线程
    for (int i = 0; i < pool->thread_count; i++) {
        sem_post(&pool->task_semaphore);
    }

    // 等待线程结束
    for (int i = 0; i < pool->thread_count; i++) {
        pthread_join(pool->threads[i], NULL);
    }

    // 清理资源
    free(pool->threads);
    pthread_mutex_destroy(&pool->queue_mutex);
    sem_destroy(&pool->task_semaphore);

    // 清理剩余任务
    task_t* task = atomic_load_explicit(&pool->task_queue, memory_order_relaxed);
    while (task) {
        task_t* next = task->next;
        free(task);
        task = next;
    }

    free(pool);
}

// 任务函数示例
void sample_task(void* arg) {
    int task_id = *(int*)arg;
    printf("Task %d executed by thread %lu\n", task_id, pthread_self());
    usleep(100000); // 模拟工作
    free(arg);
}

void demonstrate_thread_pool() {
    thread_pool_t* pool = thread_pool_create(4);
    if (!pool) {
        perror("Failed to create thread pool");
        return;
    }

    // 提交任务
    for (int i = 0; i < 20; i++) {
        int* task_id = malloc(sizeof(int));
        *task_id = i;
        thread_pool_submit(pool, sample_task, task_id);
    }

    // 等待一段时间让任务完成
    sleep(3);

    thread_pool_destroy(pool);
    printf("Thread pool destroyed\n");
}
```

## 6. 高级并发模式

### 6.1 读写锁

```c
typedef struct {
    atomic_int readers;
    atomic_int writers;
    pthread_mutex_t mutex;
    pthread_cond_t read_cond;
    pthread_cond_t write_cond;
} rwlock_t;

void rwlock_init(rwlock_t* lock) {
    atomic_init(&lock->readers, 0);
    atomic_init(&lock->writers, 0);
    pthread_mutex_init(&lock->mutex, NULL);
    pthread_cond_init(&lock->read_cond, NULL);
    pthread_cond_init(&lock->write_cond, NULL);
}

void rwlock_read_lock(rwlock_t* lock) {
    pthread_mutex_lock(&lock->mutex);
    while (atomic_load_explicit(&lock->writers, memory_order_acquire) > 0) {
        pthread_cond_wait(&lock->read_cond, &lock->mutex);
    }
    atomic_fetch_add_explicit(&lock->readers, 1, memory_order_release);
    pthread_mutex_unlock(&lock->mutex);
}

void rwlock_read_unlock(rwlock_t* lock) {
    pthread_mutex_lock(&lock->mutex);
    atomic_fetch_sub_explicit(&lock->readers, 1, memory_order_release);
    if (atomic_load_explicit(&lock->readers, memory_order_acquire) == 0) {
        pthread_cond_signal(&lock->write_cond);
    }
    pthread_mutex_unlock(&lock->mutex);
}

void rwlock_write_lock(rwlock_t* lock) {
    pthread_mutex_lock(&lock->mutex);
    while (atomic_load_explicit(&lock->writers, memory_order_acquire) > 0 ||
           atomic_load_explicit(&lock->readers, memory_order_acquire) > 0) {
        pthread_cond_wait(&lock->write_cond, &lock->mutex);
    }
    atomic_fetch_add_explicit(&lock->writers, 1, memory_order_release);
    pthread_mutex_unlock(&lock->mutex);
}

void rwlock_write_unlock(rwlock_t* lock) {
    pthread_mutex_lock(&lock->mutex);
    atomic_fetch_sub_explicit(&lock->writers, 1, memory_order_release);
    pthread_cond_broadcast(&lock->read_cond);
    pthread_cond_signal(&lock->write_cond);
    pthread_mutex_unlock(&lock->mutex);
}

void demonstrate_rwlock() {
    rwlock_t lock;
    rwlock_init(&lock);

    // 读者线程
    for (int i = 0; i < 5; i++) {
        pthread_t reader;
        pthread_create(&reader, NULL, [](void* arg) {
            rwlock_t* lock = (rwlock_t*)arg;
            rwlock_read_lock(lock);
            printf("Reader %lu: acquired read lock\n", pthread_self());
            usleep(500000);
            printf("Reader %lu: releasing read lock\n", pthread_self());
            rwlock_read_unlock(lock);
            return NULL;
        }, &lock);
    }

    // 写者线程
    for (int i = 0; i < 2; i++) {
        pthread_t writer;
        pthread_create(&writer, NULL, [](void* arg) {
            rwlock_t* lock = (rwlock_t*)arg;
            rwlock_write_lock(lock);
            printf("Writer %lu: acquired write lock\n", pthread_self());
            usleep(1000000);
            printf("Writer %lu: releasing write lock\n", pthread_self());
            rwlock_write_unlock(lock);
            return NULL;
        }, &lock);
    }

    sleep(3);
    printf("RWLock demonstration completed\n");
}
```

### 6.2 屏障同步

```c
typedef struct {
    pthread_mutex_t mutex;
    pthread_cond_t cond;
    int count;
    int expected;
} barrier_t;

void barrier_init(barrier_t* barrier, int count) {
    pthread_mutex_init(&barrier->mutex, NULL);
    pthread_cond_init(&barrier->cond, NULL);
    barrier->count = 0;
    barrier->expected = count;
}

void barrier_wait(barrier_t* barrier) {
    pthread_mutex_lock(&barrier->mutex);
    barrier->count++;

    if (barrier->count == barrier->expected) {
        barrier->count = 0;
        pthread_cond_broadcast(&barrier->cond);
    } else {
        pthread_cond_wait(&barrier->cond, &barrier->mutex);
    }

    pthread_mutex_unlock(&barrier->mutex);
}

void demonstrate_barrier() {
    barrier_t barrier;
    barrier_init(&barrier, 3);

    pthread_t threads[3];
    for (int i = 0; i < 3; i++) {
        pthread_create(&threads[i], NULL, [](void* arg) {
            barrier_t* barrier = (barrier_t*)arg;
            printf("Thread %lu: Phase 1\n", pthread_self());
            barrier_wait(barrier);
            printf("Thread %lu: Phase 2\n", pthread_self());
            barrier_wait(barrier);
            printf("Thread %lu: Phase 3\n", pthread_self());
            return NULL;
        }, &barrier);
    }

    for (int i = 0; i < 3; i++) {
        pthread_join(threads[i], NULL);
    }

    printf("Barrier demonstration completed\n");
}
```

### 6.3 生产者-消费者模式

```c
#include <stdatomic.h>
#include <pthread.h>
#include <semaphore.h>

#define BUFFER_SIZE 10

typedef struct {
    int buffer[BUFFER_SIZE];
    atomic_int head;
    atomic_int tail;
    atomic_int count;
    pthread_mutex_t mutex;
    sem_t empty;
    sem_t full;
} bounded_buffer_t;

void buffer_init(bounded_buffer_t* buffer) {
    atomic_init(&buffer->head, 0);
    atomic_init(&buffer->tail, 0);
    atomic_init(&buffer->count, 0);
    pthread_mutex_init(&buffer->mutex, NULL);
    sem_init(&buffer->empty, 0, BUFFER_SIZE);
    sem_init(&buffer->full, 0, 0);
}

void buffer_produce(bounded_buffer_t* buffer, int item) {
    sem_wait(&buffer->empty);
    pthread_mutex_lock(&buffer->mutex);

    buffer->buffer[buffer->tail] = item;
    buffer->tail = (buffer->tail + 1) % BUFFER_SIZE;
    atomic_fetch_add_explicit(&buffer->count, 1, memory_order_release);

    pthread_mutex_unlock(&buffer->mutex);
    sem_post(&buffer->full);
}

int buffer_consume(bounded_buffer_t* buffer) {
    sem_wait(&buffer->full);
    pthread_mutex_lock(&buffer->mutex);

    int item = buffer->buffer[buffer->head];
    buffer->head = (buffer->head + 1) % BUFFER_SIZE;
    atomic_fetch_sub_explicit(&buffer->count, 1, memory_order_release);

    pthread_mutex_unlock(&buffer->mutex);
    sem_post(&buffer->empty);

    return item;
}

void demonstrate_producer_consumer() {
    bounded_buffer_t buffer;
    buffer_init(&buffer);

    pthread_t producer_thread;
    pthread_t consumer_thread;

    // 生产者线程
    pthread_create(&producer_thread, NULL, [](void* arg) {
        bounded_buffer_t* buffer = (bounded_buffer_t*)arg;
        for (int i = 0; i < 20; i++) {
            buffer_produce(buffer, i);
            printf("Produced: %d\n", i);
            usleep(100000);
        }
        return NULL;
    }, &buffer);

    // 消费者线程
    pthread_create(&consumer_thread, NULL, [](void* arg) {
        bounded_buffer_t* buffer = (bounded_buffer_t*)arg;
        for (int i = 0; i < 20; i++) {
            int item = buffer_consume(buffer);
            printf("Consumed: %d\n", item);
            usleep(150000);
        }
        return NULL;
    }, &buffer);

    pthread_join(producer_thread, NULL);
    pthread_join(consumer_thread, NULL);

    printf("Producer-Consumer demonstration completed\n");
}
```

## 7. 性能优化与调试

### 7.1 缓存行优化

```c
#include <stdalign.h>

// 缓存行对齐的数据结构
typedef struct __attribute__((aligned(64))) {
    atomic_long counter1;
    atomic_long counter2;
    atomic_long counter3;
    atomic_long counter4;
    atomic_long counter5;
    atomic_long counter6;
    atomic_long counter7;
    atomic_long counter8;
    char padding[64 - 8 * sizeof(atomic_long)]; // 填充到缓存行大小
} cache_aligned_counters_t;

// 避免false sharing的数组元素
typedef struct {
    int data;
    char padding[60]; // 填充到缓存行大小
} padded_int_t;

void demonstrate_cache_optimization() {
    cache_aligned_counters_t counters;
    atomic_init(&counters.counter1, 0);
    atomic_init(&counters.counter2, 0);

    printf("Cache-aligned counters size: %zu bytes\n", sizeof(cache_aligned_counters_t));
    printf("Padded int size: %zu bytes\n", sizeof(padded_int_t));

    // 验证缓存行对齐
    printf("Counters address: %p (aligned to 64 bytes: %s)\n",
           (void*)&counters,
           ((uintptr_t)&counters & 63) == 0 ? "Yes" : "No");
}
```

### 7.2 并发调试技术

```c
#include <execinfo.h>
#include <signal.h>

// 线程安全的日志系统
typedef struct {
    pthread_mutex_t log_mutex;
    FILE* log_file;
} logger_t;

void logger_init(logger_t* logger, const char* filename) {
    pthread_mutex_init(&logger->log_mutex, NULL);
    logger->log_file = fopen(filename, "a");
}

void logger_log(logger_t* logger, const char* format, ...) {
    pthread_mutex_lock(&logger->log_mutex);

    va_list args;
    va_start(args, format);

    time_t now = time(NULL);
    char timestamp[64];
    strftime(timestamp, sizeof(timestamp), "%Y-%m-%d %H:%M:%S", localtime(&now));

    fprintf(logger->log_file, "[%s] [Thread %lu] ", timestamp, pthread_self());
    vfprintf(logger->log_file, format, args);
    fprintf(logger->log_file, "\n");
    fflush(logger->log_file);

    va_end(args);
    pthread_mutex_unlock(&logger->log_mutex);
}

// 死锁检测
void check_for_deadlocks() {
    // 简单的死锁检测示例
    pthread_mutex_t mutex1 = PTHREAD_MUTEX_INITIALIZER;
    pthread_mutex_t mutex2 = PTHREAD_MUTEX_INITIALIZER;

    // 设置超时锁定
    struct timespec timeout;
    clock_gettime(CLOCK_REALTIME, &timeout);
    timeout.tv_sec += 2; // 2秒超时

    if (pthread_mutex_timedlock(&mutex1, &timeout) != 0) {
        printf("Timeout waiting for mutex1\n");
        return;
    }

    if (pthread_mutex_timedlock(&mutex2, &timeout) != 0) {
        printf("Timeout waiting for mutex2 - potential deadlock\n");
        pthread_mutex_unlock(&mutex1);
        return;
    }

    // 正常操作
    pthread_mutex_unlock(&mutex2);
    pthread_mutex_unlock(&mutex1);
}

// 竞态条件检测
void detect_race_conditions() {
    atomic_int shared_counter = ATOMIC_VAR_INIT(0);
    const int thread_count = 10;
    const int iterations = 10000;

    pthread_t threads[thread_count];

    // 创建多个线程递增计数器
    for (int i = 0; i < thread_count; i++) {
        pthread_create(&threads[i], NULL, [](void* arg) {
            atomic_int* counter = (atomic_int*)arg;
            for (int j = 0; j < iterations; j++) {
                atomic_fetch_add(counter, 1);
            }
            return NULL;
        }, &shared_counter);
    }

    // 等待所有线程完成
    for (int i = 0; i < thread_count; i++) {
        pthread_join(threads[i], NULL);
    }

    int expected = thread_count * iterations;
    int actual = atomic_load(&shared_counter);

    printf("Expected: %d, Actual: %d\n", expected, actual);
    printf("Race condition detected: %s\n", expected != actual ? "Yes" : "No");
}

void demonstrate_concurrent_debugging() {
    logger_t logger;
    logger_init(&logger, "concurrent_debug.log");

    logger_log(&logger, "Starting concurrent debugging demonstration");

    check_for_deadlocks();
    detect_race_conditions();

    logger_log(&logger, "Concurrent debugging demonstration completed");

    fclose(logger.log_file);
    pthread_mutex_destroy(&logger.log_mutex);
}
```

### 7.3 性能分析

```c
#include <time.h>
#include <sys/resource.h>

// 性能计数器
typedef struct {
    struct timespec start_time;
    struct timespec end_time;
    clock_t start_clock;
    clock_t end_clock;
    struct rusage start_usage;
    struct rusage end_usage;
} performance_counter_t;

void performance_counter_start(performance_counter_t* counter) {
    clock_gettime(CLOCK_MONOTONIC, &counter->start_time);
    counter->start_clock = clock();
    getrusage(RUSAGE_SELF, &counter->start_usage);
}

void performance_counter_stop(performance_counter_t* counter) {
    clock_gettime(CLOCK_MONOTONIC, &counter->end_time);
    counter->end_clock = clock();
    getrusage(RUSAGE_SELF, &counter->end_usage);
}

void performance_counter_print(const performance_counter_t* counter, const char* operation) {
    double elapsed_time = (counter->end_time.tv_sec - counter->start_time.tv_sec) +
                        (counter->end_time.tv_nsec - counter->start_time.tv_nsec) / 1e9;

    double cpu_time = (counter->end_clock - counter->start_clock) / (double)CLOCKS_PER_SEC;

    long user_time = counter->end_usage.ru_utime.tv_sec - counter->start_usage.ru_utime.tv_sec;
    long system_time = counter->end_usage.ru_stime.tv_sec - counter->start_usage.ru_stime.tv_sec;

    printf("Performance results for %s:\n", operation);
    printf("  Elapsed time: %.3f seconds\n", elapsed_time);
    printf("  CPU time: %.3f seconds\n", cpu_time);
    printf("  User time: %ld seconds\n", user_time);
    printf("  System time: %ld seconds\n", system_time);
    printf("  CPU utilization: %.1f%%\n", (cpu_time / elapsed_time) * 100);
}

// 并发性能测试
void test_concurrent_performance() {
    performance_counter_t counter;
    const int thread_count = 8;
    const int operations_per_thread = 1000000;

    // 测试原子操作性能
    performance_counter_start(&counter);

    atomic_int atomic_counter = ATOMIC_VAR_INIT(0);
    pthread_t threads[thread_count];

    for (int i = 0; i < thread_count; i++) {
        pthread_create(&threads[i], NULL, [](void* arg) {
            atomic_int* counter = (atomic_int*)arg;
            for (int j = 0; j < operations_per_thread; j++) {
                atomic_fetch_add(counter, 1);
            }
            return NULL;
        }, &atomic_counter);
    }

    for (int i = 0; i < thread_count; i++) {
        pthread_join(threads[i], NULL);
    }

    performance_counter_stop(&counter);
    performance_counter_print(&counter, "atomic operations");

    // 测试互斥锁性能
    performance_counter_start(&counter);

    int mutex_counter = 0;
    pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

    for (int i = 0; i < thread_count; i++) {
        pthread_create(&threads[i], NULL, [](void* arg) {
            pthread_mutex_t* mutex = (pthread_mutex_t*)arg;
            for (int j = 0; j < operations_per_thread; j++) {
                pthread_mutex_lock(mutex);
                // 简单操作
                pthread_mutex_unlock(mutex);
            }
            return NULL;
        }, &mutex);
    }

    for (int i = 0; i < thread_count; i++) {
        pthread_join(threads[i], NULL);
    }

    performance_counter_stop(&counter);
    performance_counter_print(&counter, "mutex operations");

    pthread_mutex_destroy(&mutex);
}

void demonstrate_performance_analysis() {
    test_concurrent_performance();
}
```

## 8. 最佳实践与注意事项

### 8.1 并发编程原则

1. **最小化共享状态**：尽量减少线程间的共享数据
2. **避免死锁**：按固定顺序获取锁
3. **正确使用内存序**：选择合适的内存序保证正确性
4. **处理错误情况**：检查所有系统调用的返回值
5. **资源清理**：确保线程退出时正确释放资源

### 8.2 常见陷阱与解决方案

**陷阱1：ABA问题**
```c
// 解决方案：使用版本号或指针标记
typedef struct {
    void* ptr;
    uint32_t version;
} versioned_ptr_t;

atomic_llong global_ptr = ATOMIC_VAR_INIT(0);

// 版本化的比较交换
bool versioned_cas(atomic_llong* target, versioned_ptr_t expected, versioned_ptr_t desired) {
    uint64_t expected_value = ((uint64_t)expected.version << 32) | (uint32_t)(uintptr_t)expected.ptr;
    uint64_t desired_value = ((uint64_t)desired.version << 32) | (uint32_t)(uintptr_t)desired.ptr;

    return atomic_compare_exchange_strong(target, &expected_value, desired_value);
}
```

**陷阱2：内存泄漏**
```c
// 确保所有分配的内存都被释放
void cleanup_thread_resources() {
    // 在线程退出前清理资源
    pthread_cleanup_push(free, allocated_memory);

    // 线程工作
    do_work();

    pthread_cleanup_pop(1); // 执行清理
}
```

**陷阱3：优先级反转**
```c
// 解决方案：使用优先级继承或优先级上限
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_INHERIT);
pthread_mutex_init(&mutex, &attr);
```

### 8.3 调试工具

```c
// 使用Valgrind检测线程错误
// 编译时加入调试信息
// gcc -g -pthread -o program program.c

// 运行Helgrind检测数据竞争
// valgrind --tool=helgrind ./program

// 运行DRD检测错误
// valgrind --tool=drd ./program

// 使用gdb调试多线程程序
// (gdb) info threads
// (gdb) thread <thread-id>
// (gdb) bt
```

## 9. 总结

### 9.1 核心概念回顾

- **原子操作**：不可分割的内存操作
- **内存屏障**：控制内存操作的顺序
- **无锁数据结构**：不使用锁的并发数据结构
- **线程同步**：协调多个线程的执行
- **性能优化**：提升并发程序的性能

### 9.2 关键技术点

1. **C11原子操作**：标准化的原子操作接口
2. **内存序**：控制内存操作的可见性和顺序
3. **无锁算法**：避免锁的高性能并发算法
4. **线程池**：高效管理和复用线程
5. **调试技术**：识别和解决并发问题

### 9.3 实际应用

- **高性能服务器**：处理大量并发请求
- **实时系统**：确保响应时间的可预测性
- **数据库系统**：管理并发访问
- **图形渲染**：并行处理渲染任务
- **科学计算**：分布式计算和数据分析

通过掌握C语言并发编程的高级技术，您可以开发出高性能、高可靠性的多线程应用程序。记住，并发编程虽然强大，但也需要谨慎处理，确保程序的正确性和稳定性。