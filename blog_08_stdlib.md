# C语言标准库深度剖析：IO缓冲区、字符串处理与算法实现

## 引言

C语言标准库是每个C程序员的必备工具集，但其内部实现机制往往被忽视。本文将深入剖析标准库的核心组件，包括IO缓冲区管理、字符串处理算法、内存分配机制等，揭示其设计原理和性能优化技巧。

## 1. IO缓冲区机制深度解析

### 1.1 FILE结构体内部实现

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>

// 简化的FILE结构体实现
typedef struct {
    int fd;                     // 文件描述符
    unsigned char *buffer;      // 缓冲区指针
    size_t buffer_size;         // 缓冲区大小
    size_t buffer_pos;          // 当前缓冲区位置
    size_t buffer_used;         // 已使用的缓冲区大小
    int flags;                  // 文件状态标志
    int error;                  // 错误码
    int eof;                    // EOF标志
    mode_t mode;                // 文件打开模式
} my_FILE_t;

// 缓冲区标志位
#define MY_IONBF 0x01           // 无缓冲
#define MY_IOLBF 0x02           // 行缓冲
#define MY_IOFBF 0x04           // 全缓冲
#define MY_IOEOF 0x08           // EOF标志
#define MY_IOERR 0x10           // 错误标志

// 创建自定义FILE结构
my_FILE_t* my_fopen(const char* filename, const char* mode) {
    int flags = 0;
    int fd = -1;
    mode_t open_mode = 0644;

    // 解析打开模式
    switch (mode[0]) {
        case 'r':
            flags = O_RDONLY;
            break;
        case 'w':
            flags = O_WRONLY | O_CREAT | O_TRUNC;
            break;
        case 'a':
            flags = O_WRONLY | O_CREAT | O_APPEND;
            break;
        default:
            return NULL;
    }

    // 处理+模式
    if (mode[1] == '+') {
        flags = O_RDWR;
        if (mode[0] == 'r') {
            flags |= O_CREAT;
        }
    }

    fd = open(filename, flags, open_mode);
    if (fd == -1) {
        return NULL;
    }

    // 分配FILE结构
    my_FILE_t* file = malloc(sizeof(my_FILE_t));
    if (!file) {
        close(fd);
        return NULL;
    }

    // 初始化缓冲区
    file->buffer_size = BUFSIZ;
    file->buffer = malloc(file->buffer_size);
    if (!file->buffer) {
        free(file);
        close(fd);
        return NULL;
    }

    // 初始化FILE结构
    file->fd = fd;
    file->buffer_pos = 0;
    file->buffer_used = 0;
    file->flags = MY_IOFBF;  // 默认全缓冲
    file->error = 0;
    file->eof = 0;
    file->mode = flags;

    return file;
}
```

### 1.2 缓冲区管理策略

```c
// 刷新缓冲区
int my_fflush(my_FILE_t* file) {
    if (!file) return -1;

    // 如果缓冲区有数据，写入文件
    if (file->buffer_used > 0) {
        ssize_t written = write(file->fd, file->buffer, file->buffer_used);
        if (written == -1) {
            file->error = 1;
            return -1;
        }

        // 部分写入处理
        if (written < (ssize_t)file->buffer_used) {
            memmove(file->buffer, file->buffer + written, file->buffer_used - written);
            file->buffer_used -= written;
            return -1;
        }

        file->buffer_used = 0;
        file->buffer_pos = 0;
    }

    return 0;
}

// 带缓冲区的读取
size_t my_fread(void* ptr, size_t size, size_t count, my_FILE_t* file) {
    if (!file || !ptr || size == 0 || count == 0) {
        return 0;
    }

    size_t total_bytes = size * count;
    size_t bytes_read = 0;
    unsigned char* buffer = (unsigned char*)ptr;

    // 首先从缓冲区读取
    while (bytes_read < total_bytes) {
        // 缓冲区为空，需要从文件读取
        if (file->buffer_pos >= file->buffer_used) {
            if (file->eof) {
                break;
            }

            ssize_t n = read(file->fd, file->buffer, file->buffer_size);
            if (n == -1) {
                file->error = 1;
                break;
            } else if (n == 0) {
                file->eof = 1;
                break;
            }

            file->buffer_used = n;
            file->buffer_pos = 0;
        }

        // 从缓冲区复制数据
        size_t available = file->buffer_used - file->buffer_pos;
        size_t needed = total_bytes - bytes_read;
        size_t copy = (available < needed) ? available : needed;

        memcpy(buffer + bytes_read, file->buffer + file->buffer_pos, copy);
        file->buffer_pos += copy;
        bytes_read += copy;
    }

    return bytes_read / size;
}

// 带缓冲区的写入
size_t my_fwrite(const void* ptr, size_t size, size_t count, my_FILE_t* file) {
    if (!file || !ptr || size == 0 || count == 0) {
        return 0;
    }

    size_t total_bytes = size * count;
    size_t bytes_written = 0;
    const unsigned char* data = (const unsigned char*)ptr;

    while (bytes_written < total_bytes) {
        // 缓冲区已满，需要刷新
        if (file->buffer_used >= file->buffer_size) {
            if (my_fflush(file) != 0) {
                break;
            }
        }

        // 计算可写入的字节数
        size_t available = file->buffer_size - file->buffer_used;
        size_t needed = total_bytes - bytes_written;
        size_t copy = (available < needed) ? available : needed;

        memcpy(file->buffer + file->buffer_used, data + bytes_written, copy);
        file->buffer_used += copy;
        bytes_written += copy;

        // 行缓冲模式：遇到换行符刷新
        if ((file->flags & MY_IOLBF) &&
            file->buffer[file->buffer_used - 1] == '\n') {
            my_fflush(file);
        }
    }

    return bytes_written / size;
}

// 设置缓冲区模式
void my_setvbuf(my_FILE_t* file, char* buffer, int mode, size_t size) {
    if (!file) return;

    // 释放原有缓冲区
    if (file->buffer && file->buffer != (unsigned char*)buffer) {
        free(file->buffer);
    }

    file->flags &= ~(MY_IONBF | MY_IOLBF | MY_IOFBF);

    switch (mode) {
        case _IONBF:
            file->flags |= MY_IONBF;
            file->buffer_size = 0;
            file->buffer = NULL;
            break;
        case _IOLBF:
            file->flags |= MY_IOLBF;
            if (!buffer) {
                file->buffer_size = BUFSIZ;
                file->buffer = malloc(file->buffer_size);
            } else {
                file->buffer_size = size;
                file->buffer = (unsigned char*)buffer;
            }
            break;
        case _IOFBF:
            file->flags |= MY_IOFBF;
            if (!buffer) {
                file->buffer_size = BUFSIZ;
                file->buffer = malloc(file->buffer_size);
            } else {
                file->buffer_size = size;
                file->buffer = (unsigned char*)buffer;
            }
            break;
    }

    file->buffer_pos = 0;
    file->buffer_used = 0;
}
```

### 1.3 高级IO操作

```c
// 格式化输出简化实现
int my_fprintf(my_FILE_t* file, const char* format, ...) {
    if (!file || !format) return -1;

    va_list args;
    va_start(args, format);

    char buffer[1024];
    int written = vsnprintf(buffer, sizeof(buffer), format, args);
    va_end(args);

    if (written < 0) {
        file->error = 1;
        return -1;
    }

    if (written >= sizeof(buffer)) {
        // 缓冲区溢出，这里简化处理
        written = sizeof(buffer) - 1;
    }

    size_t result = my_fwrite(buffer, 1, written, file);
    return (result == (size_t)written) ? written : -1;
}

// 高效的文件复制函数
int my_copy_file(const char* src, const char* dst) {
    my_FILE_t* src_file = my_fopen(src, "rb");
    if (!src_file) {
        perror("Failed to open source file");
        return -1;
    }

    my_FILE_t* dst_file = my_fopen(dst, "wb");
    if (!dst_file) {
        perror("Failed to open destination file");
        my_fclose(src_file);
        return -1;
    }

    // 使用较大的缓冲区提高性能
    my_setvbuf(src_file, NULL, _IOFBF, 65536);
    my_setvbuf(dst_file, NULL, _IOFBF, 65536);

    char buffer[65536];
    size_t bytes_read;
    int result = 0;

    while ((bytes_read = my_fread(buffer, 1, sizeof(buffer), src_file)) > 0) {
        size_t bytes_written = my_fwrite(buffer, 1, bytes_read, dst_file);
        if (bytes_written != bytes_read) {
            perror("Failed to write to destination");
            result = -1;
            break;
        }
    }

    my_fclose(src_file);
    my_fclose(dst_file);

    return result;
}

// 内存映射文件IO
typedef struct {
    int fd;
    void* addr;
    size_t size;
} mapped_file_t;

mapped_file_t* map_file(const char* filename, int writable) {
    int fd = open(filename, writable ? O_RDWR : O_RDONLY);
    if (fd == -1) return NULL;

    struct stat st;
    if (fstat(fd, &st) == -1) {
        close(fd);
        return NULL;
    }

    int prot = writable ? (PROT_READ | PROT_WRITE) : PROT_READ;
    int flags = writable ? MAP_SHARED : MAP_PRIVATE;

    void* addr = mmap(NULL, st.st_size, prot, flags, fd, 0);
    if (addr == MAP_FAILED) {
        close(fd);
        return NULL;
    }

    mapped_file_t* mf = malloc(sizeof(mapped_file_t));
    if (!mf) {
        munmap(addr, st.st_size);
        close(fd);
        return NULL;
    }

    mf->fd = fd;
    mf->addr = addr;
    mf->size = st.st_size;

    return mf;
}

void unmap_file(mapped_file_t* mf) {
    if (mf) {
        munmap(mf->addr, mf->size);
        close(mf->fd);
        free(mf);
    }
}
```

## 2. 字符串处理算法深度剖析

### 2.1 字符串长度和复制

```c
#include <stdint.h>

// 优化的strlen实现
size_t my_strlen(const char* str) {
    if (!str) return 0;

    const char* ptr = str;

    // 检查对齐
    while ((uintptr_t)ptr & (sizeof(uintptr_t) - 1)) {
        if (*ptr == '\0') return ptr - str;
        ptr++;
    }

    // 字对齐的快速检查
    const uintptr_t* word_ptr = (const uintptr_t*)ptr;
    uintptr_t word;

    // 使用魔术字检测零字节
    const uintptr_t magic = (uintptr_t)-1 / 255 * 0x80;

    while (1) {
        word = *word_ptr;
        if ((word - magic) & ~word & magic) {
            break;
        }
        word_ptr++;
    }

    // 检查具体哪个字节是零
    ptr = (const char*)word_ptr;
    if (ptr[0] == '\0') return ptr - str;
    if (ptr[1] == '\0') return ptr - str + 1;
    if (ptr[2] == '\0') return ptr - str + 2;
    if (ptr[3] == '\0') return ptr - str + 3;

    return ptr - str + 4;
}

// 优化的strcpy实现
char* my_strcpy(char* dest, const char* src) {
    char* result = dest;

    // 检查对齐
    while ((uintptr_t)dest & (sizeof(uintptr_t) - 1)) {
        if ((*dest++ = *src++) == '\0') {
            return result;
        }
    }

    // 字对齐的快速复制
    uintptr_t* dest_word = (uintptr_t*)dest;
    const uintptr_t* src_word = (const uintptr_t*)src;

    // 使用魔术字检测零字节
    const uintptr_t magic = (uintptr_t)-1 / 255 * 0x80;
    uintptr_t word;

    while (1) {
        word = *src_word++;
        if ((word - magic) & ~word & magic) {
            break;
        }
        *dest_word++ = word;
    }

    // 复制剩余字节
    dest = (char*)dest_word;
    src = (const char*)src_word - sizeof(uintptr_t);

    while ((*dest++ = *src++) != '\0') {
        // 继续复制直到遇到'\0'
    }

    return result;
}

// 优化的memcpy实现
void* my_memcpy(void* dest, const void* src, size_t n) {
    if (!dest || !src || n == 0) return dest;

    // 检查重叠
    if ((const char*)src < (char*)dest && (const char*)src + n > (char*)dest) {
        // 向后复制
        char* d = (char*)dest + n - 1;
        const char* s = (const char*)src + n - 1;

        // 字节级别的向后复制
        while (n--) {
            *d-- = *s--;
        }

        return dest;
    }

    // 检查对齐
    if ((uintptr_t)dest % sizeof(uintptr_t) == 0 &&
        (uintptr_t)src % sizeof(uintptr_t) == 0) {
        // 字对齐的快速复制
        uintptr_t* d = (uintptr_t*)dest;
        const uintptr_t* s = (const uintptr_t*)src;

        size_t words = n / sizeof(uintptr_t);
        size_t bytes = n % sizeof(uintptr_t);

        while (words--) {
            *d++ = *s++;
        }

        // 复制剩余字节
        char* cd = (char*)d;
        const char* cs = (const char*)s;

        while (bytes--) {
            *cd++ = *cs++;
        }
    } else {
        // 字节级别的复制
        char* d = (char*)dest;
        const char* s = (const char*)src;

        while (n--) {
            *d++ = *s++;
        }
    }

    return dest;
}
```

### 2.2 字符串查找和比较

```c
// 优化的strcmp实现
int my_strcmp(const char* s1, const char* s2) {
    // 检查对齐
    while ((uintptr_t)s1 & (sizeof(uintptr_t) - 1)) {
        int diff = (unsigned char)*s1 - (unsigned char)*s2;
        if (diff != 0 || *s1 == '\0') {
            return diff;
        }
        s1++;
        s2++;
    }

    // 字对齐的比较
    const uintptr_t* w1 = (const uintptr_t*)s1;
    const uintptr_t* w2 = (const uintptr_t*)s2;

    const uintptr_t magic = (uintptr_t)-1 / 255 * 0x80;
    uintptr_t word1, word2;

    while (1) {
        word1 = *w1++;
        word2 = *w2++;

        if (word1 != word2) {
            // 找到第一个不同的字节
            const char* c1 = (const char*)&word1;
            const char* c2 = (const char*)&word2;

            for (int i = 0; i < (int)sizeof(uintptr_t); i++) {
                if (c1[i] != c2[i]) {
                    return (unsigned char)c1[i] - (unsigned char)c2[i];
                }
            }
        }

        if ((word1 - magic) & ~word1 & magic) {
            break;  // 遇到'\0'
        }
    }

    return 0;
}

// 优化的strstr实现 - Boyer-Moore算法
const char* my_strstr(const char* haystack, const char* needle) {
    if (!haystack || !needle) return NULL;
    if (*needle == '\0') return haystack;

    size_t needle_len = my_strlen(needle);
    size_t haystack_len = my_strlen(haystack);

    if (needle_len > haystack_len) return NULL;

    // 构建坏字符表
    int bad_char[256];
    for (int i = 0; i < 256; i++) {
        bad_char[i] = needle_len;
    }

    for (size_t i = 0; i < needle_len - 1; i++) {
        bad_char[(unsigned char)needle[i]] = needle_len - 1 - i;
    }

    // 搜索
    size_t i = 0;
    while (i <= haystack_len - needle_len) {
        int j = needle_len - 1;

        while (j >= 0 && haystack[i + j] == needle[j]) {
            j--;
        }

        if (j < 0) {
            return haystack + i;
        } else {
            i += bad_char[(unsigned char)haystack[i + j]];
        }
    }

    return NULL;
}

// KMP算法实现
typedef struct {
    int* lps;
    int len;
} kmp_state_t;

kmp_state_t* kmp_compile(const char* pattern) {
    if (!pattern) return NULL;

    int len = my_strlen(pattern);
    kmp_state_t* state = malloc(sizeof(kmp_state_t));
    if (!state) return NULL;

    state->lps = malloc(len * sizeof(int));
    if (!state->lps) {
        free(state);
        return NULL;
    }

    state->len = len;

    // 计算LPS数组
    int length = 0;
    state->lps[0] = 0;
    int i = 1;

    while (i < len) {
        if (pattern[i] == pattern[length]) {
            length++;
            state->lps[i] = length;
            i++;
        } else {
            if (length != 0) {
                length = state->lps[length - 1];
            } else {
                state->lps[i] = 0;
                i++;
            }
        }
    }

    return state;
}

const char* kmp_search(const char* text, kmp_state_t* state) {
    if (!text || !state) return NULL;

    const char* pattern = text;  // 这里简化处理，实际需要传入pattern
    int pattern_len = state->len;
    int text_len = my_strlen(text);

    if (pattern_len == 0) return text;
    if (pattern_len > text_len) return NULL;

    int i = 0;  // text的索引
    int j = 0;  // pattern的索引

    while (i < text_len) {
        if (pattern[j] == text[i]) {
            i++;
            j++;

            if (j == pattern_len) {
                return text + i - j;
            }
        } else {
            if (j != 0) {
                j = state->lps[j - 1];
            } else {
                i++;
            }
        }
    }

    return NULL;
}

void kmp_free(kmp_state_t* state) {
    if (state) {
        if (state->lps) {
            free(state->lps);
        }
        free(state);
    }
}
```

### 2.3 高级字符串操作

```c
// 字符串分割器
typedef struct {
    char* str;
    char* current;
    char* next_token;
    const char* delimiters;
} string_tokenizer_t;

string_tokenizer_t* create_tokenizer(char* str, const char* delimiters) {
    string_tokenizer_t* tokenizer = malloc(sizeof(string_tokenizer_t));
    if (!tokenizer) return NULL;

    tokenizer->str = str;
    tokenizer->current = str;
    tokenizer->next_token = NULL;
    tokenizer->delimiters = delimiters;

    return tokenizer;
}

char* next_token(string_tokenizer_t* tokenizer) {
    if (!tokenizer || !tokenizer->current) {
        return NULL;
    }

    // 跳过前导分隔符
    while (*tokenizer->current && my_strchr(tokenizer->delimiters, *tokenizer->current)) {
        tokenizer->current++;
    }

    if (*tokenizer->current == '\0') {
        return NULL;
    }

    // 找到token的结束位置
    char* token_start = tokenizer->current;
    while (*tokenizer->current && !my_strchr(tokenizer->delimiters, *tokenizer->current)) {
        tokenizer->current++;
    }

    // 标记token结束
    if (*tokenizer->current) {
        *tokenizer->current = '\0';
        tokenizer->current++;
    }

    return token_start;
}

void free_tokenizer(string_tokenizer_t* tokenizer) {
    free(tokenizer);
}

// 字符串构建器
typedef struct {
    char* buffer;
    size_t capacity;
    size_t length;
} string_builder_t;

string_builder_t* create_string_builder(size_t initial_capacity) {
    string_builder_t* builder = malloc(sizeof(string_builder_t));
    if (!builder) return NULL;

    builder->capacity = initial_capacity;
    builder->length = 0;
    builder->buffer = malloc(initial_capacity);

    if (!builder->buffer) {
        free(builder);
        return NULL;
    }

    builder->buffer[0] = '\0';
    return builder;
}

int builder_append(string_builder_t* builder, const char* str) {
    if (!builder || !str) return -1;

    size_t str_len = my_strlen(str);
    size_t new_length = builder->length + str_len;

    // 检查是否需要扩展缓冲区
    if (new_length >= builder->capacity) {
        size_t new_capacity = builder->capacity * 2;
        while (new_capacity <= new_length) {
            new_capacity *= 2;
        }

        char* new_buffer = realloc(builder->buffer, new_capacity);
        if (!new_buffer) return -1;

        builder->buffer = new_buffer;
        builder->capacity = new_capacity;
    }

    // 追加字符串
    my_memcpy(builder->buffer + builder->length, str, str_len);
    builder->length = new_length;
    builder->buffer[builder->length] = '\0';

    return 0;
}

char* builder_to_string(string_builder_t* builder) {
    if (!builder) return NULL;

    char* result = malloc(builder->length + 1);
    if (!result) return NULL;

    my_memcpy(result, builder->buffer, builder->length + 1);
    return result;
}

void free_string_builder(string_builder_t* builder) {
    if (builder) {
        if (builder->buffer) {
            free(builder->buffer);
        }
        free(builder);
    }
}

// 正则表达式简化实现
typedef struct {
    const char* pattern;
    int pattern_len;
} regex_t;

int regex_match(const char* pattern, const char* text) {
    // 简化的正则表达式匹配，只支持*和?
    const char* p = pattern;
    const char* t = text;
    const char* last_star = NULL;
    const char* last_text = NULL;

    while (*t) {
        if (*p == '*') {
            last_star = p++;
            last_text = t;
        } else if (*p == '?' || *p == *t) {
            p++;
            t++;
        } else if (last_star) {
            p = last_star + 1;
            t = ++last_text;
        } else {
            return 0;
        }
    }

    while (*p == '*') {
        p++;
    }

    return *p == '\0';
}
```

## 3. 内存管理算法

### 3.1 内存池实现

```c
#include <pthread.h>

// 固定大小内存池
typedef struct memory_pool {
    void* pool;
    size_t block_size;
    size_t total_blocks;
    size_t free_blocks;
    void** free_list;
    pthread_mutex_t lock;
} memory_pool_t;

memory_pool_t* create_memory_pool(size_t block_size, size_t total_blocks) {
    memory_pool_t* pool = malloc(sizeof(memory_pool_t));
    if (!pool) return NULL;

    pool->block_size = block_size;
    pool->total_blocks = total_blocks;
    pool->free_blocks = total_blocks;

    // 分配内存池
    pool->pool = malloc(block_size * total_blocks);
    if (!pool->pool) {
        free(pool);
        return NULL;
    }

    // 分配空闲链表
    pool->free_list = malloc(total_blocks * sizeof(void*));
    if (!pool->free_list) {
        free(pool->pool);
        free(pool);
        return NULL;
    }

    // 初始化空闲链表
    for (size_t i = 0; i < total_blocks; i++) {
        pool->free_list[i] = (char*)pool->pool + i * block_size;
    }

    // 初始化锁
    if (pthread_mutex_init(&pool->lock, NULL) != 0) {
        free(pool->free_list);
        free(pool->pool);
        free(pool);
        return NULL;
    }

    return pool;
}

void* pool_alloc(memory_pool_t* pool) {
    if (!pool || pool->free_blocks == 0) {
        return NULL;
    }

    pthread_mutex_lock(&pool->lock);

    if (pool->free_blocks == 0) {
        pthread_mutex_unlock(&pool->lock);
        return NULL;
    }

    void* block = pool->free_list[--pool->free_blocks];
    pthread_mutex_unlock(&pool->lock);

    return block;
}

void pool_free(memory_pool_t* pool, void* block) {
    if (!pool || !block) return;

    pthread_mutex_lock(&pool->lock);

    if (pool->free_blocks < pool->total_blocks) {
        pool->free_list[pool->free_blocks++] = block;
    }

    pthread_mutex_unlock(&pool->lock);
}

void destroy_memory_pool(memory_pool_t* pool) {
    if (pool) {
        pthread_mutex_destroy(&pool->lock);
        if (pool->free_list) free(pool->free_list);
        if (pool->pool) free(pool->pool);
        free(pool);
    }
}

// 多级内存分配器
typedef struct multi_pool_allocator {
    memory_pool_t* pools[4];  // 不同大小的池
    pthread_mutex_t lock;
} multi_pool_allocator_t;

multi_pool_allocator_t* create_multi_pool_allocator() {
    multi_pool_allocator_t* allocator = malloc(sizeof(multi_pool_allocator_t));
    if (!allocator) return NULL;

    // 创建不同大小的内存池
    allocator->pools[0] = create_memory_pool(16, 1024);    // 小对象
    allocator->pools[1] = create_memory_pool(64, 512);     // 中等对象
    allocator->pools[2] = create_memory_pool(256, 256);    // 大对象
    allocator->pools[3] = create_memory_pool(1024, 64);    // 超大对象

    for (int i = 0; i < 4; i++) {
        if (!allocator->pools[i]) {
            // 清理已创建的池
            for (int j = 0; j < i; j++) {
                destroy_memory_pool(allocator->pools[j]);
            }
            free(allocator);
            return NULL;
        }
    }

    if (pthread_mutex_init(&allocator->lock, NULL) != 0) {
        for (int i = 0; i < 4; i++) {
            destroy_memory_pool(allocator->pools[i]);
        }
        free(allocator);
        return NULL;
    }

    return allocator;
}

void* multi_pool_alloc(multi_pool_allocator_t* allocator, size_t size) {
    if (!allocator) return NULL;

    int pool_index = -1;

    // 选择合适的内存池
    if (size <= 16) pool_index = 0;
    else if (size <= 64) pool_index = 1;
    else if (size <= 256) pool_index = 2;
    else if (size <= 1024) pool_index = 3;

    if (pool_index == -1) {
        // 超过最大大小，使用普通malloc
        return malloc(size);
    }

    return pool_alloc(allocator->pools[pool_index]);
}

void multi_pool_free(multi_pool_allocator_t* allocator, void* ptr, size_t size) {
    if (!allocator || !ptr) return;

    int pool_index = -1;

    // 确定内存池
    if (size <= 16) pool_index = 0;
    else if (size <= 64) pool_index = 1;
    else if (size <= 256) pool_index = 2;
    else if (size <= 1024) pool_index = 3;

    if (pool_index == -1) {
        // 使用普通free
        free(ptr);
    } else {
        pool_free(allocator->pools[pool_index], ptr);
    }
}

void destroy_multi_pool_allocator(multi_pool_allocator_t* allocator) {
    if (allocator) {
        pthread_mutex_destroy(&allocator->lock);
        for (int i = 0; i < 4; i++) {
            destroy_memory_pool(allocator->pools[i]);
        }
        free(allocator);
    }
}
```

### 3.2 垃圾回收机制

```c
// 简单的标记-清除垃圾回收器
typedef struct gc_object {
    void* data;
    size_t size;
    int marked;
    void (*destructor)(void*);
    struct gc_object* next;
} gc_object_t;

typedef struct {
    gc_object_t* objects;
    gc_object_t** roots;
    size_t root_count;
    size_t root_capacity;
    pthread_mutex_t lock;
} garbage_collector_t;

garbage_collector_t* create_garbage_collector() {
    garbage_collector_t* gc = malloc(sizeof(garbage_collector_t));
    if (!gc) return NULL;

    gc->objects = NULL;
    gc->root_count = 0;
    gc->root_capacity = 16;
    gc->roots = malloc(gc->root_capacity * sizeof(gc_object_t*));

    if (!gc->roots) {
        free(gc);
        return NULL;
    }

    if (pthread_mutex_init(&gc->lock, NULL) != 0) {
        free(gc->roots);
        free(gc);
        return NULL;
    }

    return gc;
}

void* gc_alloc(garbage_collector_t* gc, size_t size, void (*destructor)(void*)) {
    if (!gc) return NULL;

    pthread_mutex_lock(&gc->lock);

    // 分配内存
    void* data = malloc(size);
    if (!data) {
        pthread_mutex_unlock(&gc->lock);
        return NULL;
    }

    // 创建GC对象
    gc_object_t* obj = malloc(sizeof(gc_object_t));
    if (!obj) {
        free(data);
        pthread_mutex_unlock(&gc->lock);
        return NULL;
    }

    obj->data = data;
    obj->size = size;
    obj->marked = 0;
    obj->destructor = destructor;
    obj->next = gc->objects;
    gc->objects = obj;

    pthread_mutex_unlock(&gc->lock);
    return data;
}

void gc_add_root(garbage_collector_t* gc, void* ptr) {
    if (!gc || !ptr) return;

    pthread_mutex_lock(&gc->lock);

    // 扩展根数组
    if (gc->root_count >= gc->root_capacity) {
        gc->root_capacity *= 2;
        gc_object_t** new_roots = realloc(gc->roots, gc->root_capacity * sizeof(gc_object_t*));
        if (!new_roots) {
            pthread_mutex_unlock(&gc->lock);
            return;
        }
        gc->roots = new_roots;
    }

    // 查找对应的GC对象
    gc_object_t* obj = gc->objects;
    while (obj) {
        if (obj->data == ptr) {
            gc->roots[gc->root_count++] = obj;
            break;
        }
        obj = obj->next;
    }

    pthread_mutex_unlock(&gc->lock);
}

void gc_mark(garbage_collector_t* gc) {
    if (!gc) return;

    // 标记所有根对象
    for (size_t i = 0; i < gc->root_count; i++) {
        gc_object_t* obj = gc->roots[i];
        if (obj && !obj->marked) {
            obj->marked = 1;
        }
    }
}

void gc_sweep(garbage_collector_t* gc) {
    if (!gc) return;

    gc_object_t** current = &gc->objects;
    while (*current) {
        gc_object_t* obj = *current;

        if (!obj->marked) {
            // 删除未标记的对象
            *current = obj->next;

            if (obj->destructor && obj->data) {
                obj->destructor(obj->data);
            }
            if (obj->data) {
                free(obj->data);
            }
            free(obj);
        } else {
            // 重置标记
            obj->marked = 0;
            current = &obj->next;
        }
    }
}

void gc_collect(garbage_collector_t* gc) {
    if (!gc) return;

    pthread_mutex_lock(&gc->lock);
    gc_mark(gc);
    gc_sweep(gc);
    pthread_mutex_unlock(&gc->lock);
}

void destroy_garbage_collector(garbage_collector_t* gc) {
    if (!gc) return;

    // 清理所有对象
    gc_object_t* obj = gc->objects;
    while (obj) {
        gc_object_t* next = obj->next;

        if (obj->destructor && obj->data) {
            obj->destructor(obj->data);
        }
        if (obj->data) {
            free(obj->data);
        }
        free(obj);

        obj = next;
    }

    // 清理资源
    pthread_mutex_destroy(&gc->lock);
    free(gc->roots);
    free(gc);
}
```

## 4. 数据结构实现

### 4.1 动态数组实现

```c
typedef struct {
    void** data;
    size_t size;
    size_t capacity;
    size_t element_size;
    void (*destructor)(void*);
} dynamic_array_t;

dynamic_array_t* create_dynamic_array(size_t element_size, size_t initial_capacity,
                                    void (*destructor)(void*)) {
    dynamic_array_t* array = malloc(sizeof(dynamic_array_t));
    if (!array) return NULL;

    array->data = malloc(element_size * initial_capacity);
    if (!array->data) {
        free(array);
        return NULL;
    }

    array->size = 0;
    array->capacity = initial_capacity;
    array->element_size = element_size;
    array->destructor = destructor;

    return array;
}

int array_push_back(dynamic_array_t* array, void* element) {
    if (!array || !element) return -1;

    // 检查是否需要扩展
    if (array->size >= array->capacity) {
        size_t new_capacity = array->capacity * 2;
        void** new_data = realloc(array->data, array->element_size * new_capacity);
        if (!new_data) return -1;

        array->data = new_data;
        array->capacity = new_capacity;
    }

    // 复制元素
    memcpy((char*)array->data + array->size * array->element_size, element, array->element_size);
    array->size++;

    return 0;
}

void* array_get(dynamic_array_t* array, size_t index) {
    if (!array || index >= array->size) {
        return NULL;
    }

    return (char*)array->data + index * array->element_size;
}

int array_remove(dynamic_array_t* array, size_t index) {
    if (!array || index >= array->size) return -1;

    // 调用析构函数
    if (array->destructor) {
        void* element = array_get(array, index);
        array->destructor(element);
    }

    // 移动元素
    size_t elements_to_move = array->size - index - 1;
    if (elements_to_move > 0) {
        memmove((char*)array->data + index * array->element_size,
                (char*)array->data + (index + 1) * array->element_size,
                elements_to_move * array->element_size);
    }

    array->size--;
    return 0;
}

void destroy_dynamic_array(dynamic_array_t* array) {
    if (!array) return;

    // 调用析构函数
    if (array->destructor) {
        for (size_t i = 0; i < array->size; i++) {
            void* element = array_get(array, i);
            array->destructor(element);
        }
    }

    free(array->data);
    free(array);
}
```

### 4.2 哈希表实现

```c
typedef struct hash_entry {
    void* key;
    void* value;
    struct hash_entry* next;
} hash_entry_t;

typedef struct {
    hash_entry_t** buckets;
    size_t capacity;
    size_t size;
    size_t (*hash_func)(const void*);
    int (*key_compare)(const void*, const void*);
    void (*key_destructor)(void*);
    void (*value_destructor)(void*);
} hash_table_t;

// 默认哈希函数
size_t default_hash_func(const void* key) {
    // 简单的字符串哈希
    const char* str = (const char*)key;
    size_t hash = 5381;
    int c;

    while ((c = *str++)) {
        hash = ((hash << 5) + hash) + c;  // hash * 33 + c
    }

    return hash;
}

// 默认键比较函数
int default_key_compare(const void* key1, const void* key2) {
    return strcmp((const char*)key1, (const char*)key2);
}

hash_table_t* create_hash_table(size_t capacity,
                              size_t (*hash_func)(const void*),
                              int (*key_compare)(const void*, const void*),
                              void (*key_destructor)(void*),
                              void (*value_destructor)(void*)) {
    hash_table_t* table = malloc(sizeof(hash_table_t));
    if (!table) return NULL;

    table->buckets = calloc(capacity, sizeof(hash_entry_t*));
    if (!table->buckets) {
        free(table);
        return NULL;
    }

    table->capacity = capacity;
    table->size = 0;
    table->hash_func = hash_func ? hash_func : default_hash_func;
    table->key_compare = key_compare ? key_compare : default_key_compare;
    table->key_destructor = key_destructor;
    table->value_destructor = value_destructor;

    return table;
}

int hash_insert(hash_table_t* table, void* key, void* value) {
    if (!table || !key || !value) return -1;

    size_t hash = table->hash_func(key);
    size_t index = hash % table->capacity;

    // 检查是否已存在
    hash_entry_t* entry = table->buckets[index];
    while (entry) {
        if (table->key_compare(entry->key, key) == 0) {
            // 键已存在，更新值
            if (table->value_destructor && entry->value != value) {
                table->value_destructor(entry->value);
            }
            entry->value = value;
            return 0;
        }
        entry = entry->next;
    }

    // 创建新条目
    hash_entry_t* new_entry = malloc(sizeof(hash_entry_t));
    if (!new_entry) return -1;

    new_entry->key = key;
    new_entry->value = value;
    new_entry->next = table->buckets[index];
    table->buckets[index] = new_entry;
    table->size++;

    return 0;
}

void* hash_lookup(hash_table_t* table, const void* key) {
    if (!table || !key) return NULL;

    size_t hash = table->hash_func(key);
    size_t index = hash % table->capacity;

    hash_entry_t* entry = table->buckets[index];
    while (entry) {
        if (table->key_compare(entry->key, key) == 0) {
            return entry->value;
        }
        entry = entry->next;
    }

    return NULL;
}

int hash_remove(hash_table_t* table, const void* key) {
    if (!table || !key) return -1;

    size_t hash = table->hash_func(key);
    size_t index = hash % table->capacity;

    hash_entry_t** current = &table->buckets[index];
    while (*current) {
        hash_entry_t* entry = *current;

        if (table->key_compare(entry->key, key) == 0) {
            *current = entry->next;

            // 调用析构函数
            if (table->key_destructor) {
                table->key_destructor(entry->key);
            }
            if (table->value_destructor) {
                table->value_destructor(entry->value);
            }

            free(entry);
            table->size--;
            return 0;
        }

        current = &entry->next;
    }

    return -1;  // 键未找到
}

void destroy_hash_table(hash_table_t* table) {
    if (!table) return;

    // 清理所有条目
    for (size_t i = 0; i < table->capacity; i++) {
        hash_entry_t* entry = table->buckets[i];
        while (entry) {
            hash_entry_t* next = entry->next;

            if (table->key_destructor) {
                table->key_destructor(entry->key);
            }
            if (table->value_destructor) {
                table->value_destructor(entry->value);
            }

            free(entry);
            entry = next;
        }
    }

    free(table->buckets);
    free(table);
}
```

## 5. 算法实现

### 5.1 排序算法

```c
// 快速排序实现
void quick_sort(void* base, size_t nmemb, size_t size,
                int (*compar)(const void*, const void*)) {
    if (!base || nmemb <= 1 || size == 0 || !compar) return;

    // 递归太深时使用堆排序
    if (nmemb > 1000) {
        // 堆排序实现
        return;
    }

    // 快速排序
    if (nmemb <= 16) {
        // 小数组使用插入排序
        for (size_t i = 1; i < nmemb; i++) {
            void* key = (char*)base + i * size;
            size_t j = i;

            while (j > 0 && compar((char*)base + (j - 1) * size, key) > 0) {
                memcpy((char*)base + j * size, (char*)base + (j - 1) * size, size);
                j--;
            }

            memcpy((char*)base + j * size, key, size);
        }
        return;
    }

    // 选择枢轴
    void* pivot = (char*)base + (nmemb / 2) * size;

    // 分区
    size_t i = 0;
    size_t j = nmemb - 1;

    while (1) {
        while (i < nmemb && compar((char*)base + i * size, pivot) < 0) i++;
        while (j > 0 && compar((char*)base + j * size, pivot) > 0) j--;

        if (i >= j) break;

        // 交换元素
        char temp[size];
        memcpy(temp, (char*)base + i * size, size);
        memcpy((char*)base + i * size, (char*)base + j * size, size);
        memcpy((char*)base + j * size, temp, size);

        i++;
        j--;
    }

    // 递归排序
    quick_sort(base, i, size, compar);
    quick_sort((char*)base + i * size, nmemb - i, size, compar);
}

// 归并排序实现
void merge(void* base, size_t nmemb, size_t size,
           int (*compar)(const void*, const void*),
           void* temp) {
    if (nmemb <= 1) return;

    size_t mid = nmemb / 2;

    merge(base, mid, size, compar, temp);
    merge((char*)base + mid * size, nmemb - mid, size, compar, temp);

    // 合并
    size_t i = 0, j = mid, k = 0;

    while (i < mid && j < nmemb) {
        if (compar((char*)base + i * size, (char*)base + j * size) <= 0) {
            memcpy((char*)temp + k * size, (char*)base + i * size, size);
            i++;
        } else {
            memcpy((char*)temp + k * size, (char*)base + j * size, size);
            j++;
        }
        k++;
    }

    while (i < mid) {
        memcpy((char*)temp + k * size, (char*)base + i * size, size);
        i++;
        k++;
    }

    while (j < nmemb) {
        memcpy((char*)temp + k * size, (char*)base + j * size, size);
        j++;
        k++;
    }

    memcpy(base, temp, nmemb * size);
}

void merge_sort(void* base, size_t nmemb, size_t size,
                int (*compar)(const void*, const void*)) {
    if (!base || nmemb <= 1 || size == 0 || !compar) return;

    void* temp = malloc(nmemb * size);
    if (!temp) return;

    merge(base, nmemb, size, compar, temp);

    free(temp);
}

// 二分查找
void* binary_search(const void* key, const void* base, size_t nmemb, size_t size,
                   int (*compar)(const void*, const void*)) {
    if (!key || !base || nmemb == 0 || size == 0 || !compar) {
        return NULL;
    }

    size_t low = 0;
    size_t high = nmemb - 1;

    while (low <= high) {
        size_t mid = low + (high - low) / 2;
        const void* mid_elem = (const char*)base + mid * size;
        int cmp = compar(key, mid_elem);

        if (cmp == 0) {
            return (void*)mid_elem;
        } else if (cmp < 0) {
            if (mid == 0) break;
            high = mid - 1;
        } else {
            low = mid + 1;
        }
    }

    return NULL;
}
```

### 5.2 搜索算法

```c
// 线性搜索优化
void* linear_search(const void* key, const void* base, size_t nmemb, size_t size,
                   int (*compar)(const void*, const void*)) {
    if (!key || !base || nmemb == 0 || size == 0 || !compar) {
        return NULL;
    }

    // 字对齐的快速搜索
    const uintptr_t* word_ptr = (const uintptr_t*)base;
    uintptr_t word_key = *(const uintptr_t*)key;

    size_t words = (nmemb * size) / sizeof(uintptr_t);
    size_t i;

    for (i = 0; i < words; i++) {
        if (word_ptr[i] == word_key) {
            return (void*)((char*)base + i * sizeof(uintptr_t));
        }
    }

    // 检查剩余字节
    size_t bytes_processed = words * sizeof(uintptr_t);
    size_t remaining_bytes = nmemb * size - bytes_processed;

    for (i = 0; i < remaining_bytes; i++) {
        const void* elem = (const char*)base + bytes_processed + i;
        if (compar(key, elem) == 0) {
            return (void*)elem;
        }
    }

    return NULL;
}

// KMP搜索算法实现
int kmp_search_bytes(const unsigned char* text, size_t text_len,
                    const unsigned char* pattern, size_t pattern_len) {
    if (!text || !pattern || pattern_len == 0 || text_len < pattern_len) {
        return -1;
    }

    if (pattern_len == 1) {
        // 单字符搜索
        for (size_t i = 0; i < text_len; i++) {
            if (text[i] == pattern[0]) {
                return i;
            }
        }
        return -1;
    }

    // 构建LPS数组
    int* lps = malloc(pattern_len * sizeof(int));
    if (!lps) return -1;

    lps[0] = 0;
    int len = 0;
    int i = 1;

    while (i < (int)pattern_len) {
        if (pattern[i] == pattern[len]) {
            len++;
            lps[i] = len;
            i++;
        } else {
            if (len != 0) {
                len = lps[len - 1];
            } else {
                lps[i] = 0;
                i++;
            }
        }
    }

    // 搜索
    int result = -1;
    i = 0;
    int j = 0;

    while (i < (int)text_len) {
        if (pattern[j] == text[i]) {
            i++;
            j++;

            if (j == (int)pattern_len) {
                result = i - j;
                break;
            }
        } else {
            if (j != 0) {
                j = lps[j - 1];
            } else {
                i++;
            }
        }
    }

    free(lps);
    return result;
}
```

## 6. 性能优化与基准测试

### 6.1 性能基准测试框架

```c
#include <time.h>

typedef struct {
    const char* name;
    void (*func)(void*);
    void* arg;
    double elapsed_time;
    size_t iterations;
} benchmark_t;

void run_benchmark(benchmark_t* bench) {
    struct timespec start, end;

    clock_gettime(CLOCK_MONOTONIC, &start);

    for (size_t i = 0; i < bench->iterations; i++) {
        bench->func(bench->arg);
    }

    clock_gettime(CLOCK_MONOTONIC, &end);

    bench->elapsed_time = (end.tv_sec - start.tv_sec) +
                         (end.tv_nsec - start.tv_nsec) / 1e9;
}

void benchmark_string_functions() {
    const int string_length = 1000;
    const int iterations = 100000;

    // 准备测试数据
    char* test_string = malloc(string_length + 1);
    if (!test_string) return;

    memset(test_string, 'A', string_length);
    test_string[string_length] = '\0';

    // strlen测试
    benchmark_t strlen_bench = {
        .name = "strlen",
        .func = (void(*)(void*))strlen,
        .arg = test_string,
        .iterations = iterations
    };

    run_benchmark(&strlen_bench);
    printf("strlen: %.6f seconds for %zu iterations\n",
           strlen_bench.elapsed_time, strlen_bench.iterations);

    // 自定义strlen测试
    benchmark_t my_strlen_bench = {
        .name = "my_strlen",
        .func = (void(*)(void*))my_strlen,
        .arg = test_string,
        .iterations = iterations
    };

    run_benchmark(&my_strlen_bench);
    printf("my_strlen: %.6f seconds for %zu iterations\n",
           my_strlen_bench.elapsed_time, my_strlen_bench.iterations);

    // 性能比较
    double improvement = strlen_bench.elapsed_time / my_strlen_bench.elapsed_time;
    printf("Improvement: %.2fx\n", improvement);

    free(test_string);
}

void benchmark_memory_functions() {
    const int buffer_size = 1024 * 1024;  // 1MB
    const int iterations = 1000;

    // 准备测试数据
    void* src = malloc(buffer_size);
    void* dst = malloc(buffer_size);
    if (!src || !dst) {
        if (src) free(src);
        if (dst) free(dst);
        return;
    }

    memset(src, 0xAA, buffer_size);

    // memcpy测试
    benchmark_t memcpy_bench = {
        .name = "memcpy",
        .func = (void(*)(void*))memcpy,
        .arg = dst,
        .iterations = iterations
    };

    // 创建参数结构
    struct memcpy_args {
        void* dst;
        const void* src;
        size_t size;
    } args = {dst, src, buffer_size};

    benchmark_t my_memcpy_bench = {
        .name = "my_memcpy",
        .func = (void(*)(void*))my_memcpy,
        .arg = &args,
        .iterations = iterations
    };

    run_benchmark(&memcpy_bench);
    run_benchmark(&my_memcpy_bench);

    printf("memcpy: %.6f seconds for %zu iterations (%.2f MB/s)\n",
           memcpy_bench.elapsed_time, memcpy_bench.iterations,
           (buffer_size * iterations) / (memcpy_bench.elapsed_time * 1024 * 1024));

    printf("my_memcpy: %.6f seconds for %zu iterations (%.2f MB/s)\n",
           my_memcpy_bench.elapsed_time, my_memcpy_bench.iterations,
           (buffer_size * iterations) / (my_memcpy_bench.elapsed_time * 1024 * 1024));

    free(src);
    free(dst);
}
```

## 7. 最佳实践与注意事项

### 7.1 内存安全最佳实践

1. **边界检查**：始终检查缓冲区边界
2. **输入验证**：验证所有输入参数
3. **错误处理**：正确处理内存分配失败
4. **资源清理**：确保释放所有分配的资源
5. **线程安全**：在多线程环境中使用适当的同步机制

### 7.2 性能优化建议

1. **批量操作**：减少系统调用次数
2. **内存对齐**：使用对齐的内存访问
3. **缓存友好**：优化数据访问模式
4. **避免拷贝**：使用指针和引用
5. **预分配**：避免频繁的内存分配和释放

### 7.3 调试技巧

1. **内存检查**：使用valgrind等工具检测内存问题
2. **性能分析**：使用perf等工具分析性能瓶颈
3. **断言检查**：使用assert验证假设
4. **日志记录**：记录重要的操作和错误
5. **单元测试**：编写全面的测试用例

## 8. 总结

C语言标准库是系统编程的基础，深入理解其内部实现对于编写高性能、可靠的C程序至关重要。本文通过实现自定义的IO缓冲区、字符串处理函数、内存管理器和数据结构，揭示了标准库的设计原理和优化技巧。

关键要点：
- **IO缓冲区**：理解不同缓冲模式及其性能影响
- **字符串处理**：掌握优化算法和内存对齐技术
- **内存管理**：实现高效的内存分配和垃圾回收
- **数据结构**：理解常用数据结构的实现细节
- **算法优化**：选择合适的算法并进行性能优化

通过掌握这些技术，您可以更好地理解和使用C语言标准库，并在需要时实现自定义的高性能库函数。