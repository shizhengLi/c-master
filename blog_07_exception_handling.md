# C语言异常处理机制：setjmp/longjmp与错误处理模式

## 引言

C语言没有内置的异常处理机制，但提供了setjmp/longjmp函数来实现非局部跳转。这种机制可以模拟异常处理，但也带来了复杂性和风险。本文将深入探讨C语言的异常处理技术，包括setjmp/longjmp的使用、错误处理模式、资源管理和现代C语言的最佳实践。

## 1. setjmp/longjmp基础

### 1.1 基本概念和用法

```c
#include <setjmp.h>
#include <stdio.h>

// 基本的setjmp/longjmp示例
jmp_buf jump_buffer;

void nested_function() {
    printf("Inside nested_function\n");
    printf("Jumping back to main...\n");
    longjmp(jump_buffer, 1);  // 跳转到setjmp的位置
    printf("This line will never be executed\n");
}

void demonstrate_basic_jmp() {
    int ret_value = setjmp(jump_buffer);

    if (ret_value == 0) {
        printf("setjmp returned 0, normal execution\n");
        nested_function();
    } else {
        printf("longjmp returned %d, after jump\n", ret_value);
    }
}
```

### 1.2 返回值处理

```c
// 不同返回值的处理
void demonstrate_return_values() {
    jmp_buf env;

    int val = setjmp(env);
    if (val == 0) {
        printf("First time through setjmp\n");

        // 模拟不同类型的错误
        int error_type = 2;
        longjmp(env, error_type);
    } else {
        switch (val) {
            case 1:
                printf("Error type 1: Memory allocation failed\n");
                break;
            case 2:
                printf("Error type 2: File not found\n");
                break;
            case 3:
                printf("Error type 3: Invalid input\n");
                break;
            default:
                printf("Unknown error type: %d\n", val);
                break;
        }
    }
}
```

### 1.3 多层嵌套跳转

```c
// 多层函数调用中的跳转
jmp_buf global_env;

void level3() {
    printf("Entering level 3\n");
    printf("Critical error in level 3, jumping to top level\n");
    longjmp(global_env, 3);
    printf("This line in level 3 will not execute\n");
}

void level2() {
    printf("Entering level 2\n");
    level3();
    printf("This line in level 2 will not execute\n");
}

void level1() {
    printf("Entering level 1\n");
    level2();
    printf("This line in level 1 will not execute\n");
}

void demonstrate_nested_jumps() {
    int ret = setjmp(global_env);

    if (ret == 0) {
        printf("Starting normal execution\n");
        level1();
        printf("Normal execution completed\n");
    } else {
        printf("Error occurred at level %d\n", ret);
        printf("Recovering from error...\n");
    }
}
```

## 2. 异常处理框架

### 2.1 简单异常处理框架

```c
#include <stdlib.h>

// 异常类型定义
typedef enum {
    EXCEPTION_NONE = 0,
    EXCEPTION_MEMORY,
    EXCEPTION_FILE,
    EXCEPTION_NETWORK,
    EXCEPTION_INVALID_ARGUMENT,
    EXCEPTION_TIMEOUT,
    EXCEPTION_MAX
} exception_type_t;

// 异常处理上下文
typedef struct {
    jmp_buf env;
    exception_type_t exception;
    const char* message;
    void* cleanup_data;
    void (*cleanup_func)(void*);
} exception_context_t;

// 线程局部存储的异常上下文
static __thread exception_context_t* current_context = NULL;

// 异常抛出宏
#define THROW(type, msg) \
    do { \
        if (current_context) { \
            current_context->exception = (type); \
            current_context->message = (msg); \
            longjmp(current_context->env, 1); \
        } else { \
            fprintf(stderr, "Uncaught exception: %s\n", (msg)); \
            abort(); \
        } \
    } while(0)

// 异常处理宏
#define TRY \
    do { \
        exception_context_t __ctx; \
        memset(&__ctx, 0, sizeof(__ctx)); \
        exception_context_t* __old_context = current_context; \
        current_context = &__ctx; \
        if (setjmp(__ctx.env) == 0) {

#define CATCH(type, handler) \
        } else if (__ctx.exception == (type)) { \
            exception_context_t* __catch_ctx = &__ctx; \
            current_context = __old_context; \
            { handler }

#define FINALLY \
        } { \
            exception_context_t* __finally_ctx = &__ctx; \
            current_context = __old_context; \
            {

#define END_TRY \
            } \
        } \
    } while(0)

// 使用示例
void demonstrate_exception_framework() {
    TRY {
        printf("Inside try block\n");

        // 模拟正常操作
        int* ptr = malloc(sizeof(int));
        if (!ptr) {
            THROW(EXCEPTION_MEMORY, "Memory allocation failed");
        }

        *ptr = 42;
        printf("Value: %d\n", *ptr);
        free(ptr);

        // 模拟错误
        THROW(EXCEPTION_INVALID_ARGUMENT, "Invalid argument detected");

    } CATCH(EXCEPTION_MEMORY, {
        printf("Caught memory exception: %s\n", __catch_ctx->message);
    }) CATCH(EXCEPTION_INVALID_ARGUMENT, {
        printf("Caught invalid argument exception: %s\n", __catch_ctx->message);
    }) FINALLY {
        printf("Finally block executed\n");
    } END_TRY;
}
```

### 2.2 资源管理框架

```c
// 资源管理器
typedef struct {
    void* resource;
    void (*cleanup_func)(void*);
    struct resource_manager* next;
} resource_manager_t;

typedef struct {
    resource_manager_t* resources;
    jmp_buf env;
    exception_type_t exception;
    const char* message;
} resource_context_t;

// 注册清理函数
void register_cleanup(resource_context_t* ctx, void* resource, void (*cleanup_func)(void*)) {
    resource_manager_t* rm = malloc(sizeof(resource_manager_t));
    if (!rm) {
        return;
    }

    rm->resource = resource;
    rm->cleanup_func = cleanup_func;
    rm->next = ctx->resources;
    ctx->resources = rm;
}

// 清理所有资源
void cleanup_resources(resource_context_t* ctx) {
    resource_manager_t* current = ctx->resources;
    while (current) {
        resource_manager_t* next = current->next;
        if (current->cleanup_func && current->resource) {
            current->cleanup_func(current->resource);
        }
        free(current);
        current = next;
    }
    ctx->resources = NULL;
}

// 带资源管理的TRY宏
#define TRY_WITH_RESOURCES(ctx) \
    do { \
        resource_context_t __ctx = {0}; \
        if (setjmp(__ctx.env) == 0) {

#define CATCH_RESOURCES(type, handler) \
        } else if (__ctx.exception == (type)) { \
            cleanup_resources(&__ctx); \
            { handler }

#define FINALLY_RESOURCES \
        } { \
            cleanup_resources(&__ctx); \
            {

#define END_TRY_RESOURCES \
            } \
        } \
    } while(0)

// 使用示例
void* my_malloc(size_t size) {
    void* ptr = malloc(size);
    if (!ptr) {
        THROW(EXCEPTION_MEMORY, "Allocation failed");
    }
    return ptr;
}

void demonstrate_resource_management() {
    resource_context_t ctx = {0};

    TRY_WITH_RESOURCES(ctx) {
        printf("Allocating resources...\n");

        int* ptr1 = my_malloc(sizeof(int));
        register_cleanup(&ctx, ptr1, free);

        int* ptr2 = my_malloc(1000);
        register_cleanup(&ctx, ptr2, free);

        FILE* file = fopen("test.txt", "w");
        if (!file) {
            THROW(EXCEPTION_FILE, "Failed to open file");
        }
        register_cleanup(&ctx, file, (void(*)(void*))fclose);

        // 使用资源
        *ptr1 = 42;
        fprintf(file, "Test data: %d\n", *ptr1);

        // 模拟错误
        THROW(EXCEPTION_TIMEOUT, "Operation timed out");

    } CATCH_RESOURCES(EXCEPTION_MEMORY, {
        printf("Memory exception caught: %s\n", ctx.message);
    }) CATCH_RESOURCES(EXCEPTION_FILE, {
        printf("File exception caught: %s\n", ctx.message);
    }) CATCH_RESOURCES(EXCEPTION_TIMEOUT, {
        printf("Timeout exception caught: %s\n", ctx.message);
    }) FINALLY_RESOURCES {
        printf("Resources cleaned up\n");
    } END_TRY_RESOURCES;
}
```

## 3. 错误处理模式

### 3.1 错误码模式

```c
// 错误码定义
typedef enum {
    ERROR_SUCCESS = 0,
    ERROR_INVALID_PARAMETER = -1,
    ERROR_OUT_OF_MEMORY = -2,
    ERROR_FILE_NOT_FOUND = -3,
    ERROR_PERMISSION_DENIED = -4,
    ERROR_TIMEOUT = -5,
    ERROR_UNKNOWN = -99
} error_code_t;

// 错误码字符串表
const char* error_strings[] = {
    "Success",
    "Invalid parameter",
    "Out of memory",
    "File not found",
    "Permission denied",
    "Timeout",
    "Unknown error"
};

const char* error_to_string(error_code_t error) {
    if (error >= ERROR_UNKNOWN) {
        return error_strings[sizeof(error_strings)/sizeof(error_strings[0]) - 1];
    }
    return error_strings[-error];
}

// 错误码检查宏
#define CHECK_ERROR(call) \
    do { \
        error_code_t __err = (call); \
        if (__err != ERROR_SUCCESS) { \
            fprintf(stderr, "Error in %s: %s\n", #call, error_to_string(__err)); \
            return __err; \
        } \
    } while(0)

#define CHECK_ERROR_WITH_CLEANUP(call, cleanup) \
    do { \
        error_code_t __err = (call); \
        if (__err != ERROR_SUCCESS) { \
            fprintf(stderr, "Error in %s: %s\n", #call, error_to_string(__err)); \
            cleanup; \
            return __err; \
        } \
    } while(0)

// 使用示例
error_code_t process_file(const char* filename) {
    FILE* file = fopen(filename, "r");
    if (!file) {
        return ERROR_FILE_NOT_FOUND;
    }

    char* buffer = malloc(1024);
    if (!buffer) {
        fclose(file);
        return ERROR_OUT_OF_MEMORY;
    }

    // 处理文件
    size_t bytes_read = fread(buffer, 1, 1024, file);
    if (bytes_read == 0) {
        free(buffer);
        fclose(file);
        return ERROR_UNKNOWN;
    }

    printf("Read %zu bytes from file\n", bytes_read);

    free(buffer);
    fclose(file);
    return ERROR_SUCCESS;
}

void demonstrate_error_codes() {
    error_code_t err = process_file("nonexistent.txt");
    if (err != ERROR_SUCCESS) {
        printf("Error: %s\n", error_to_string(err));
    }
}
```

### 3.2 链式错误处理

```c
// 链式调用结构
typedef struct {
    error_code_t error;
    const char* function;
    int line;
    const char* message;
} error_context_t;

// 错误上下文宏
#define RETURN_ERROR(err, msg) \
    do { \
        error_context_t __ctx = {err, __func__, __LINE__, msg}; \
        return __ctx; \
    } while(0)

#define CHECK_CHAIN(call) \
    do { \
        error_context_t __ctx = (call); \
        if (__ctx.error != ERROR_SUCCESS) { \
            return __ctx; \
        } \
    } while(0)

// 链式函数示例
error_context_t read_file_header(FILE* file) {
    if (!file) {
        RETURN_ERROR(ERROR_INVALID_PARAMETER, "File pointer is NULL");
    }

    char header[4];
    if (fread(header, 1, 4, file) != 4) {
        RETURN_ERROR(ERROR_FILE_NOT_FOUND, "Failed to read header");
    }

    if (memcmp(header, "MAGIC", 4) != 0) {
        RETURN_ERROR(ERROR_INVALID_PARAMETER, "Invalid file header");
    }

    error_context_t success = {ERROR_SUCCESS, __func__, __LINE__, "Success"};
    return success;
}

error_context_t process_file_content(FILE* file) {
    CHECK_CHAIN(read_file_header(file));

    // 处理文件内容
    char buffer[1024];
    if (fread(buffer, 1, 1024, file) == 0) {
        RETURN_ERROR(ERROR_UNKNOWN, "Failed to read content");
    }

    error_context_t success = {ERROR_SUCCESS, __func__, __LINE__, "Success"};
    return success;
}

error_context_t complete_file_processing(const char* filename) {
    FILE* file = fopen(filename, "rb");
    if (!file) {
        RETURN_ERROR(ERROR_FILE_NOT_FOUND, "Cannot open file");
    }

    error_context_t result = process_file_content(file);
    fclose(file);

    return result;
}

void demonstrate_chained_errors() {
    error_context_t ctx = complete_file_processing("test.txt");

    if (ctx.error != ERROR_SUCCESS) {
        printf("Error in %s at line %d: %s\n",
               ctx.function, ctx.line, ctx.message);
    } else {
        printf("Processing completed successfully\n");
    }
}
```

### 3.3 观察者模式错误处理

```c
// 错误观察者接口
typedef struct error_observer {
    void (*on_error)(struct error_observer*, error_code_t, const char*);
    struct error_observer* next;
} error_observer_t;

// 错误通知系统
typedef struct {
    error_observer_t* observers;
    error_code_t last_error;
    char last_message[256];
} error_notifier_t;

void error_notifier_init(error_notifier_t* notifier) {
    notifier->observers = NULL;
    notifier->last_error = ERROR_SUCCESS;
    notifier->last_message[0] = '\0';
}

void add_error_observer(error_notifier_t* notifier, error_observer_t* observer) {
    observer->next = notifier->observers;
    notifier->observers = observer;
}

void notify_error(error_notifier_t* notifier, error_code_t error, const char* message) {
    notifier->last_error = error;
    strncpy(notifier->last_message, message, sizeof(notifier->last_message) - 1);
    notifier->last_message[sizeof(notifier->last_message) - 1] = '\0';

    error_observer_t* observer = notifier->observers;
    while (observer) {
        observer->on_error(observer, error, message);
        observer = observer->next;
    }
}

// 观察者实现
typedef struct {
    error_observer_t base;
    const char* name;
} console_observer_t;

void console_observer_on_error(error_observer_t* observer, error_code_t error, const char* message) {
    console_observer_t* console = (console_observer_t*)observer;
    printf("[%s] Error %d: %s\n", console->name, error, message);
}

typedef struct {
    error_observer_t base;
    FILE* log_file;
} file_observer_t;

void file_observer_on_error(error_observer_t* observer, error_code_t error, const char* message) {
    file_observer_t* file_obs = (file_observer_t*)observer;
    if (file_obs->log_file) {
        fprintf(file_obs->log_file, "Error %d: %s\n", error, message);
        fflush(file_obs->log_file);
    }
}

void demonstrate_observer_pattern() {
    error_notifier_t notifier;
    error_notifier_init(&notifier);

    // 创建观察者
    console_observer_t console_obs = {
        .base.on_error = console_observer_on_error,
        .name = "Console"
    };

    FILE* log_file = fopen("error.log", "a");
    file_observer_t file_obs = {
        .base.on_error = file_observer_on_error,
        .log_file = log_file
    };

    // 注册观察者
    add_error_observer(&notifier, &console_obs.base);
    add_error_observer(&notifier, &file_obs.base);

    // 触发错误
    notify_error(&notifier, ERROR_FILE_NOT_FOUND, "Configuration file not found");
    notify_error(&notifier, ERROR_TIMEOUT, "Database connection timeout");

    if (log_file) {
        fclose(log_file);
    }
}
```

## 4. 高级异常处理技术

### 4.1 上下文恢复

```c
// 上下文保存和恢复
typedef struct {
    jmp_buf env;
    void* saved_state;
    size_t state_size;
} exception_context_v2_t;

// 保存上下文
exception_context_v2_t* create_exception_context(size_t state_size) {
    exception_context_v2_t* ctx = malloc(sizeof(exception_context_v2_t));
    if (!ctx) return NULL;

    ctx->saved_state = malloc(state_size);
    if (!ctx->saved_state) {
        free(ctx);
        return NULL;
    }

    ctx->state_size = state_size;
    return ctx;
}

// 保存状态到上下文
void save_state(exception_context_v2_t* ctx, const void* state) {
    if (ctx && ctx->saved_state && state) {
        memcpy(ctx->saved_state, state, ctx->state_size);
    }
}

// 从上下文恢复状态
void restore_state(exception_context_v2_t* ctx, void* state) {
    if (ctx && ctx->saved_state && state) {
        memcpy(state, ctx->saved_state, ctx->state_size);
    }
}

// 清理上下文
void destroy_exception_context(exception_context_v2_t* ctx) {
    if (ctx) {
        if (ctx->saved_state) {
            free(ctx->saved_state);
        }
        free(ctx);
    }
}

// 使用示例
void demonstrate_context_restoration() {
    exception_context_v2_t* ctx = create_exception_context(sizeof(int) * 3);
    if (!ctx) {
        printf("Failed to create exception context\n");
        return;
    }

    int state[3] = {10, 20, 30};

    if (setjmp(ctx->env) == 0) {
        printf("Normal execution\n");
        save_state(ctx, state);

        // 修改状态
        state[0] = 100;
        state[1] = 200;
        state[2] = 300;

        printf("Modified state: %d, %d, %d\n", state[0], state[1], state[2]);

        // 模拟错误
        if (state[0] == 100) {
            longjmp(ctx->env, 1);
        }

    } else {
        printf("After longjmp\n");
        restore_state(ctx, state);
        printf("Restored state: %d, %d, %d\n", state[0], state[1], state[2]);
    }

    destroy_exception_context(ctx);
}
```

### 4.2 异常安全保证

```c
// 异常安全的数据结构
typedef struct {
    int* data;
    size_t size;
    size_t capacity;
    jmp_buf* env;
} safe_vector_t;

// 异常安全的向量创建
safe_vector_t* safe_vector_create(size_t capacity, jmp_buf* env) {
    safe_vector_t* vec = malloc(sizeof(safe_vector_t));
    if (!vec) {
        if (env) longjmp(*env, ERROR_OUT_OF_MEMORY);
        return NULL;
    }

    vec->data = malloc(capacity * sizeof(int));
    if (!vec->data) {
        free(vec);
        if (env) longjmp(*env, ERROR_OUT_OF_MEMORY);
        return NULL;
    }

    vec->size = 0;
    vec->capacity = capacity;
    vec->env = env;
    return vec;
}

// 异常安全的元素添加
void safe_vector_push(safe_vector_t* vec, int value) {
    if (vec->size >= vec->capacity) {
        size_t new_capacity = vec->capacity * 2;
        int* new_data = realloc(vec->data, new_capacity * sizeof(int));
        if (!new_data) {
            if (vec->env) longjmp(*vec->env, ERROR_OUT_OF_MEMORY);
            return;
        }
        vec->data = new_data;
        vec->capacity = new_capacity;
    }

    vec->data[vec->size++] = value;
}

// 异常安全的向量销毁
void safe_vector_destroy(safe_vector_t* vec) {
    if (vec) {
        if (vec->data) {
            free(vec->data);
        }
        free(vec);
    }
}

void demonstrate_exception_safety() {
    jmp_buf env;

    if (setjmp(env) == 0) {
        safe_vector_t* vec = safe_vector_create(4, &env);
        printf("Vector created successfully\n");

        safe_vector_push(vec, 10);
        safe_vector_push(vec, 20);
        safe_vector_push(vec, 30);

        printf("Vector size: %zu\n", vec->size);

        // 测试内存不足情况
        // safe_vector_push(vec, 40);  // 正常情况

        // 强制测试错误处理
        // vec->capacity = 0;  // 测试扩展失败

        safe_vector_destroy(vec);
        printf("Vector destroyed successfully\n");

    } else {
        printf("Exception caught during vector operations\n");
    }
}
```

### 4.3 异步安全异常处理

```c
#include <signal.h>
#include <unistd.h>

// 异步信号安全的异常处理
typedef struct {
    sig_atomic_t error_flag;
    sig_atomic_t error_code;
    jmp_buf env;
} async_exception_context_t;

static volatile async_exception_context_t* global_async_context = NULL;

// 信号处理函数
void async_signal_handler(int sig) {
    if (global_async_context) {
        global_async_context->error_flag = 1;
        global_async_context->error_code = sig;
        longjmp(global_async_context->env, 1);
    }
}

// 设置异步异常处理
void setup_async_exception(async_exception_context_t* ctx) {
    global_async_context = ctx;

    // 设置信号处理器
    struct sigaction sa;
    sa.sa_handler = async_signal_handler;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0;

    sigaction(SIGINT, &sa, NULL);
    sigaction(SIGTERM, &sa, NULL);
    sigaction(SIGUSR1, &sa, NULL);
}

// 清理异步异常处理
void cleanup_async_exception() {
    global_async_context = NULL;
}

// 异步安全的操作
void async_safe_operation() {
    printf("Starting async-safe operation...\n");

    for (int i = 0; i < 10; i++) {
        printf("Working... %d/10\n", i + 1);
        sleep(1);
    }

    printf("Operation completed safely\n");
}

void demonstrate_async_exceptions() {
    async_exception_context_t ctx = {0};

    if (setjmp(ctx.env) == 0) {
        setup_async_exception(&ctx);
        printf("Async exception handler set up\n");
        printf("Try sending SIGINT (Ctrl+C) to test async exception\n");

        async_safe_operation();

        cleanup_async_exception();
        printf("No async exceptions occurred\n");

    } else {
        cleanup_async_exception();
        printf("Async exception caught: signal %d\n", ctx.error_code);
    }
}
```

## 5. 现代C语言错误处理

### 5.1 类型安全的错误处理

```c
#include <stdalign.h>

// 类型安全的错误类型
typedef struct {
    alignas(16) error_code_t code;
    alignas(16) const char* message;
    alignas(16) const char* function;
    alignas(16) int line;
    alignas(16) const char* file;
} typed_error_t;

// 错误构造函数
static inline typed_error_t make_error(error_code_t code, const char* message,
                                       const char* function, int line, const char* file) {
    typed_error_t err = {
        .code = code,
        .message = message,
        .function = function,
        .line = line,
        .file = file
    };
    return err;
}

// 错误检查宏
#define MAKE_TYPED_ERROR(code, msg) \
    make_error(code, msg, __func__, __LINE__, __FILE__)

#define RETURN_TYPED_ERROR(code, msg) \
    return MAKE_TYPED_ERROR(code, msg)

// 可选类型
typedef struct {
    bool has_value;
    union {
        int value;
        typed_error_t error;
    };
} result_int_t;

// 成功结果构造
static inline result_int_t make_success_int(int value) {
    result_int_t result = {
        .has_value = true,
        .value = value
    };
    return result;
}

// 错误结果构造
static inline result_int_t make_error_int(typed_error_t error) {
    result_int_t result = {
        .has_value = false,
        .error = error
    };
    return result;
}

// 安全访问
static inline bool get_result_int(result_int_t result, int* out_value) {
    if (result.has_value) {
        if (out_value) *out_value = result.value;
        return true;
    }
    return false;
}

static inline typed_error_t get_error_int(result_int_t result) {
    return result.error;
}

// 使用示例
result_int_t safe_division(int a, int b) {
    if (b == 0) {
        RETURN_TYPED_ERROR(ERROR_INVALID_PARAMETER, "Division by zero");
    }
    return make_success_int(a / b);
}

result_int_t safe_string_to_int(const char* str) {
    if (!str) {
        RETURN_TYPED_ERROR(ERROR_INVALID_PARAMETER, "NULL string");
    }

    char* endptr;
    long value = strtol(str, &endptr, 10);

    if (*endptr != '\0') {
        RETURN_TYPED_ERROR(ERROR_INVALID_PARAMETER, "Invalid number format");
    }

    if (value < INT_MIN || value > INT_MAX) {
        RETURN_TYPED_ERROR(ERROR_INVALID_PARAMETER, "Number out of range");
    }

    return make_success_int((int)value);
}

void demonstrate_typed_errors() {
    // 测试安全除法
    result_int_t div_result = safe_division(10, 2);
    if (get_result_int(div_result, &(int){0})) {
        printf("Division result: %d\n", div_result.value);
    } else {
        typed_error_t err = get_error_int(div_result);
        printf("Division error: %s\n", err.message);
    }

    // 测试错误情况
    div_result = safe_division(10, 0);
    if (get_result_int(div_result, &(int){0})) {
        printf("Division result: %d\n", div_result.value);
    } else {
        typed_error_t err = get_error_int(div_result);
        printf("Division error: %s in %s at %s:%d\n",
               err.message, err.function, err.file, err.line);
    }

    // 测试字符串转换
    result_int_t conv_result = safe_string_to_int("123");
    if (get_result_int(conv_result, &(int){0})) {
        printf("Converted value: %d\n", conv_result.value);
    } else {
        typed_error_t err = get_error_int(conv_result);
        printf("Conversion error: %s\n", err.message);
    }
}
```

### 5.2 函数式错误处理

```c
// 函数式风格的错误处理
typedef struct {
    int value;
    typed_error_t error;
    bool is_success;
} either_int_error_t;

// Left构造（错误）
static inline either_int_error_t either_left(typed_error_t error) {
    either_int_error_t either = {
        .error = error,
        .is_success = false
    };
    return either;
}

// Right构造（成功）
static inline either_int_error_t either_right(int value) {
    either_int_error_t either = {
        .value = value,
        .is_success = true
    };
    return either;
}

// Either映射
either_int_error_t either_map(either_int_error_t either, int (*func)(int)) {
    if (either.is_success) {
        return either_right(func(either.value));
    } else {
        return either;
    }
}

// Either绑定
either_int_error_t either_bind(either_int_error_t either,
                              either_int_error_t (*func)(int)) {
    if (either.is_success) {
        return func(either.value);
    } else {
        return either;
    }
}

// 使用示例
either_int_error_t process_number(int x) {
    if (x < 0) {
        return either_left(MAKE_TYPED_ERROR(ERROR_INVALID_PARAMETER, "Negative number"));
    }
    return either_right(x * 2);
}

either_int_error_t validate_number(int x) {
    if (x > 1000) {
        return either_left(MAKE_TYPED_ERROR(ERROR_INVALID_PARAMETER, "Number too large"));
    }
    return either_right(x);
}

void demonstrate_functional_error_handling() {
    // 链式操作
    either_int_error_t result = either_right(10);
    result = either_bind(result, validate_number);
    result = either_map(result, process_number);

    if (result.is_success) {
        printf("Functional result: %d\n", result.value);
    } else {
        printf("Functional error: %s\n", result.error.message);
    }

    // 错误情况的链式操作
    result = either_right(-5);
    result = either_bind(result, validate_number);
    result = either_map(result, process_number);

    if (result.is_success) {
        printf("Functional result: %d\n", result.value);
    } else {
        printf("Functional error: %s\n", result.error.message);
    }
}
```

## 6. 调试与诊断

### 6.1 异常跟踪

```c
// 异常调用栈跟踪
#define MAX_STACK_TRACE 32

typedef struct {
    const char* function;
    const char* file;
    int line;
} stack_frame_t;

typedef struct {
    stack_frame_t frames[MAX_STACK_TRACE];
    int frame_count;
    typed_error_t error;
} exception_trace_t;

static __thread exception_trace_t current_trace = {0};

// 推入调用栈帧
void push_stack_frame(const char* function, const char* file, int line) {
    if (current_trace.frame_count < MAX_STACK_TRACE) {
        current_trace.frames[current_trace.frame_count].function = function;
        current_trace.frames[current_trace.frame_count].file = file;
        current_trace.frames[current_trace.frame_count].line = line;
        current_trace.frame_count++;
    }
}

// 弹出调用栈帧
void pop_stack_frame() {
    if (current_trace.frame_count > 0) {
        current_trace.frame_count--;
    }
}

// 设置异常
void set_exception_trace(typed_error_t error) {
    current_trace.error = error;
}

// 打印调用栈
void print_stack_trace() {
    printf("Exception stack trace:\n");
    printf("Error: %s in %s at %s:%d\n",
           current_trace.error.message,
           current_trace.error.function,
           current_trace.error.file,
           current_trace.error.line);

    printf("Call stack:\n");
    for (int i = current_trace.frame_count - 1; i >= 0; i--) {
        printf("  %s at %s:%d\n",
               current_trace.frames[i].function,
               current_trace.frames[i].file,
               current_trace.frames[i].line);
    }
}

// 跟踪宏
#define TRACE_ENTER() push_stack_frame(__func__, __FILE__, __LINE__)
#define TRACE_EXIT() pop_stack_frame()
#define TRACE_THROW(code, msg) \
    do { \
        set_exception_trace(MAKE_TYPED_ERROR(code, msg)); \
        longjmp(current_trace.env, 1); \
    } while(0)

void demonstrate_exception_tracing() {
    jmp_buf env;

    if (setjmp(env) == 0) {
        current_trace.env = env;
        current_trace.frame_count = 0;

        TRACE_ENTER();

        // 模拟深层调用
        {
            TRACE_ENTER();

            {
                TRACE_ENTER();

                // 触发异常
                TRACE_THROW(ERROR_TIMEOUT, "Operation timed out");

                TRACE_EXIT();
            }

            TRACE_EXIT();
        }

        TRACE_EXIT();

    } else {
        print_stack_trace();
    }
}
```

### 6.2 性能监控

```c
#include <time.h>

// 异常性能统计
typedef struct {
    size_t exception_count;
    size_t setjmp_count;
    size_t longjmp_count;
    double total_exception_time;
    struct timespec last_exception_time;
} exception_stats_t;

static exception_stats_t global_stats = {0};

// 更新异常统计
void update_exception_stats() {
    global_stats.exception_count++;
    global_stats.longjmp_count++;

    struct timespec now;
    clock_gettime(CLOCK_MONOTONIC, &now);

    if (global_stats.last_exception_time.tv_sec != 0) {
        double time_diff = (now.tv_sec - global_stats.last_exception_time.tv_sec) +
                         (now.tv_nsec - global_stats.last_exception_time.tv_nsec) / 1e9;
        global_stats.total_exception_time += time_diff;
    }

    global_stats.last_exception_time = now;
}

// 打印异常统计
void print_exception_stats() {
    printf("Exception Statistics:\n");
    printf("  Total exceptions: %zu\n", global_stats.exception_count);
    printf("  Total setjmp calls: %zu\n", global_stats.setjmp_count);
    printf("  Total longjmp calls: %zu\n", global_stats.longjmp_count);
    printf("  Total exception handling time: %.6f seconds\n", global_stats.total_exception_time);

    if (global_stats.exception_count > 0) {
        double avg_time = global_stats.total_exception_time / global_stats.exception_count;
        printf("  Average exception handling time: %.6f seconds\n", avg_time);
    }
}

// 性能监控的setjmp
#define MONITORED_SETJMP(env) \
    do { \
        global_stats.setjmp_count++; \
        if (setjmp(env) == 0) { \
            /* Normal execution */ \
        } else { \
            update_exception_stats(); \
            /* Exception handling */ \
        } \
    } while(0)

void demonstrate_exception_performance() {
    jmp_buf env;

    MONITORED_SETJMP(env) {
        printf("Normal execution path\n");

        // 模拟一些操作
        for (int i = 0; i < 1000000; i++) {
            // 一些计算
            volatile int x = i * i;
        }

        printf("Completing normal execution\n");

    } else {
        printf("Exception handling path\n");
    }

    print_exception_stats();
}
```

## 7. 最佳实践与注意事项

### 7.1 异常处理最佳实践

1. **最小化异常范围**：只在必要时使用异常处理
2. **清理资源**：确保异常时正确释放资源
3. **避免异常滥用**：不要用异常处理正常流程
4. **文档化异常**：清楚记录函数可能抛出的异常
5. **性能考虑**：setjmp/longjmp有性能开销

### 7.2 常见陷阱与解决方案

**陷阱1：资源泄漏**
```c
// 错误示例
void dangerous_function() {
    FILE* file = fopen("test.txt", "r");
    int* ptr = malloc(sizeof(int));

    if (some_error_condition) {
        longjmp(env, 1);  // 资源泄漏！
    }

    // 正常清理
    free(ptr);
    fclose(file);
}

// 正确做法
void safe_function() {
    FILE* file = NULL;
    int* ptr = NULL;

    file = fopen("test.txt", "r");
    if (!file) goto cleanup;

    ptr = malloc(sizeof(int));
    if (!ptr) goto cleanup;

    if (some_error_condition) {
        goto cleanup;
    }

cleanup:
    if (ptr) free(ptr);
    if (file) fclose(file);

    if (error_occurred) {
        longjmp(env, 1);
    }
}
```

**陷阱2：栈帧不一致**
```c
// 错误示例
void problematic_function() {
    int buffer[100];

    if (setjmp(env) == 0) {
        // 使用栈缓冲区
        strcpy(buffer, "Hello");

        some_function_that_might_longjmp();

        // 继续使用buffer，但可能已经被跳过
        printf("Buffer contents: %s\n", buffer);
    }
}
```

**陷阱3：信号处理冲突**
```c
// 错误示例
void signal_unsafe_function() {
    jmp_buf env;

    if (setjmp(env) == 0) {
        // 在信号处理函数中调用longjmp
        signal(SIGINT, signal_handler);

        // 危险：信号可能在setjmp保存环境之后但完成设置之前触发
    }
}
```

### 7.3 调试技巧

```c
// 调试辅助宏
#define DEBUG_JMP(env) \
    printf("setjmp called at %s:%d, env=%p\n", __func__, __LINE__, (void*)(env))

#define DEBUG_LONGJMP(env, val) \
    printf("longjmp called with env=%p, val=%d\n", (void*)(env), (val))

// 环境检查
void check_jmp_buf(jmp_buf* env) {
    printf("Jump buffer check:\n");
    printf("  Address: %p\n", (void*)env);

    // 注意：直接检查jmp_buf内容是平台相关的
    #if defined(__x86_64__)
        printf("  Architecture: x86_64\n");
    #elif defined(__aarch64__)
        printf("  Architecture: ARM64\n");
    #endif
}

// 使用示例
void demonstrate_debugging() {
    jmp_buf env;
    DEBUG_JMP(&env);

    if (setjmp(env) == 0) {
        printf("Normal execution\n");

        // 检查环境
        check_jmp_buf(&env);

        // 模拟longjmp
        if (1) {
            DEBUG_LONGJMP(&env, 42);
        }

    } else {
        printf("After longjmp\n");
        check_jmp_buf(&env);
    }
}
```

## 8. 总结

### 8.1 异常处理技术总结

1. **setjmp/longjmp**：C语言的非局部跳转机制
2. **异常处理框架**：基于setjmp/longjmp的高级封装
3. **错误处理模式**：错误码、链式错误、观察者模式
4. **资源管理**：确保异常时的资源清理
5. **现代实践**：类型安全、函数式错误处理

### 8.2 适用场景

- **系统编程**：操作系统内核、设备驱动
- **嵌入式系统**：资源受限的环境
- **高性能计算**：需要低开销的错误处理
- **遗留系统**：维护使用setjmp/longjmp的代码

### 8.3 发展趋势

虽然C语言没有内置异常处理，但现代C语言实践倾向于：

1. **错误码优先**：简单、可预测、低开销
2. **资源管理器**：自动化的资源清理
3. **类型安全**：编译时错误检查
4. **函数式风格**：可选类型和Either模式
5. **调试支持**：更好的错误诊断工具

通过掌握C语言的异常处理技术，您可以在需要时实现复杂的错误处理逻辑，同时保持代码的健壮性和可维护性。记住，异常处理是强大的工具，但需要谨慎使用，避免过度设计。