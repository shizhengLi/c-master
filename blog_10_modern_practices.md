# C语言现代实践：工具链、调试技术与性能分析

## 引言

现代C语言开发已经从传统的单一文件编译发展为复杂的工具链生态系统。构建系统、静态分析、调试工具、性能分析等工具的有机结合，构成了现代C语言开发的基础设施。本文将深入探讨现代C语言开发的工具链、调试技术和性能分析方法，帮助开发者在保持C语言高效性的同时，提升开发效率和代码质量。

## 1. 现代C语言工具链

### 1.1 构建系统演进

#### 传统构建方式

```c
/* 传统Makefile示例 */
CC = gcc
CFLAGS = -Wall -Wextra -O2
TARGET = program

SRCS = main.c utils.c network.c
OBJS = $(SRCS:.c=.o)

all: $(TARGET)

$(TARGET): $(OBJS)
	$(CC) $(OBJS) -o $(TARGET) -lm -lpthread

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

clean:
	rm -f $(OBJS) $(TARGET)

.PHONY: all clean
```

#### 现代构建系统

```c
/* CMakeLists.txt 现代构建系统示例 */
cmake_minimum_required(VERSION 3.10)
project(ModernCProgram C)

# 设置C标准
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)

# 编译选项
add_compile_options(-Wall -Wextra -Wpedantic)
add_compile_options(-O2 -march=native)

# 查找依赖包
find_package(Threads REQUIRED)
find_package(PkgConfig REQUIRED)
pkg_check_modules(SDL2 REQUIRED sdl2)

# 可执行文件
add_executable(${PROJECT_NAME}
    src/main.c
    src/utils.c
    src/network.c
    src/database.c
)

# 链接库
target_link_libraries(${PROJECT_NAME}
    PRIVATE
        Threads::Threads
        ${SDL2_LIBRARIES}
        m
)

# 包含目录
target_include_directories(${PROJECT_NAME}
    PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}/include
        ${SDL2_INCLUDE_DIRS}
)

# 安装规则
install(TARGETS ${PROJECT_NAME}
    RUNTIME DESTINATION bin
)
```

### 1.2 包管理与依赖管理

```c
/* conanfile.txt - Conan包管理示例 */
[requires]
openssl/1.1.1l
zlib/1.2.11
libcurl/7.80.0

[generators]
cmake
cmake_find_package

[options]
openssl:shared=True
libcurl:with_openssl=True

/* vcpkg.json - Vcpkg包管理示例 */
{
    "name": "modern-c-program",
    "version": "1.0.0",
    "dependencies": [
        "openssl",
        "zlib",
        "libcurl",
        "sqlite3",
        "gtest"
    ],
    "features": {
        "ssl": {
            "description": "Enable SSL support",
            "dependencies": [
                "openssl"
            ]
        }
    }
}
```

### 1.3 跨平台构建配置

```c
/* meson.build - Meson构建系统 */
project('modern-c-program', 'c',
    version: '1.0.0',
    default_options: [
        'c_std=c11',
        'warning_level=2',
        'optimization=2'
    ]
)

# 编译器检测
cc = meson.get_compiler('c')

# 依赖检查
math_dep = cc.find_library('m', required: false)
thread_dep = dependency('threads')
sdl2_dep = dependency('sdl2', required: get_option('sdl'))

# 配置数据
conf_data = configuration_data()
conf_data.set('VERSION', meson.project_version())
conf_data.set('HAVE_SDL2', sdl2_dep.found())

# 配置头文件
config_h = configure_file(
    input: 'config.h.in',
    output: 'config.h',
    configuration: conf_data
)

# 源文件
sources = [
    'src/main.c',
    'src/utils.c',
    'src/network.c',
    'src/database.c'
]

# 可执行文件
executable(
    'modern-c-program',
    sources,
    config_h,
    dependencies: [math_dep, thread_dep, sdl2_dep],
    install: true,
    install_dir: 'bin'
)
```

## 2. 静态代码分析与质量保证

### 2.1 静态分析工具集成

```c
/* .clang-tidy - Clang-Tidy配置 */
Checks: '-*, readability-identifier-naming, modernize-use-nullptr,
        performance-unnecessary-value-param, bugprone-*
WarningsAsErrors: '*'
HeaderFilterRegex: '.*'
AnalyzeTemporaryDtors: false
FormatStyle: file

/* .cppcheck.json - Cppcheck配置 */
{
    "suppressions": [
        {
            "id": "unusedFunction",
            "fileName": "test_.*\\.c"
        }
    ],
    "addons": ["misra"],
    "rules": [
        {
            "severity": "warning",
            "id": "variableScope"
        }
    ]
}
```

### 2.2 自定义静态分析脚本

```c
/* static_analyzer.h */
#ifndef STATIC_ANALYZER_H
#define STATIC_ANALYZER_H

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <regex.h>
#include <stdbool.h>

typedef struct {
    char* file_path;
    int total_lines;
    int comment_lines;
    int code_lines;
    int function_count;
    int complexity_score;
} code_metrics_t;

typedef struct {
    char* rule_name;
    char* pattern;
    int severity;  // 0=info, 1=warning, 2=error
    char* description;
} analysis_rule_t;

/* 分析规则定义 */
static analysis_rule_t analysis_rules[] = {
    {"no_goto", "\\bgoto\\b", 2, "Avoid using goto statements"},
    {"no_magic_numbers", "[^0-9a-fA-F_](\\d{3,})[^0-9a-fA-F_]", 1, "Magic number detected"},
    {"function_length", "^[a-zA-Z_][a-zA-Z0-9_]*\\(.*\\)\\s*\\{", 1, "Function too long"},
    {"pointer_arithmetic", "\\*\\s*\\+|\\+\\s*\\*|\\*\\s*-|-\\s*\\*", 1, "Pointer arithmetic detected"},
    {"unused_variable", "^\\s*[a-zA-Z_][a-zA-Z0-9_]*\\s+[a-zA-Z_][a-zA-Z0-9_]*\\s*;", 1, "Potentially unused variable"},
    {NULL, NULL, 0, NULL}
};

/* 代码分析函数 */
code_metrics_t* analyze_code_file(const char* file_path);
void print_code_metrics(const code_metrics_t* metrics);
void apply_analysis_rules(const char* file_path, const analysis_rule_t* rules);
bool matches_pattern(const char* line, const char* pattern);
void calculate_complexity(const char* file_path, code_metrics_t* metrics);

#endif

/* static_analyzer.c */
#include "static_analyzer.h"

code_metrics_t* analyze_code_file(const char* file_path) {
    FILE* file = fopen(file_path, "r");
    if (!file) {
        perror("Failed to open file");
        return NULL;
    }

    code_metrics_t* metrics = calloc(1, sizeof(code_metrics_t));
    if (!metrics) {
        fclose(file);
        return NULL;
    }

    metrics->file_path = strdup(file_path);
    char line[4096];
    bool in_comment = false;
    bool in_function = false;
    int brace_count = 0;

    while (fgets(line, sizeof(line), file)) {
        metrics->total_lines++;

        // 去除空白字符
        char* trimmed = line;
        while (isspace(*trimmed)) trimmed++;

        // 跳过空行
        if (strlen(trimmed) == 0) continue;

        // 检查注释
        if (strstr(trimmed, "//") == trimmed) {
            metrics->comment_lines++;
            continue;
        }

        // 处理多行注释
        if (strstr(trimmed, "/*") != NULL) {
            in_comment = true;
            metrics->comment_lines++;
        }
        if (in_comment) {
            if (strstr(trimmed, "*/") != NULL) {
                in_comment = false;
            }
            metrics->comment_lines++;
            continue;
        }

        // 代码行
        metrics->code_lines++;

        // 检测函数定义
        if (strstr(trimmed, "(") != NULL && strstr(trimmed, ")") != NULL &&
            strstr(trimmed, "{") != NULL && !in_function) {
            metrics->function_count++;
            in_function = true;
            brace_count = 1;
        }

        // 跟踪函数块
        if (in_function) {
            char* brace = strchr(trimmed, '{');
            while (brace) {
                brace_count++;
                brace = strchr(brace + 1, '{');
            }

            brace = strchr(trimmed, '}');
            while (brace) {
                brace_count--;
                if (brace_count == 0) {
                    in_function = false;
                    break;
                }
                brace = strchr(brace + 1, '}');
            }
        }
    }

    fclose(file);
    calculate_complexity(file_path, metrics);
    return metrics;
}

void calculate_complexity(const char* file_path, code_metrics_t* metrics) {
    FILE* file = fopen(file_path, "r");
    if (!file) return;

    char line[4096];
    int complexity = 1;  // 基础复杂度

    while (fgets(line, sizeof(line), file)) {
        // 计算控制流关键字
        if (strstr(line, "if")) complexity++;
        if (strstr(line, "else if")) complexity++;
        if (strstr(line, "while")) complexity++;
        if (strstr(line, "for")) complexity++;
        if (strstr(line, "switch")) complexity++;
        if (strstr(line, "case")) complexity++;
        if (strstr(line, "&&")) complexity++;
        if (strstr(line, "||")) complexity++;
        if (strstr(line, "?")) complexity++;
    }

    metrics->complexity_score = complexity;
    fclose(file);
}

void apply_analysis_rules(const char* file_path, const analysis_rule_t* rules) {
    FILE* file = fopen(file_path, "r");
    if (!file) return;

    char line[4096];
    int line_number = 0;

    printf("Analyzing file: %s\n", file_path);
    printf("=====================================\n");

    while (fgets(line, sizeof(line), file)) {
        line_number++;

        for (int i = 0; rules[i].rule_name != NULL; i++) {
            if (matches_pattern(line, rules[i].pattern)) {
                const char* severity = rules[i].severity == 2 ? "ERROR" :
                                      rules[i].severity == 1 ? "WARNING" : "INFO";
                printf("%s:%d: %s: %s\n", severity, line_number,
                       rules[i].rule_name, rules[i].description);
                printf("    %s", line);
            }
        }
    }

    fclose(file);
}

bool matches_pattern(const char* line, const char* pattern) {
    regex_t regex;
    int ret = regcomp(&regex, pattern, REG_EXTENDED);
    if (ret != 0) return false;

    ret = regexec(&regex, line, 0, NULL, 0);
    regfree(&regex);

    return ret == 0;
}

void print_code_metrics(const code_metrics_t* metrics) {
    if (!metrics) return;

    printf("Code Metrics for: %s\n", metrics->file_path);
    printf("=====================================\n");
    printf("Total lines: %d\n", metrics->total_lines);
    printf("Code lines: %d\n", metrics->code_lines);
    printf("Comment lines: %d\n", metrics->comment_lines);
    printf("Function count: %d\n", metrics->function_count);
    printf("Complexity score: %d\n", metrics->complexity_score);
    printf("Comment ratio: %.2f%%\n",
           (float)metrics->comment_lines / metrics->total_lines * 100);
    printf("Avg function size: %.1f lines\n",
           (float)metrics->code_lines / (metrics->function_count ?: 1));
    printf("\n");
}
```

### 2.3 代码质量门禁

```c
/* quality_gate.h */
#ifndef QUALITY_GATE_H
#define QUALITY_GATE_H

#include "static_analyzer.h"

typedef struct {
    int max_complexity;
    int max_function_length;
    int min_comment_ratio;
    int max_file_length;
    int max_parameter_count;
    int min_test_coverage;
} quality_thresholds_t;

typedef struct {
    bool passed;
    int failed_checks;
    char* failure_messages[10];
    int message_count;
} quality_result_t;

quality_result_t* check_quality_gate(const code_metrics_t* metrics,
                                     const quality_thresholds_t* thresholds);
void add_failure_message(quality_result_t* result, const char* message);
void print_quality_result(const quality_result_t* result);

#endif

/* quality_gate.c */
#include "quality_gate.h"

quality_result_t* check_quality_gate(const code_metrics_t* metrics,
                                     const quality_thresholds_t* thresholds) {
    quality_result_t* result = calloc(1, sizeof(quality_result_t));
    if (!result) return NULL;

    result->passed = true;

    // 检查复杂度
    if (metrics->complexity_score > thresholds->max_complexity) {
        char msg[256];
        snprintf(msg, sizeof(msg), "Complexity %d exceeds threshold %d",
                metrics->complexity_score, thresholds->max_complexity);
        add_failure_message(result, msg);
        result->passed = false;
    }

    // 检查函数数量
    if (metrics->function_count > thresholds->max_function_length) {
        char msg[256];
        snprintf(msg, sizeof(msg), "Function count %d exceeds threshold %d",
                metrics->function_count, thresholds->max_function_length);
        add_failure_message(result, msg);
        result->passed = false;
    }

    // 检查注释比例
    float comment_ratio = (float)metrics->comment_lines / metrics->total_lines * 100;
    if (comment_ratio < thresholds->min_comment_ratio) {
        char msg[256];
        snprintf(msg, sizeof(msg), "Comment ratio %.2f%% below threshold %d%%",
                comment_ratio, thresholds->min_comment_ratio);
        add_failure_message(result, msg);
        result->passed = false;
    }

    // 检查文件长度
    if (metrics->total_lines > thresholds->max_file_length) {
        char msg[256];
        snprintf(msg, sizeof(msg), "File length %d exceeds threshold %d",
                metrics->total_lines, thresholds->max_file_length);
        add_failure_message(result, msg);
        result->passed = false;
    }

    return result;
}

void add_failure_message(quality_result_t* result, const char* message) {
    if (result->message_count < 10) {
        result->failure_messages[result->message_count] = strdup(message);
        result->message_count++;
        result->failed_checks++;
    }
}

void print_quality_result(const quality_result_t* result) {
    if (!result) return;

    printf("Quality Gate Result: %s\n", result->passed ? "PASSED" : "FAILED");
    printf("========================\n");

    if (!result->passed) {
        printf("Failed checks (%d):\n", result->failed_checks);
        for (int i = 0; i < result->message_count; i++) {
            printf("  - %s\n", result->failure_messages[i]);
        }
    } else {
        printf("All quality checks passed!\n");
    }
}
```

## 3. 调试技术与工具

### 3.1 GDB高级调试技巧

```c
/* debug_utils.h */
#ifndef DEBUG_UTILS_H
#define DEBUG_UTILS_H

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <execinfo.h>
#include <signal.h>
#include <unistd.h>
#include <dlfcn.h>

/* 调试宏定义 */
#ifdef DEBUG
    #define DEBUG_PRINT(fmt, args...) \
        fprintf(stderr, "DEBUG [%s:%d] " fmt "\n", __FILE__, __LINE__, ##args)
    #define ASSERT(condition, message) \
        do { \
            if (!(condition)) { \
                fprintf(stderr, "Assertion failed: %s\n", message); \
                print_stack_trace(); \
                abort(); \
            } \
        } while (0)
#else
    #define DEBUG_PRINT(fmt, args...)
    #define ASSERT(condition, message)
#endif

/* 函数追踪宏 */
#define TRACE_FUNCTION() \
    printf("Entering %s\n", __func__)
#define TRACE_FUNCTION_EXIT() \
    printf("Exiting %s\n", __func__)

/* 内存调试宏 */
#define DEBUG_ALLOC(ptr, size) \
    debug_alloc(ptr, size, __FILE__, __LINE__)
#define DEBUG_FREE(ptr) \
    debug_free(ptr, __FILE__, __LINE__)

/* 调试函数声明 */
void print_stack_trace(void);
void setup_signal_handlers(void);
void* debug_alloc(void* ptr, size_t size, const char* file, int line);
void debug_free(void* ptr, const char* file, int line);
void dump_memory(void* ptr, size_t size);
void enable_memory_leak_detection(void);

#endif

/* debug_utils.c */
#include "debug_utils.h"

typedef struct {
    void* ptr;
    size_t size;
    char* file;
    int line;
    time_t timestamp;
} allocation_record_t;

static allocation_record_t* allocations = NULL;
static size_t allocation_count = 0;
static size_t allocation_capacity = 0;

void print_stack_trace(void) {
    void* callstack[128];
    int frames = backtrace(callstack, 128);
    char** strs = backtrace_symbols(callstack, frames);

    fprintf(stderr, "Stack trace:\n");
    for (int i = 0; i < frames; ++i) {
        fprintf(stderr, "%s\n", strs[i]);
    }

    free(strs);
}

void setup_signal_handlers(void) {
    signal(SIGSEGV, signal_handler);
    signal(SIGABRT, signal_handler);
    signal(SIGFPE, signal_handler);
    signal(SIGILL, signal_handler);
}

void signal_handler(int sig) {
    fprintf(stderr, "Caught signal %d\n", sig);
    print_stack_trace();
    exit(sig);
}

void* debug_alloc(void* ptr, size_t size, const char* file, int line) {
    ptr = malloc(size);
    if (!ptr) {
        fprintf(stderr, "Memory allocation failed at %s:%d\n", file, line);
        return NULL;
    }

    // 记录分配
    if (allocation_count >= allocation_capacity) {
        allocation_capacity = allocation_capacity == 0 ? 1024 : allocation_capacity * 2;
        allocations = realloc(allocations, allocation_capacity * sizeof(allocation_record_t));
    }

    allocations[allocation_count].ptr = ptr;
    allocations[allocation_count].size = size;
    allocations[allocation_count].file = strdup(file);
    allocations[allocation_count].line = line;
    allocations[allocation_count].timestamp = time(NULL);
    allocation_count++;

    DEBUG_PRINT("Allocated %zu bytes at %p (%s:%d)", size, ptr, file, line);
    return ptr;
}

void debug_free(void* ptr, const char* file, int line) {
    if (!ptr) {
        fprintf(stderr, "Attempt to free NULL pointer at %s:%d\n", file, line);
        return;
    }

    // 查找并删除分配记录
    for (size_t i = 0; i < allocation_count; i++) {
        if (allocations[i].ptr == ptr) {
            free(allocations[i].file);
            // 移动最后一个元素到当前位置
            allocations[i] = allocations[allocation_count - 1];
            allocation_count--;
            break;
        }
    }

    free(ptr);
    DEBUG_PRINT("Freed pointer %p (%s:%d)", ptr, file, line);
}

void dump_memory(void* ptr, size_t size) {
    unsigned char* bytes = (unsigned char*)ptr;
    printf("Memory dump at %p (size: %zu):\n", ptr, size);

    for (size_t i = 0; i < size; i += 16) {
        printf("%08zx: ", i);

        // 十六进制输出
        for (int j = 0; j < 16 && i + j < size; j++) {
            printf("%02x ", bytes[i + j]);
        }

        // ASCII输出
        printf(" |");
        for (int j = 0; j < 16 && i + j < size; j++) {
            char c = bytes[i + j];
            printf("%c", isprint(c) ? c : '.');
        }
        printf("|\n");
    }
}

void enable_memory_leak_detection(void) {
    atexit(memory_leak_report);
}

void memory_leak_report(void) {
    if (allocation_count == 0) {
        printf("No memory leaks detected.\n");
        return;
    }

    printf("\n=== MEMORY LEAK REPORT ===\n");
    printf("Total leaked allocations: %zu\n", allocation_count);
    printf("Total leaked memory: %zu bytes\n",
           get_total_leaked_memory());

    printf("\nLeak details:\n");
    for (size_t i = 0; i < allocation_count; i++) {
        printf("  %p: %zu bytes allocated at %s:%d\n",
               allocations[i].ptr, allocations[i].size,
               allocations[i].file, allocations[i].line);
    }

    printf("=========================\n");
}

size_t get_total_leaked_memory(void) {
    size_t total = 0;
    for (size_t i = 0; i < allocation_count; i++) {
        total += allocations[i].size;
    }
    return total;
}
```

### 3.2 自定义GDB命令

```c
/* gdb_commands.py - Python GDB脚本 */
import gdb
import re

class PrintStructCommand(gdb.Command):
    def __init__(self):
        super(PrintStructCommand, self).__init__("print_struct",
                                                gdb.COMMAND_USER_DEFINED)

    def invoke(self, arg, from_tty):
        if not arg:
            print("Usage: print_struct <struct_variable>")
            return

        try:
            # 获取结构体变量
            struct_var = gdb.parse_and_eval(arg)
            struct_type = struct_var.type

            print(f"Structure: {struct_type.name}")
            print("=" * 50)

            # 打印每个字段
            for field in struct_type.fields():
                field_name = field.name
                field_type = field.type
                field_value = struct_var[field_name]

                print(f"{field_name:20} ({field_type}): {field_value}")

        except Exception as e:
            print(f"Error: {e}")

class TraceFunctionCommand(gdb.Command):
    def __init__(self):
        super(TraceFunctionCommand, self).__init__("trace_function",
                                                   gdb.COMMAND_USER_DEFINED)

    def invoke(self, arg, from_tty):
        if not arg:
            print("Usage: trace_function <function_name>")
            return

        # 设置断点
        bp = gdb.Breakpoint(arg)
        bp.silent = True

        # 设置回调
        def stop_handler(event):
            if hasattr(event, 'breakpoint') and event.breakpoint == bp:
                frame = gdb.selected_frame()
                print(f"Called {arg} from {frame.name()}")
                # 打印参数
                try:
                    block = frame.block()
                    for symbol in block:
                        if symbol.is_argument:
                            print(f"  {symbol.name}: {symbol.value(frame)}")
                except:
                    pass

        gdb.events.stop.connect(stop_handler)

class MemoryMapCommand(gdb.Command):
    def __init__(self):
        super(MemoryMapCommand, self).__init__("mem_map",
                                               gdb.COMMAND_USER_DEFINED)

    def invoke(self, arg, from_tty):
        try:
            inferior = gdb.selected_inferior()
            maps = inferior.read_memory_maps()

            print("Memory Map:")
            print("=" * 80)
            print(f"{'Start':<16} {'End':<16} {'Size':<12} {'Permissions':<10} {'File'}")
            print("-" * 80)

            for mmap in maps:
                size = mmap.end - mmap.start
                perms = mmap.permissions
                filename = getattr(mmap, 'filename', '[anonymous]')

                print(f"{mmap.start:#016x} {mmap.end:#016x} {size:#010x} {perms:<10} {filename}")

        except Exception as e:
            print(f"Error: {e}")

# 注册命令
PrintStructCommand()
TraceFunctionCommand()
MemoryMapCommand()

/* .gdbinit - GDB初始化文件 */
set pagination off
set height 0
set print pretty on
set print array on
set print union on
set print demangle on
set history save on
set history filename ~/.gdb_history

# 加载自定义命令
source gdb_commands.py

# 定义常用别名
alias pp = print_struct
alias tf = trace_function
alias mm = mem_map

# 默认断点设置
break assert_fail
break exit
abort

# 显示欢迎信息
echo "GDB initialized with custom commands.\n"
echo "Available commands: print_struct, trace_function, mem_map\n"
echo "Type 'help' for more information.\n"
```

### 3.3 内存调试与泄漏检测

```c
/* memory_debugger.h */
#ifndef MEMORY_DEBUGGER_H
#define MEMORY_DEBUGGER_H

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <pthread.h>
#include <sys/mman.h>

/* 内存调试配置 */
typedef struct {
    bool enable_bounds_checking;
    bool enable_pattern_fill;
    bool enable_leak_detection;
    bool enable_thread_safety;
    size_t alignment;
    unsigned char fill_pattern;
    unsigned char free_pattern;
} memory_debug_config_t;

/* 内存块头信息 */
typedef struct memory_block_header {
    size_t size;
    const char* file;
    int line;
    struct memory_block_header* next;
    struct memory_block_header* prev;
    unsigned char canary[4];
    pthread_t thread_id;
    time_t timestamp;
} memory_block_header_t;

/* 全局配置 */
extern memory_debug_config_t g_mem_debug_config;
extern memory_block_header_t* g_allocated_blocks;
extern pthread_mutex_t g_memory_mutex;

/* 内存调试函数 */
void memory_debug_init(const memory_debug_config_t* config);
void* debug_malloc(size_t size, const char* file, int line);
void* debug_calloc(size_t nmemb, size_t size, const char* file, int line);
void* debug_realloc(void* ptr, size_t size, const char* file, int line);
void debug_free(void* ptr, const char* file, int line);
void memory_debug_report(void);
void memory_debug_check_bounds(void);
void memory_debug_set_pattern(unsigned char alloc_pattern, unsigned char free_pattern);

/* 便利宏 */
#define DEBUG_MALLOC(size) debug_malloc(size, __FILE__, __LINE__)
#define DEBUG_CALLOC(nmemb, size) debug_calloc(nmemb, size, __FILE__, __LINE__)
#define DEBUG_REALLOC(ptr, size) debug_realloc(ptr, size, __FILE__, __LINE__)
#define DEBUG_FREE(ptr) debug_free(ptr, __FILE__, __LINE__)

#endif

/* memory_debugger.c */
#include "memory_debugger.h"

/* 全局变量 */
memory_debug_config_t g_mem_debug_config = {
    .enable_bounds_checking = true,
    .enable_pattern_fill = true,
    .enable_leak_detection = true,
    .enable_thread_safety = true,
    .alignment = 16,
    .fill_pattern = 0xAA,
    .free_pattern = 0xDD
};

memory_block_header_t* g_allocated_blocks = NULL;
pthread_mutex_t g_memory_mutex = PTHREAD_MUTEX_INITIALIZER;

void memory_debug_init(const memory_debug_config_t* config) {
    if (config) {
        memcpy(&g_mem_debug_config, config, sizeof(memory_debug_config_t));
    }

    atexit(memory_debug_report);
    printf("Memory debugger initialized with bounds checking: %s\n",
           g_mem_debug_config.enable_bounds_checking ? "enabled" : "disabled");
}

void* debug_malloc(size_t size, const char* file, int line) {
    if (g_mem_debug_config.enable_thread_safety) {
        pthread_mutex_lock(&g_memory_mutex);
    }

    // 分配额外的空间用于头部信息和对齐
    size_t total_size = sizeof(memory_block_header_t) + size + g_mem_debug_config.alignment;
    size_t aligned_size = (total_size + g_mem_debug_config.alignment - 1) &
                         ~(g_mem_debug_config.alignment - 1);

    void* block = malloc(aligned_size);
    if (!block) {
        if (g_mem_debug_config.enable_thread_safety) {
            pthread_mutex_unlock(&g_memory_mutex);
        }
        return NULL;
    }

    // 计算用户数据地址
    memory_block_header_t* header = (memory_block_header_t*)block;
    void* user_ptr = (void*)((uintptr_t)block + sizeof(memory_block_header_t));

    // 填充头部信息
    header->size = size;
    header->file = file;
    header->line = line;
    header->next = g_allocated_blocks;
    header->prev = NULL;
    header->thread_id = pthread_self();
    header->timestamp = time(NULL);

    // 设置canary值
    header->canary[0] = 0xDE;
    header->canary[1] = 0xAD;
    header->canary[2] = 0xBE;
    header->canary[3] = 0xEF;

    // 更新链表
    if (g_allocated_blocks) {
        g_allocated_blocks->prev = header;
    }
    g_allocated_blocks = header;

    // 填充模式
    if (g_mem_debug_config.enable_pattern_fill) {
        memset(user_ptr, g_mem_debug_config.fill_pattern, size);
    }

    if (g_mem_debug_config.enable_thread_safety) {
        pthread_mutex_unlock(&g_memory_mutex);
    }

    printf("DEBUG MALLOC: %zu bytes at %p (%s:%d)\n", size, user_ptr, file, line);
    return user_ptr;
}

void debug_free(void* ptr, const char* file, int line) {
    if (!ptr) {
        printf("DEBUG FREE: Attempt to free NULL pointer (%s:%d)\n", file, line);
        return;
    }

    if (g_mem_debug_config.enable_thread_safety) {
        pthread_mutex_lock(&g_memory_mutex);
    }

    // 获取头部信息
    memory_block_header_t* header = (memory_block_header_t*)((uintptr_t)ptr - sizeof(memory_block_header_t));

    // 验证canary值
    if (header->canary[0] != 0xDE || header->canary[1] != 0xAD ||
        header->canary[2] != 0xBE || header->canary[3] != 0xEF) {
        printf("DEBUG FREE: Memory corruption detected at %p (%s:%d)\n", ptr, file, line);
        // 继续释放以避免进一步损坏
    }

    // 检查边界
    if (g_mem_debug_config.enable_bounds_checking) {
        memory_debug_check_bounds_for_block(header);
    }

    // 填充释放模式
    if (g_mem_debug_config.enable_pattern_fill) {
        memset(ptr, g_mem_debug_config.free_pattern, header->size);
    }

    // 从链表中移除
    if (header->prev) {
        header->prev->next = header->next;
    } else {
        g_allocated_blocks = header->next;
    }
    if (header->next) {
        header->next->prev = header->prev;
    }

    printf("DEBUG FREE: %zu bytes at %p (%s:%d)\n", header->size, ptr, file, line);

    // 释放内存
    free(header);

    if (g_mem_debug_config.enable_thread_safety) {
        pthread_mutex_unlock(&g_memory_mutex);
    }
}

void memory_debug_check_bounds(void) {
    if (g_mem_debug_config.enable_thread_safety) {
        pthread_mutex_lock(&g_memory_mutex);
    }

    memory_block_header_t* current = g_allocated_blocks;
    while (current) {
        memory_debug_check_bounds_for_block(current);
        current = current->next;
    }

    if (g_mem_debug_config.enable_thread_safety) {
        pthread_mutex_unlock(&g_memory_mutex);
    }
}

void memory_debug_check_bounds_for_block(memory_block_header_t* header) {
    void* user_ptr = (void*)((uintptr_t)header + sizeof(memory_block_header_t));
    void* end_ptr = (void*)((uintptr_t)user_ptr + header->size);

    // 检查块后的区域是否被修改
    size_t check_size = g_mem_debug_config.alignment;
    unsigned char* check_area = (unsigned char*)end_ptr;

    for (size_t i = 0; i < check_size; i++) {
        if (check_area[i] != g_mem_debug_config.fill_pattern) {
            printf("BOUNDS CHECK: Buffer overflow detected after block %p\n", user_ptr);
            printf("  Expected: 0x%02X, Got: 0x%02X at offset %zu\n",
                   g_mem_debug_config.fill_pattern, check_area[i], i);
            break;
        }
    }
}

void memory_debug_report(void) {
    if (!g_mem_debug_config.enable_leak_detection) {
        return;
    }

    if (g_mem_debug_config.enable_thread_safety) {
        pthread_mutex_lock(&g_memory_mutex);
    }

    printf("\n=== MEMORY DEBUG REPORT ===\n");

    size_t total_blocks = 0;
    size_t total_size = 0;
    memory_block_header_t* current = g_allocated_blocks;

    while (current) {
        total_blocks++;
        total_size += current->size;
        current = current->next;
    }

    if (total_blocks == 0) {
        printf("No memory leaks detected.\n");
    } else {
        printf("Memory leaks detected:\n");
        printf("  Total blocks: %zu\n", total_blocks);
        printf("  Total size: %zu bytes\n", total_size);
        printf("\nDetailed leak information:\n");

        current = g_allocated_blocks;
        while (current) {
            printf("  Block %p: %zu bytes allocated at %s:%d\n",
                   (void*)((uintptr_t)current + sizeof(memory_block_header_t)),
                   current->size, current->file, current->line);
            printf("    Thread: %lu, Time: %s",
                   current->thread_id, ctime(&current->timestamp));
            current = current->next;
        }
    }

    printf("============================\n");

    if (g_mem_debug_config.enable_thread_safety) {
        pthread_mutex_unlock(&g_memory_mutex);
    }
}
```

## 4. 性能分析与优化

### 4.1 性能计数器与分析

```c
/* performance_profiler.h */
#ifndef PERFORMANCE_PROFILER_H
#define PERFORMANCE_PROFILER_H

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <sys/time.h>
#include <pthread.h>
#include <stdbool.h>

/* 性能计数器类型 */
typedef enum {
    COUNTER_CYCLES,
    COUNTER_INSTRUCTIONS,
    COUNTER_CACHE_MISSES,
    COUNTER_BRANCH_MISSES,
    COUNTER_L1_CACHE_HITS,
    COUNTER_L2_CACHE_HITS,
    COUNTER_L3_CACHE_HITS,
    COUNTER_TLB_MISSES,
    COUNTER_PAGE_FAULTS,
    COUNTER_CONTEXT_SWITCHES,
    COUNTER_MAX
} perf_counter_type_t;

/* 性能事件 */
typedef struct {
    const char* name;
    perf_counter_type_t type;
    unsigned long long value;
    bool enabled;
} perf_event_t;

/* 函数性能统计 */
typedef struct function_stats {
    const char* name;
    unsigned long call_count;
    unsigned long long total_time;
    unsigned long long min_time;
    unsigned long long max_time;
    struct function_stats* next;
} function_stats_t;

/* 性能分析器配置 */
typedef struct {
    bool enable_sampling;
    bool enable_tracing;
    int sampling_frequency;
    int max_stack_depth;
    bool enable_counters;
} profiler_config_t;

/* 性能分析器 */
typedef struct {
    profiler_config_t config;
    function_stats_t* function_stats;
    perf_event_t* counters;
    pthread_t sampling_thread;
    bool running;
    FILE* output_file;
} performance_profiler_t;

/* 全局分析器实例 */
extern performance_profiler_t* g_profiler;

/* 性能分析函数 */
performance_profiler_t* performance_profiler_create(const profiler_config_t* config);
void performance_profiler_destroy(performance_profiler_t* profiler);
void performance_profiler_start(performance_profiler_t* profiler);
void performance_profiler_stop(performance_profiler_t* profiler);
void performance_profiler_function_enter(performance_profiler_t* profiler,
                                        const char* function_name);
void performance_profiler_function_exit(performance_profiler_t* profiler,
                                       const char* function_name);
void performance_profiler_dump_stats(performance_profiler_t* profiler);
void performance_profiler_reset(performance_profiler_t* profiler);

/* 性能计数器函数 */
unsigned long long read_counter(perf_counter_type_t type);
void enable_counter(perf_counter_type_t type);
void disable_counter(perf_counter_type_t type);
void reset_all_counters(void);

/* 便利宏 */
#define PROFILE_FUNCTION(profiler) \
    performance_profiler_function_enter(profiler, __func__); \
    profiler_scope_t scope = {profiler, __func__}

#define PROFILE_SCOPE(profiler, name) \
    performance_profiler_function_enter(profiler, name); \
    profiler_scope_t scope = {profiler, name}

#define PROFILE_COUNT(profiler, counter) \
    do { \
        unsigned long long value = read_counter(counter); \
        printf("%s: %llu\n", #counter, value); \
    } while (0)

/* 作用域对象 */
typedef struct {
    performance_profiler_t* profiler;
    const char* name;
} profiler_scope_t;

static inline void profiler_scope_exit(profiler_scope_t* scope) {
    performance_profiler_function_exit(scope->profiler, scope->name);
}

#define SCOPE_EXIT(var) __attribute__((cleanup(profiler_scope_exit))) profiler_scope_t var

#endif

/* performance_profiler.c */
#include "performance_profiler.h"

/* 时间戳函数 */
static inline unsigned long long get_timestamp(void) {
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return (unsigned long long)ts.tv_sec * 1000000000ULL + ts.tv_nsec;
}

/* 性能计数器读取 */
#if defined(__x86_64__)
static inline unsigned long long read_rdtsc(void) {
    unsigned int lo, hi;
    __asm__ __volatile__ ("rdtsc" : "=a" (lo), "=d" (hi));
    return ((unsigned long long)hi << 32) | lo;
}

static inline unsigned long long read_pmc(int counter) {
    unsigned int lo, hi;
    __asm__ __volatile__ ("rdpmc" : "=a" (lo), "=d" (hi) : "c" (counter));
    return ((unsigned long long)hi << 32) | lo;
}
#elif defined(__aarch64__)
static inline unsigned long long read_cntvct(void) {
    unsigned long long val;
    __asm__ __volatile__ ("mrs %0, cntvct_el0" : "=r" (val));
    return val;
}
#endif

/* 创建性能分析器 */
performance_profiler_t* performance_profiler_create(const profiler_config_t* config) {
    performance_profiler_t* profiler = calloc(1, sizeof(performance_profiler_t));
    if (!profiler) return NULL;

    memcpy(&profiler->config, config, sizeof(profiler_config_t));

    // 初始化性能计数器
    profiler->counters = calloc(COUNTER_MAX, sizeof(perf_event_t));
    if (!profiler->counters) {
        free(profiler);
        return NULL;
    }

    // 初始化计数器名称
    profiler->counters[COUNTER_CYCLES].name = "cycles";
    profiler->counters[COUNTER_INSTRUCTIONS].name = "instructions";
    profiler->counters[COUNTER_CACHE_MISSES].name = "cache-misses";
    profiler->counters[COUNTER_BRANCH_MISSES].name = "branch-misses";
    profiler->counters[COUNTER_L1_CACHE_HITS].name = "L1-cache-hits";
    profiler->counters[COUNTER_L2_CACHE_HITS].name = "L2-cache-hits";
    profiler->counters[COUNTER_L3_CACHE_HITS].name = "L3-cache-hits";
    profiler->counters[COUNTER_TLB_MISSES].name = "tlb-misses";
    profiler->counters[COUNTER_PAGE_FAULTS].name = "page-faults";
    profiler->counters[COUNTER_CONTEXT_SWITCHES].name = "context-switches";

    // 设置类型
    for (int i = 0; i < COUNTER_MAX; i++) {
        profiler->counters[i].type = i;
        profiler->counters[i].enabled = false;
    }

    // 打开输出文件
    profiler->output_file = fopen("profile_output.txt", "w");
    if (!profiler->output_file) {
        perror("Failed to open profile output file");
        free(profiler->counters);
        free(profiler);
        return NULL;
    }

    return profiler;
}

/* 销毁性能分析器 */
void performance_profiler_destroy(performance_profiler_t* profiler) {
    if (!profiler) return;

    // 停止分析器
    if (profiler->running) {
        performance_profiler_stop(profiler);
    }

    // 释放函数统计
    function_stats_t* current = profiler->function_stats;
    while (current) {
        function_stats_t* next = current->next;
        free((void*)current->name);
        free(current);
        current = next;
    }

    // 关闭输出文件
    if (profiler->output_file) {
        fclose(profiler->output_file);
    }

    // 释放计数器
    free(profiler->counters);
    free(profiler);
}

/* 开始性能分析 */
void performance_profiler_start(performance_profiler_t* profiler) {
    if (!profiler || profiler->running) return;

    profiler->running = true;

    // 启用性能计数器
    if (profiler->config.enable_counters) {
        enable_counter(COUNTER_CYCLES);
        enable_counter(COUNTER_INSTRUCTIONS);
        enable_counter(COUNTER_CACHE_MISSES);
        enable_counter(COUNTER_BRANCH_MISSES);
    }

    // 启动采样线程
    if (profiler->config.enable_sampling) {
        pthread_create(&profiler->sampling_thread, NULL,
                      sampling_thread_func, profiler);
    }

    printf("Performance profiler started\n");
}

/* 停止性能分析 */
void performance_profiler_stop(performance_profiler_t* profiler) {
    if (!profiler || !profiler->running) return;

    profiler->running = false;

    // 停止采样线程
    if (profiler->config.enable_sampling) {
        pthread_join(profiler->sampling_thread, NULL);
    }

    // 禁用性能计数器
    if (profiler->config.enable_counters) {
        disable_counter(COUNTER_CYCLES);
        disable_counter(COUNTER_INSTRUCTIONS);
        disable_counter(COUNTER_CACHE_MISSES);
        disable_counter(COUNTER_BRANCH_MISSES);
    }

    printf("Performance profiler stopped\n");
}

/* 函数进入 */
void performance_profiler_function_enter(performance_profiler_t* profiler,
                                        const char* function_name) {
    if (!profiler || !profiler->config.enable_tracing) return;

    // 查找或创建函数统计
    function_stats_t* stats = NULL;
    function_stats_t* current = profiler->function_stats;

    while (current) {
        if (strcmp(current->name, function_name) == 0) {
            stats = current;
            break;
        }
        current = current->next;
    }

    if (!stats) {
        stats = calloc(1, sizeof(function_stats_t));
        stats->name = strdup(function_name);
        stats->min_time = ULLONG_MAX;
        stats->next = profiler->function_stats;
        profiler->function_stats = stats;
    }

    stats->call_count++;
    stats->total_time += get_timestamp();  // 这里应该记录开始时间
}

/* 函数退出 */
void performance_profiler_function_exit(performance_profiler_t* profiler,
                                       const char* function_name) {
    if (!profiler || !profiler->config.enable_tracing) return;

    // 更新函数统计
    function_stats_t* current = profiler->function_stats;
    while (current) {
        if (strcmp(current->name, function_name) == 0) {
            unsigned long long elapsed = get_timestamp() - current->total_time;
            current->total_time += elapsed;

            if (elapsed < current->min_time) {
                current->min_time = elapsed;
            }
            if (elapsed > current->max_time) {
                current->max_time = elapsed;
            }
            break;
        }
        current = current->next;
    }
}

/* 转储统计信息 */
void performance_profiler_dump_stats(performance_profiler_t* profiler) {
    if (!profiler) return;

    fprintf(profiler->output_file, "=== PERFORMANCE PROFILE ===\n");
    fprintf(profiler->output_file, "Generated at: %s", ctime(&(time_t){time(NULL)}));
    fprintf(profiler->output_file, "\n");

    // 函数调用统计
    if (profiler->config.enable_tracing) {
        fprintf(profiler->output_file, "FUNCTION CALL STATISTICS:\n");
        fprintf(profiler->output_file, "%-30s %-10s %-15s %-15s %-15s\n",
                "Function", "Calls", "Total Time", "Avg Time", "Max Time");
        fprintf(profiler->output_file, "%-30s %-10s %-15s %-15s %-15s\n",
                "--------", "-----", "----------", "--------", "--------");

        function_stats_t* current = profiler->function_stats;
        while (current) {
            double avg_time = current->call_count > 0 ?
                            (double)current->total_time / current->call_count : 0;

            fprintf(profiler->output_file, "%-30s %-10lu %-15.3f %-15.3f %-15.3f\n",
                    current->name, current->call_count,
                    current->total_time / 1000000.0,  // 转换为毫秒
                    avg_time / 1000000.0,
                    current->max_time / 1000000.0);

            current = current->next;
        }
        fprintf(profiler->output_file, "\n");
    }

    // 性能计数器
    if (profiler->config.enable_counters) {
        fprintf(profiler->output_file, "PERFORMANCE COUNTERS:\n");
        fprintf(profiler->output_file, "%-20s %-15s\n", "Counter", "Value");
        fprintf(profiler->output_file, "%-20s %-15s\n", "-------", "-----");

        for (int i = 0; i < COUNTER_MAX; i++) {
            if (profiler->counters[i].enabled) {
                fprintf(profiler->output_file, "%-20s %-15llu\n",
                        profiler->counters[i].name,
                        profiler->counters[i].value);
            }
        }
        fprintf(profiler->output_file, "\n");
    }

    fflush(profiler->output_file);
}

/* 采样线程函数 */
void* sampling_thread_func(void* arg) {
    performance_profiler_t* profiler = (performance_profiler_t*)arg;

    while (profiler->running) {
        usleep(1000000 / profiler->config.sampling_frequency);

        // 采集性能计数器数据
        if (profiler->config.enable_counters) {
            for (int i = 0; i < COUNTER_MAX; i++) {
                if (profiler->counters[i].enabled) {
                    profiler->counters[i].value += read_counter(i);
                }
            }
        }
    }

    return NULL;
}
```

### 4.2 缓存分析器

```c
/* cache_analyzer.h */
#ifndef CACHE_ANALYZER_H
#define CACHE_ANALYZER_H

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>
#include <stdbool.h>
#include <pthread.h>

/* 缓存配置 */
typedef struct {
    size_t l1_size;
    size_t l1_line_size;
    size_t l1_associativity;
    size_t l2_size;
    size_t l2_line_size;
    size_t l2_associativity;
    size_t l3_size;
    size_t l3_line_size;
    size_t l3_associativity;
} cache_config_t;

/* 缓存访问统计 */
typedef struct {
    unsigned long long l1_hits;
    unsigned long long l1_misses;
    unsigned long long l2_hits;
    unsigned long long l2_misses;
    unsigned long long l3_hits;
    unsigned long long l3_misses;
    unsigned long long total_accesses;
} cache_stats_t;

/* 内存访问模式 */
typedef enum {
    ACCESS_SEQUENTIAL,
    ACCESS_RANDOM,
    ACCESS_STRIDED,
    ACCESS_REVERSE,
    ACCESS_PATTERN_MAX
} access_pattern_t;

/* 缓存分析器 */
typedef struct {
    cache_config_t config;
    cache_stats_t stats;
    bool enabled;
    pthread_mutex_t stats_mutex;
} cache_analyzer_t;

/* 缓存分析函数 */
cache_analyzer_t* cache_analyzer_create(void);
void cache_analyzer_destroy(cache_analyzer_t* analyzer);
void cache_analyzer_enable(cache_analyzer_t* analyzer);
void cache_analyzer_disable(cache_analyzer_t* analyzer);
void cache_analyzer_reset(cache_analyzer_t* analyzer);
void cache_analyzer_record_access(cache_analyzer_t* analyzer, void* addr, size_t size);
void cache_analyzer_print_stats(cache_analyzer_t* analyzer);
double cache_analyzer_get_hit_ratio(cache_analyzer_t* analyzer, int level);

/* 缓存测试函数 */
void cache_benchmark_sequential(cache_analyzer_t* analyzer, size_t size, size_t iterations);
void cache_benchmark_random(cache_analyzer_t* analyzer, size_t size, size_t iterations);
void cache_benchmark_strided(cache_analyzer_t* analyzer, size_t size, size_t stride, size_t iterations);
void cache_benchmark_matrix_multiply(cache_analyzer_t* analyzer, size_t matrix_size);

/* 缓存友好操作 */
void* cache_aligned_alloc(size_t size, size_t alignment);
void cache_prefetch(void* addr);
void cache_flush(void* addr, size_t size);
void cache_invalidate(void* addr, size_t size);

/* 缓存优化建议 */
typedef struct {
    char* description;
    float expected_improvement;
    int difficulty;
} optimization_suggestion_t;

optimization_suggestion_t* cache_analyzer_get_suggestions(cache_analyzer_t* analyzer);
void optimization_suggestions_free(optimization_suggestion_t* suggestions);

#endif

/* cache_analyzer.c */
#include "cache_analyzer.h"

/* 获取缓存配置 */
static void get_cache_config(cache_config_t* config) {
#if defined(__linux__)
    FILE* fp = fopen("/sys/devices/system/cpu/cpu0/cache/index0/size", "r");
    if (fp) {
        char size_str[32];
        fgets(size_str, sizeof(size_str), fp);
        // 解析缓存大小
        fclose(fp);
    }
#elif defined(__APPLE__)
    size_t size = sizeof(size_t);
    sysctlbyname("hw.cachesize", &config->l1_size, &size, NULL, 0);
#endif

    // 默认值
    config->l1_size = 32 * 1024;      // 32KB
    config->l1_line_size = 64;         // 64 bytes
    config->l1_associativity = 8;      // 8-way
    config->l2_size = 256 * 1024;     // 256KB
    config->l2_line_size = 64;         // 64 bytes
    config->l2_associativity = 4;      // 4-way
    config->l3_size = 8 * 1024 * 1024; // 8MB
    config->l3_line_size = 64;         // 64 bytes
    config->l3_associativity = 16;     // 16-way
}

/* 创建缓存分析器 */
cache_analyzer_t* cache_analyzer_create(void) {
    cache_analyzer_t* analyzer = calloc(1, sizeof(cache_analyzer_t));
    if (!analyzer) return NULL;

    get_cache_config(&analyzer->config);
    pthread_mutex_init(&analyzer->stats_mutex, NULL);

    return analyzer;
}

/* 销毁缓存分析器 */
void cache_analyzer_destroy(cache_analyzer_t* analyzer) {
    if (!analyzer) return;

    pthread_mutex_destroy(&analyzer->stats_mutex);
    free(analyzer);
}

/* 启用缓存分析 */
void cache_analyzer_enable(cache_analyzer_t* analyzer) {
    if (!analyzer) return;

    pthread_mutex_lock(&analyzer->stats_mutex);
    analyzer->enabled = true;
    pthread_mutex_unlock(&analyzer->stats_mutex);
}

/* 禁用缓存分析 */
void cache_analyzer_disable(cache_analyzer_t* analyzer) {
    if (!analyzer) return;

    pthread_mutex_lock(&analyzer->stats_mutex);
    analyzer->enabled = false;
    pthread_mutex_unlock(&analyzer->stats_mutex);
}

/* 记录缓存访问 */
void cache_analyzer_record_access(cache_analyzer_t* analyzer, void* addr, size_t size) {
    if (!analyzer || !analyzer->enabled) return;

    pthread_mutex_lock(&analyzer->stats_mutex);

    // 简化的缓存模拟
    // 实际实现需要更复杂的缓存模拟算法
    analyzer->stats.total_accesses++;

    // 基于地址的简单哈希来模拟缓存命中/未命中
    uintptr_t addr_int = (uintptr_t)addr;
    int cache_line = addr_int / analyzer->config.l1_line_size;

    // 简单的LRU模拟
    static int lru_cache[1024] = {0};
    static int lru_counter = 0;

    if (lru_cache[cache_line % 1024] != lru_counter) {
        analyzer->stats.l1_misses++;
        lru_cache[cache_line % 1024] = lru_counter;
    } else {
        analyzer->stats.l1_hits++;
    }

    lru_counter++;

    pthread_mutex_unlock(&analyzer->stats_mutex);
}

/* 获取缓存命中率 */
double cache_analyzer_get_hit_ratio(cache_analyzer_t* analyzer, int level) {
    if (!analyzer) return 0.0;

    pthread_mutex_lock(&analyzer->stats_mutex);

    double hit_ratio = 0.0;
    unsigned long long hits = 0, misses = 0;

    switch (level) {
        case 1:
            hits = analyzer->stats.l1_hits;
            misses = analyzer->stats.l1_misses;
            break;
        case 2:
            hits = analyzer->stats.l2_hits;
            misses = analyzer->stats.l2_misses;
            break;
        case 3:
            hits = analyzer->stats.l3_hits;
            misses = analyzer->stats.l3_misses;
            break;
    }

    if (hits + misses > 0) {
        hit_ratio = (double)hits / (hits + misses);
    }

    pthread_mutex_unlock(&analyzer->stats_mutex);
    return hit_ratio;
}

/* 缓存对齐分配 */
void* cache_aligned_alloc(size_t size, size_t alignment) {
    void* ptr = NULL;
    if (posix_memalign(&ptr, alignment, size) != 0) {
        return NULL;
    }
    return ptr;
}

/* 缓存预取 */
void cache_prefetch(void* addr) {
#if defined(__GNUC__)
    __builtin_prefetch(addr, 0, 3);  // 预取到所有缓存级别
#endif
}

/* 缓存刷新 */
void cache_flush(void* addr, size_t size) {
#if defined(__x86_64__)
    // 使用CLFLUSH指令
    uintptr_t start = (uintptr_t)addr;
    uintptr_t end = start + size;

    for (uintptr_t p = start; p < end; p += 64) {
        __asm__ __volatile__ ("clflush (%0)" : : "r" (p));
    }
#endif
}

/* 缓存失效 */
void cache_invalidate(void* addr, size_t size) {
#if defined(__x86_64__)
    // 使用CLFLUSHOPT指令
    uintptr_t start = (uintptr_t)addr;
    uintptr_t end = start + size;

    for (uintptr_t p = start; p < end; p += 64) {
        __asm__ __volatile__ ("clflushopt (%0)" : : "r" (p));
    }
#endif
}

/* 顺序访问测试 */
void cache_benchmark_sequential(cache_analyzer_t* analyzer, size_t size, size_t iterations) {
    if (!analyzer) return;

    size_t* data = malloc(size);
    if (!data) return;

    // 初始化数据
    for (size_t i = 0; i < size / sizeof(size_t); i++) {
        data[i] = i;
    }

    // 顺序访问测试
    cache_analyzer_enable(analyzer);
    cache_analyzer_reset(analyzer);

    size_t sum = 0;
    for (size_t iter = 0; iter < iterations; iter++) {
        for (size_t i = 0; i < size / sizeof(size_t); i++) {
            sum += data[i];
            cache_analyzer_record_access(analyzer, &data[i], sizeof(size_t));
        }
    }

    cache_analyzer_disable(analyzer);

    printf("Sequential access benchmark:\n");
    printf("  Size: %zu bytes, Iterations: %zu\n", size, iterations);
    printf("  L1 hit ratio: %.2f%%\n", cache_analyzer_get_hit_ratio(analyzer, 1) * 100);
    printf("  Sum: %zu\n", sum);

    free(data);
}

/* 随机访问测试 */
void cache_benchmark_random(cache_analyzer_t* analyzer, size_t size, size_t iterations) {
    if (!analyzer) return;

    size_t* data = malloc(size);
    size_t* indices = malloc(iterations * sizeof(size_t));
    if (!data || !indices) {
        free(data);
        free(indices);
        return;
    }

    // 初始化数据和随机索引
    for (size_t i = 0; i < size / sizeof(size_t); i++) {
        data[i] = i;
    }

    for (size_t i = 0; i < iterations; i++) {
        indices[i] = rand() % (size / sizeof(size_t));
    }

    // 随机访问测试
    cache_analyzer_enable(analyzer);
    cache_analyzer_reset(analyzer);

    size_t sum = 0;
    for (size_t iter = 0; iter < iterations; iter++) {
        size_t idx = indices[iter];
        sum += data[idx];
        cache_analyzer_record_access(analyzer, &data[idx], sizeof(size_t));
    }

    cache_analyzer_disable(analyzer);

    printf("Random access benchmark:\n");
    printf("  Size: %zu bytes, Iterations: %zu\n", size, iterations);
    printf("  L1 hit ratio: %.2f%%\n", cache_analyzer_get_hit_ratio(analyzer, 1) * 100);
    printf("  Sum: %zu\n", sum);

    free(data);
    free(indices);
}

/* 矩阵乘法测试 */
void cache_benchmark_matrix_multiply(cache_analyzer_t* analyzer, size_t matrix_size) {
    if (!analyzer) return;

    size_t size = matrix_size * matrix_size;
    double* a = cache_aligned_alloc(size * sizeof(double), 64);
    double* b = cache_aligned_alloc(size * sizeof(double), 64);
    double* c = cache_aligned_alloc(size * sizeof(double), 64);

    if (!a || !b || !c) {
        free(a);
        free(b);
        free(c);
        return;
    }

    // 初始化矩阵
    for (size_t i = 0; i < size; i++) {
        a[i] = (double)rand() / RAND_MAX;
        b[i] = (double)rand() / RAND_MAX;
        c[i] = 0.0;
    }

    // 矩阵乘法
    cache_analyzer_enable(analyzer);
    cache_analyzer_reset(analyzer);

    for (size_t i = 0; i < matrix_size; i++) {
        for (size_t j = 0; j < matrix_size; j++) {
            double sum = 0.0;
            for (size_t k = 0; k < matrix_size; k++) {
                sum += a[i * matrix_size + k] * b[k * matrix_size + j];
                cache_analyzer_record_access(analyzer, &a[i * matrix_size + k], sizeof(double));
                cache_analyzer_record_access(analyzer, &b[k * matrix_size + j], sizeof(double));
            }
            c[i * matrix_size + j] = sum;
        }
    }

    cache_analyzer_disable(analyzer);

    printf("Matrix multiplication benchmark:\n");
    printf("  Matrix size: %zu x %zu\n", matrix_size, matrix_size);
    printf("  L1 hit ratio: %.2f%%\n", cache_analyzer_get_hit_ratio(analyzer, 1) * 100);
    printf("  L2 hit ratio: %.2f%%\n", cache_analyzer_get_hit_ratio(analyzer, 2) * 100);

    free(a);
    free(b);
    free(c);
}

/* 打印缓存统计 */
void cache_analyzer_print_stats(cache_analyzer_t* analyzer) {
    if (!analyzer) return;

    pthread_mutex_lock(&analyzer->stats_mutex);

    printf("Cache Analysis Statistics:\n");
    printf("========================\n");
    printf("Total accesses: %llu\n", analyzer->stats.total_accesses);
    printf("L1 hits: %llu\n", analyzer->stats.l1_hits);
    printf("L1 misses: %llu\n", analyzer->stats.l1_misses);
    printf("L2 hits: %llu\n", analyzer->stats.l2_hits);
    printf("L2 misses: %llu\n", analyzer->stats.l2_misses);
    printf("L3 hits: %llu\n", analyzer->stats.l3_hits);
    printf("L3 misses: %llu\n", analyzer->stats.l3_misses);

    if (analyzer->stats.total_accesses > 0) {
        printf("\nHit Ratios:\n");
        printf("L1: %.2f%%\n", (double)analyzer->stats.l1_hits / analyzer->stats.total_accesses * 100);
        printf("L2: %.2f%%\n", (double)analyzer->stats.l2_hits / analyzer->stats.total_accesses * 100);
        printf("L3: %.2f%%\n", (double)analyzer->stats.l3_hits / analyzer->stats.total_accesses * 100);
    }

    pthread_mutex_unlock(&analyzer->stats_mutex);
}

/* 获取优化建议 */
optimization_suggestion_t* cache_analyzer_get_suggestions(cache_analyzer_t* analyzer) {
    if (!analyzer) return NULL;

    optimization_suggestion_t* suggestions = calloc(5, sizeof(optimization_suggestion_t));
    if (!suggestions) return NULL;

    pthread_mutex_lock(&analyzer->stats_mutex);

    double l1_ratio = analyzer->stats.total_accesses > 0 ?
                     (double)analyzer->stats.l1_hits / analyzer->stats.total_accesses : 0;
    double l2_ratio = analyzer->stats.total_accesses > 0 ?
                     (double)analyzer->stats.l2_hits / analyzer->stats.total_accesses : 0;

    int suggestion_count = 0;

    if (l1_ratio < 0.8) {
        suggestions[suggestion_count].description = strdup("Low L1 cache hit ratio. Consider data structure reorganization.");
        suggestions[suggestion_count].expected_improvement = 15.0f;
        suggestions[suggestion_count].difficulty = 3;
        suggestion_count++;
    }

    if (l2_ratio < 0.6) {
        suggestions[suggestion_count].description = strdup("Low L2 cache hit ratio. Consider blocking/tiling algorithms.");
        suggestions[suggestion_count].expected_improvement = 25.0f;
        suggestions[suggestion_count].difficulty = 4;
        suggestion_count++;
    }

    if (analyzer->stats.total_accesses > 1000000) {
        suggestions[suggestion_count].description = strdup("High access count. Consider loop optimization and vectorization.");
        suggestions[suggestion_count].expected_improvement = 20.0f;
        suggestions[suggestion_count].difficulty = 2;
        suggestion_count++;
    }

    pthread_mutex_unlock(&analyzer->stats_mutex);

    return suggestions;
}

/* 释放优化建议 */
void optimization_suggestions_free(optimization_suggestion_t* suggestions) {
    if (!suggestions) return;

    for (int i = 0; i < 5; i++) {
        if (suggestions[i].description) {
            free(suggestions[i].description);
        }
    }
    free(suggestions);
}
```

## 5. 现代C语言开发工作流

### 5.1 CI/CD集成

```c
/* .github/workflows/ci.yml - GitHub Actions配置 */
name: C Language CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        compiler: [gcc, clang]
        build_type: [Debug, Release]

    steps:
    - uses: actions/checkout@v3

    - name: Install dependencies
      run: |
        if [ "$RUNNER_OS" == "Linux" ]; then
          sudo apt-get update
          sudo apt-get install -y cmake ninja-build valgrind cppcheck
        elif [ "$RUNNER_OS" == "macOS" ]; then
          brew install cmake ninja cppcheck
        fi

    - name: Configure
      run: |
        cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=${{matrix.build_type}} \
              -DCMAKE_C_COMPILER=${{matrix.compiler}}

    - name: Build
      run: cmake --build build

    - name: Run tests
      run: cd build && ctest --output-on-failure

    - name: Run static analysis
      run: |
        cppcheck --enable=all --suppress=missingIncludeSystem \
                 --inline-suppr --error-exitcode=1 src/

    - name: Run memory check (Linux)
      if: runner.os == 'Linux' && matrix.build_type == 'Debug'
      run: |
        valgrind --tool=memcheck --leak-check=full --error-exitcode=1 \
                 ./build/your_program

  coverage:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3

    - name: Install dependencies
      run: |
        sudo apt-get update
        sudo apt-get install -y cmake gcovr lcov

    - name: Configure with coverage
      run: |
        cmake -B build -DCMAKE_BUILD_TYPE=Coverage \
              -DCMAKE_C_FLAGS="-coverage"

    - name: Build
      run: cmake --build build

    - name: Run tests
      run: cd build && ctest --output-on-failure

    - name: Generate coverage report
      run: |
        cd build
        gcovr --root ../ --exclude '.*test.*' --exclude '.*_test.c' \
              --xml coverage.xml --html coverage.html

    - name: Upload coverage
      uses: codecov/codecov-action@v3
      with:
        file: ./build/coverage.xml

  benchmark:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3

    - name: Install dependencies
      run: |
        sudo apt-get update
        sudo apt-get install -y cmake google-benchmark

    - name: Configure
      run: |
        cmake -B build -DCMAKE_BUILD_TYPE=Release \
              -DBUILD_BENCHMARKS=ON

    - name: Build
      run: cmake --build build

    - name: Run benchmarks
      run: ./build/benchmarks --benchmark_format=json > benchmark_results.json

    - name: Upload benchmark results
      uses: benchmark-action/github-action-benchmark@v1
      with:
        tool: 'googlecpp'
        output-file-path: benchmark_results.json
```

### 5.2 容器化开发环境

```c
/* Dockerfile - 开发环境容器 */
FROM ubuntu:22.04

# 安装基础工具
RUN apt-get update && apt-get install -y \
    build-essential \
    cmake \
    ninja-build \
    git \
    gdb \
    valgrind \
    cppcheck \
    clang \
    clang-format \
    clang-tidy \
    llvm-dev \
    python3 \
    python3-pip \
    doxygen \
    graphviz \
    && rm -rf /var/lib/apt/lists/*

# 安装Python工具
RUN pip3 install \
    conan \
    meson \
    gcovr

# 安装性能分析工具
RUN apt-get update && apt-get install -y \
    linux-perf \
    linux-tools-generic \
    hwloc \
    libhwloc-dev \
    && rm -rf /var/lib/apt/lists/*

# 设置工作目录
WORKDIR /workspace

# 创建非root用户
RUN useradd -m -s /bin/bash developer
USER developer

# 设置环境变量
ENV CC=clang
ENV CXX=clang++
ENV TERM=xterm-256color

# 复制开发脚本
COPY --chown=developer:developer docker/entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]

/* docker-compose.yml - 开发环境编排 */
version: '3.8'

services:
  c-dev:
    build: .
    volumes:
      - .:/workspace
      - ~/.ssh:/home/developer/.ssh:ro
      - ~/.gitconfig:/home/developer/.gitconfig:ro
    environment:
      - DISPLAY=${DISPLAY}
      - QT_X11_NO_MITSHM=1
    network_mode: host
    stdin_open: true
    tty: true

  c-builder:
    build:
      context: .
      dockerfile: Dockerfile.builder
    volumes:
      - .:/workspace
    command: /bin/bash -c "cmake -B build -G Ninja && cmake --build build"

  c-tester:
    build:
      context: .
      dockerfile: Dockerfile.tester
    volumes:
      - .:/workspace
    command: /bin/bash -c "cd build && ctest --output-on-failure"

  c-linter:
    build:
      context: .
      dockerfile: Dockerfile.linter
    volumes:
      - .:/workspace
    command: /bin/bash -c "cppcheck --enable=all src/"

/* entrypoint.sh - 容器启动脚本 */
#!/bin/bash

# 检查是否是开发环境
if [ "$1" = "dev" ]; then
    echo "Starting C development environment..."
    exec /bin/bash
elif [ "$1" = "build" ]; then
    echo "Building project..."
    cmake -B build -G Ninja
    cmake --build build
elif [ "$1" = "test" ]; then
    echo "Running tests..."
    cd build
    ctest --output-on-failure
elif [ "$1" = "debug" ]; then
    echo "Starting debug session..."
    gdb ./build/your_program
elif [ "$1" = "profile" ]; then
    echo "Running performance profile..."
    perf record -g ./build/your_program
    perf report
elif [ "$1" = "memory" ]; then
    echo "Running memory check..."
    valgrind --tool=memcheck --leak-check=full ./build/your_program
else
    exec "$@"
fi
```

### 5.3 开发工具配置

```c
/* .vscode/settings.json - VS Code配置 */
{
    "C_Cpp.default.configurationProvider": "ms-vscode.cmake-tools",
    "C_Cpp.intelliSenseEngine": "default",
    "C_Cpp.intelliSenseEngineFallback": "enabled",
    "C_Cpp.intelliSenseMode": "linux-gcc-x64",
    "C_Cpp.autocomplete": "default",
    "C_Cpp.enhancedColorization": "enabled",
    "C_Cpp.errorSquiggles": "enabled",
    "C_Cpp.inlayHints.parameterNames.enabled": true,
    "C_Cpp.inlayHints.referenceNames.enabled": true,

    "editor.formatOnSave": true,
    "editor.formatOnPaste": true,
    "editor.formatOnType": true,
    "editor.renderWhitespace": "boundary",
    "editor.renderControlCharacters": true,
    "editor.rulers": [80, 120],

    "files.associations": {
        "*.h": "c",
        "*.c": "c"
    },

    "search.exclude": {
        "**/build": true,
        "**/.git": true,
        "**/node_modules": true
    },

    "files.watcherExclude": {
        "**/build/**": true,
        "**/.git/objects/**": true
    },

    "cmake.configureOnOpen": true,
    "cmake.buildDirectory": "${workspaceFolder}/build",
    "cmake.generator": "Ninja",
    "cmake.configureArgs": [
        "-DCMAKE_BUILD_TYPE=Debug",
        "-DCMAKE_EXPORT_COMPILE_COMMANDS=ON"
    ],

    "cSpell.words": [
        "malloc",
        "calloc",
        "realloc",
        "stddef",
        "stdint",
        "stdbool",
        "pthread",
        "valgrind",
        "cppcheck",
        "clang",
        "gcc",
        "cmake",
        "ninja"
    ],

    "cSpell.enableFiletypes": [
        "c",
        "cpp"
    ]
}

/* .clang-format - 代码格式化配置 */
---
Language: C
BasedOnStyle: LLVM
IndentWidth: 4
TabWidth: 4
UseTab: Never
ColumnLimit: 120
IndentCaseLabels: true
AlignEscapedNewlines: Left
AllowShortIfStatementsOnASingleLine: false
AllowShortLoopsOnASingleLine: false
AlwaysBreakBeforeMultilineStrings: true
BreakBeforeBinaryOperators: None
BreakBeforeTernaryOperators: true
BreakConstructorInitializers: BeforeColon
BreakAfterJavaFieldAnnotations: false
BreakStringLiterals: true
CommentPragmas: '^ IWYU pragma:'
CompactNamespaces: false
ConstructorInitializerAllOnOneLineOrOnePerLine: false
ConstructorInitializerIndentWidth: 4
ContinuationIndentWidth: 4
Cpp11BracedListStyle: true
DerivePointerAlignment: false
DisableFormat: false
ExperimentalAutoDetectBinPacking: false
FixNamespaceComments: true
ForEachMacros: ['foreach', 'Q_FOREACH', 'BOOST_FOREACH']
IncludeCategories:
  - Regex: '^<.*\.h>'
    Priority: 1
  - Regex: '^<.*'
    Priority: 2
  - Regex: '.*'
    Priority: 3
IncludeIsMainRegex: '([-_](test|unittest))?$'
IndentCaseLabels: true
IndentPPDirectives: None
IndentWrappedFunctionNames: false
JavaScriptQuotes: Leave
JavaScriptWrapImports: true
KeepEmptyLinesAtTheStartOfBlocks: true
MacroBlockBegin: ''
MacroBlockEnd: ''
MaxEmptyLinesToKeep: 1
NamespaceIndentation: None
ObjCBlockIndentWidth: 2
ObjCSpaceAfterProperty: false
ObjCSpaceBeforeProtocolList: true
PenaltyBreakAssignment: 2
PenaltyBreakBeforeFirstCallParameter: 1
PenaltyBreakComment: 300
PenaltyBreakFirstLessLess: 120
PenaltyBreakString: 1000
PenaltyExcessCharacter: 1000000
PenaltyReturnTypeOnItsOwnLine: 200
PointerAlignment: Left
ReflowComments: true
SortIncludes: true
SpaceAfterCStyleCast: false
SpaceBeforeAssignmentOperators: true
SpaceBeforeParens: ControlStatements
SpaceInEmptyParentheses: false
SpacesBeforeTrailingComments: 2
SpacesInAngles: false
SpacesInContainerLiterals: false
SpacesInCStyleCastParentheses: false
SpacesInParentheses: false
SpacesInSquareBrackets: false
Standard: C11
TabWidth: 4
UseTab: Never
```

## 6. 高级调试技术

### 6.1 符号化崩溃分析

```c
/* crash_analyzer.h */
#ifndef CRASH_ANALYZER_H
#define CRASH_ANALYZER_H

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <signal.h>
#include <execinfo.h>
#include <unistd.h>
#include <ucontext.h>
#include <dlfcn.h>
#include <fcntl.h>
#include <sys/stat.h>

/* 崩溃信息结构 */
typedef struct {
    int signal_number;
    char* signal_name;
    void* fault_address;
    void* stack_trace[128];
    int stack_depth;
    ucontext_t* context;
    time_t timestamp;
    pid_t pid;
    pid_t tid;
    char* executable_path;
    char* working_directory;
} crash_info_t;

/* 寄存器信息 */
typedef struct {
    unsigned long long rax;
    unsigned long long rbx;
    unsigned long long rcx;
    unsigned long long rdx;
    unsigned long long rsi;
    unsigned long long rdi;
    unsigned long long rbp;
    unsigned long long rsp;
    unsigned long long r8;
    unsigned long long r9;
    unsigned long long r10;
    unsigned long long r11;
    unsigned long long r12;
    unsigned long long r13;
    unsigned long long r14;
    unsigned long long r15;
    unsigned long long rip;
    unsigned long long eflags;
    unsigned long long cs;
    unsigned long long ss;
    unsigned long long ds;
    unsigned long long es;
    unsigned long long fs;
    unsigned long long gs;
} register_info_t;

/* 崩溃分析器 */
typedef struct {
    crash_info_t* crash_info;
    register_info_t* register_info;
    char* crash_log_path;
    bool initialized;
    bool log_to_file;
    bool log_to_stderr;
    bool generate_core_dump;
} crash_analyzer_t;

/* 崩溃分析函数 */
crash_analyzer_t* crash_analyzer_create(const char* log_path, bool log_to_file, bool log_to_stderr);
void crash_analyzer_destroy(crash_analyzer_t* analyzer);
bool crash_analyzer_init(crash_analyzer_t* analyzer);
void crash_analyzer_set_signal_handlers(crash_analyzer_t* analyzer);
void crash_analyzer_generate_report(crash_analyzer_t* analyzer, const crash_info_t* crash_info);
char* crash_analyzer_symbolize_address(crash_analyzer_t* analyzer, void* address);
void crash_analyzer_write_minidump(crash_analyzer_t* analyzer, const crash_info_t* crash_info);
void crash_analyzer_enable_core_dump(void);

/* 便利宏 */
#define CRASH_ANALYZER_INIT(log_path) \
    crash_analyzer_t* __crash_analyzer = crash_analyzer_create(log_path, true, true); \
    if (__crash_analyzer) { \
        crash_analyzer_init(__crash_analyzer); \
        crash_analyzer_set_signal_handlers(__crash_analyzer); \
    }

#define CRASH_ANALYZER_CLEANUP() \
    if (__crash_analyzer) { \
        crash_analyzer_destroy(__crash_analyzer); \
    }

#endif

/* crash_analyzer.c */
#include "crash_analyzer.h"

/* 信号处理函数 */
static void signal_handler(int sig, siginfo_t* info, void* context);
static void crash_analyzer_signal_handler(crash_analyzer_t* analyzer, int sig, siginfo_t* info, void* context);

/* 创建崩溃分析器 */
crash_analyzer_t* crash_analyzer_create(const char* log_path, bool log_to_file, bool log_to_stderr) {
    crash_analyzer_t* analyzer = calloc(1, sizeof(crash_analyzer_t));
    if (!analyzer) return NULL;

    analyzer->crash_info = calloc(1, sizeof(crash_info_t));
    if (!analyzer->crash_info) {
        free(analyzer);
        return NULL;
    }

    analyzer->register_info = calloc(1, sizeof(register_info_t));
    if (!analyzer->register_info) {
        free(analyzer->crash_info);
        free(analyzer);
        return NULL;
    }

    if (log_path) {
        analyzer->crash_log_path = strdup(log_path);
    }

    analyzer->log_to_file = log_to_file;
    analyzer->log_to_stderr = log_to_stderr;
    analyzer->generate_core_dump = true;

    return analyzer;
}

/* 销毁崩溃分析器 */
void crash_analyzer_destroy(crash_analyzer_t* analyzer) {
    if (!analyzer) return;

    if (analyzer->crash_info) {
        free(analyzer->crash_info->signal_name);
        free(analyzer->crash_info->executable_path);
        free(analyzer->crash_info->working_directory);
        free(analyzer->crash_info);
    }

    free(analyzer->register_info);
    free(analyzer->crash_log_path);
    free(analyzer);
}

/* 初始化崩溃分析器 */
bool crash_analyzer_init(crash_analyzer_t* analyzer) {
    if (!analyzer) return false;

    // 获取可执行文件路径
    char exe_path[4096];
    ssize_t len = readlink("/proc/self/exe", exe_path, sizeof(exe_path) - 1);
    if (len > 0) {
        exe_path[len] = '\0';
        analyzer->crash_info->executable_path = strdup(exe_path);
    }

    // 获取工作目录
    char cwd[4096];
    if (getcwd(cwd, sizeof(cwd))) {
        analyzer->crash_info->working_directory = strdup(cwd);
    }

    // 获取进程信息
    analyzer->crash_info->pid = getpid();
    analyzer->crash_info->tid = gettid();

    analyzer->initialized = true;
    return true;
}

/* 设置信号处理器 */
void crash_analyzer_set_signal_handlers(crash_analyzer_t* analyzer) {
    if (!analyzer || !analyzer->initialized) return;

    struct sigaction sa;
    memset(&sa, 0, sizeof(sa));
    sa.sa_sigaction = (void*)signal_handler;
    sa.sa_flags = SA_SIGINFO | SA_RESETHAND;

    // 设置信号处理器
    sigaction(SIGSEGV, &sa, NULL);
    sigaction(SIGBUS, &sa, NULL);
    sigaction(SIGFPE, &sa, NULL);
    sigaction(SIGILL, &sa, NULL);
    sigaction(SIGABRT, &sa, NULL);
    sigaction(SIGSYS, &sa, NULL);
    sigaction(SIGPIPE, &sa, NULL);
}

/* 信号处理函数 */
static void signal_handler(int sig, siginfo_t* info, void* context) {
    static crash_analyzer_t* analyzer = NULL;

    if (!analyzer) {
        analyzer = crash_analyzer_create("crash_report.txt", true, true);
        if (analyzer) {
            crash_analyzer_init(analyzer);
        }
    }

    crash_analyzer_signal_handler(analyzer, sig, info, context);
}

/* 崩溃分析信号处理 */
static void crash_analyzer_signal_handler(crash_analyzer_t* analyzer, int sig, siginfo_t* info, void* context) {
    if (!analyzer) return;

    // 填充崩溃信息
    analyzer->crash_info->signal_number = sig;
    analyzer->crash_info->fault_address = info->si_addr;
    analyzer->crash_info->timestamp = time(NULL);
    analyzer->crash_info->context = (ucontext_t*)context;

    // 获取信号名称
    switch (sig) {
        case SIGSEGV:
            analyzer->crash_info->signal_name = strdup("SIGSEGV (Segmentation Fault)");
            break;
        case SIGBUS:
            analyzer->crash_info->signal_name = strdup("SIGBUS (Bus Error)");
            break;
        case SIGFPE:
            analyzer->crash_info->signal_name = strdup("SIGFPE (Floating Point Exception)");
            break;
        case SIGILL:
            analyzer->crash_info->signal_name = strdup("SIGILL (Illegal Instruction)");
            break;
        case SIGABRT:
            analyzer->crash_info->signal_name = strdup("SIGABRT (Abort)");
            break;
        default:
            analyzer->crash_info->signal_name = strdup("Unknown Signal");
            break;
    }

    // 获取堆栈跟踪
    analyzer->crash_info->stack_depth = backtrace(analyzer->crash_info->stack_trace, 128);

    // 填充寄存器信息
    if (analyzer->crash_info->context) {
        ucontext_t* ctx = analyzer->crash_info->context;
#if defined(__x86_64__)
        analyzer->register_info->rax = ctx->uc_mcontext.gregs[REG_RAX];
        analyzer->register_info->rbx = ctx->uc_mcontext.gregs[REG_RBX];
        analyzer->register_info->rcx = ctx->uc_mcontext.gregs[REG_RCX];
        analyzer->register_info->rdx = ctx->uc_mcontext.gregs[REG_RDX];
        analyzer->register_info->rsi = ctx->uc_mcontext.gregs[REG_RSI];
        analyzer->register_info->rdi = ctx->uc_mcontext.gregs[REG_RDI];
        analyzer->register_info->rbp = ctx->uc_mcontext.gregs[REG_RBP];
        analyzer->register_info->rsp = ctx->uc_mcontext.gregs[REG_RSP];
        analyzer->register_info->r8 = ctx->uc_mcontext.gregs[REG_R8];
        analyzer->register_info->r9 = ctx->uc_mcontext.gregs[REG_R9];
        analyzer->register_info->r10 = ctx->uc_mcontext.gregs[REG_R10];
        analyzer->register_info->r11 = ctx->uc_mcontext.gregs[REG_R11];
        analyzer->register_info->r12 = ctx->uc_mcontext.gregs[REG_R12];
        analyzer->register_info->r13 = ctx->uc_mcontext.gregs[REG_R13];
        analyzer->register_info->r14 = ctx->uc_mcontext.gregs[REG_R14];
        analyzer->register_info->r15 = ctx->uc_mcontext.gregs[REG_R15];
        analyzer->register_info->rip = ctx->uc_mcontext.gregs[REG_RIP];
        analyzer->register_info->eflags = ctx->uc_mcontext.gregs[REG_EFL];
#endif
    }

    // 生成崩溃报告
    crash_analyzer_generate_report(analyzer, analyzer->crash_info);

    // 生成minidump
    if (analyzer->generate_core_dump) {
        crash_analyzer_write_minidump(analyzer, analyzer->crash_info);
    }

    // 退出程序
    _exit(sig);
}

/* 生成崩溃报告 */
void crash_analyzer_generate_report(crash_analyzer_t* analyzer, const crash_info_t* crash_info) {
    if (!analyzer || !crash_info) return;

    FILE* file = NULL;
    if (analyzer->log_to_file && analyzer->crash_log_path) {
        file = fopen(analyzer->crash_log_path, "a");
    }

    // 输出到文件和stderr
    FILE* outputs[2] = {file, analyzer->log_to_stderr ? stderr : NULL};

    for (int i = 0; i < 2; i++) {
        if (!outputs[i]) continue;

        fprintf(outputs[i], "=== CRASH REPORT ===\n");
        fprintf(outputs[i], "Time: %s", ctime(&crash_info->timestamp));
        fprintf(outputs[i], "Signal: %s\n", crash_info->signal_name);
        fprintf(outputs[i], "Process ID: %d\n", crash_info->pid);
        fprintf(outputs[i], "Thread ID: %d\n", crash_info->tid);
        fprintf(outputs[i], "Fault Address: %p\n", crash_info->fault_address);
        fprintf(outputs[i], "Executable: %s\n", crash_info->executable_path);
        fprintf(outputs[i], "Working Directory: %s\n", crash_info->working_directory);
        fprintf(outputs[i], "\n");

        // 寄存器信息
        fprintf(outputs[i], "Register Information:\n");
        fprintf(outputs[i], "RAX: 0x%016llx  RBX: 0x%016llx\n",
                analyzer->register_info->rax, analyzer->register_info->rbx);
        fprintf(outputs[i], "RCX: 0x%016llx  RDX: 0x%016llx\n",
                analyzer->register_info->rcx, analyzer->register_info->rdx);
        fprintf(outputs[i], "RSI: 0x%016llx  RDI: 0x%016llx\n",
                analyzer->register_info->rsi, analyzer->register_info->rdi);
        fprintf(outputs[i], "RBP: 0x%016llx  RSP: 0x%016llx\n",
                analyzer->register_info->rbp, analyzer->register_info->rsp);
        fprintf(outputs[i], "R8:  0x%016llx  R9:  0x%016llx\n",
                analyzer->register_info->r8, analyzer->register_info->r9);
        fprintf(outputs[i], "R10: 0x%016llx  R11: 0x%016llx\n",
                analyzer->register_info->r10, analyzer->register_info->r11);
        fprintf(outputs[i], "R12: 0x%016llx  R13: 0x%016llx\n",
                analyzer->register_info->r12, analyzer->register_info->r13);
        fprintf(outputs[i], "R14: 0x%016llx  R15: 0x%016llx\n",
                analyzer->register_info->r14, analyzer->register_info->r15);
        fprintf(outputs[i], "RIP: 0x%016llx\n", analyzer->register_info->rip);
        fprintf(outputs[i], "\n");

        // 堆栈跟踪
        fprintf(outputs[i], "Stack Trace:\n");
        char** symbols = backtrace_symbols(crash_info->stack_trace, crash_info->stack_depth);
        if (symbols) {
            for (int j = 0; j < crash_info->stack_depth; j++) {
                char* symbolized = crash_analyzer_symbolize_address(analyzer, crash_info->stack_trace[j]);
                if (symbolized) {
                    fprintf(outputs[i], "#%d %p %s\n", j, crash_info->stack_trace[j], symbolized);
                    free(symbolized);
                } else {
                    fprintf(outputs[i], "#%d %p %s\n", j, crash_info->stack_trace[j], symbols[j]);
                }
            }
            free(symbols);
        }
        fprintf(outputs[i], "\n");

        // 内存映射
        fprintf(outputs[i], "Memory Map:\n");
        FILE* maps = fopen("/proc/self/maps", "r");
        if (maps) {
            char line[1024];
            while (fgets(line, sizeof(line), maps)) {
                fprintf(outputs[i], "  %s", line);
            }
            fclose(maps);
        }
        fprintf(outputs[i], "====================\n\n");
    }

    if (file) {
        fclose(file);
    }
}

/* 符号化地址 */
char* crash_analyzer_symbolize_address(crash_analyzer_t* analyzer, void* address) {
    if (!analyzer || !address) return NULL;

    char command[1024];
    snprintf(command, sizeof(command), "addr2line -e %s %p",
             analyzer->crash_info->executable_path, address);

    FILE* pipe = popen(command, "r");
    if (!pipe) return NULL;

    char result[512];
    if (fgets(result, sizeof(result), pipe)) {
        // 去除换行符
        char* newline = strchr(result, '\n');
        if (newline) *newline = '\0';
        pclose(pipe);
        return strdup(result);
    }

    pclose(pipe);
    return NULL;
}

/* 启用核心转储 */
void crash_analyzer_enable_core_dump(void) {
    struct rlimit limit;
    limit.rlim_cur = RLIM_INFINITY;
    limit.rlim_max = RLIM_INFINITY;

    if (setrlimit(RLIMIT_CORE, &limit) != 0) {
        perror("Failed to enable core dump");
    }
}

/* 写入minidump */
void crash_analyzer_write_minidump(crash_analyzer_t* analyzer, const crash_info_t* crash_info) {
    if (!analyzer || !crash_info) return;

    char dump_path[512];
    snprintf(dump_path, sizeof(dump_path), "core.%d.%d", crash_info->pid, (int)crash_info->timestamp);

    FILE* dump = fopen(dump_path, "wb");
    if (!dump) return;

    // 写入minidump头部
    fwrite("MINIDUMP", 8, 1, dump);

    // 写入崩溃信息
    fwrite(crash_info, sizeof(crash_info_t), 1, dump);

    // 写入寄存器信息
    fwrite(analyzer->register_info, sizeof(register_info_t), 1, dump);

    // 写入堆栈跟踪
    fwrite(crash_info->stack_trace, sizeof(void*), crash_info->stack_depth, dump);

    fclose(dump);
}
```

### 6.2 实时性能监控

```c
/* performance_monitor.h */
#ifndef PERFORMANCE_MONITOR_H
#define PERFORMANCE_MONITOR_H

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <pthread.h>
#include <stdbool.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>

/* 性能指标类型 */
typedef enum {
    METRIC_CPU_USAGE,
    METRIC_MEMORY_USAGE,
    METRIC_DISK_IO,
    METRIC_NETWORK_IO,
    METRIC_THREAD_COUNT,
    METRIC_HANDLE_COUNT,
    METRIC_CONTEXT_SWITCHES,
    METRIC_PAGE_FAULTS,
    METRIC_MAX
} metric_type_t;

/* 性能指标 */
typedef struct {
    metric_type_t type;
    char* name;
    char* unit;
    double value;
    double min_value;
    double max_value;
    double avg_value;
    unsigned long sample_count;
    time_t last_update;
} performance_metric_t;

/* 监控配置 */
typedef struct {
    int sampling_interval_ms;
    int max_samples;
    bool enable_real_time_display;
    bool enable_logging;
    bool enable_alerts;
    double cpu_alert_threshold;
    double memory_alert_threshold;
    char* log_file_path;
} monitor_config_t;

/* 性能监控器 */
typedef struct {
    monitor_config_t config;
    performance_metric_t metrics[METRIC_MAX];
    pthread_t monitor_thread;
    bool running;
    FILE* log_file;
    pid_t monitored_pid;
    char* process_name;
    time_t start_time;
    unsigned long total_samples;
} performance_monitor_t;

/* 性能监控函数 */
performance_monitor_t* performance_monitor_create(const monitor_config_t* config);
void performance_monitor_destroy(performance_monitor_t* monitor);
bool performance_monitor_start(performance_monitor_t* monitor);
void performance_monitor_stop(performance_monitor_t* monitor);
void performance_monitor_update_metrics(performance_monitor_t* monitor);
void performance_monitor_display_status(performance_monitor_t* monitor);
void performance_monitor_log_metrics(performance_monitor_t* monitor);
double performance_monitor_get_metric_value(performance_monitor_t* monitor, metric_type_t type);

/* 报警函数 */
typedef void (*alert_callback_t)(metric_type_t type, double value, double threshold);
void performance_monitor_set_alert_callback(performance_monitor_t* monitor, alert_callback_t callback);

/* 导出函数 */
void performance_monitor_export_csv(performance_monitor_t* monitor, const char* filename);
void performance_monitor_export_json(performance_monitor_t* monitor, const char* filename);

/* 便利宏 */
#define MONITOR_CPU_USAGE(monitor) performance_monitor_get_metric_value(monitor, METRIC_CPU_USAGE)
#define MONITOR_MEMORY_USAGE(monitor) performance_monitor_get_metric_value(monitor, METRIC_MEMORY_USAGE)

#endif

/* performance_monitor.c */
#include "performance_monitor.h"

/* 系统信息获取 */
static double get_cpu_usage(void);
static double get_memory_usage(void);
static double get_disk_io(void);
static double get_network_io(void);
static unsigned long get_thread_count(void);
static unsigned long get_handle_count(void);
static unsigned long get_context_switches(void);
static unsigned long get_page_faults(void);

/* 监控线程函数 */
static void* monitor_thread_func(void* arg);

/* 创建性能监控器 */
performance_monitor_t* performance_monitor_create(const monitor_config_t* config) {
    performance_monitor_t* monitor = calloc(1, sizeof(performance_monitor_t));
    if (!monitor) return NULL;

    memcpy(&monitor->config, config, sizeof(monitor_config_t));

    // 初始化指标
    monitor->metrics[METRIC_CPU_USAGE].type = METRIC_CPU_USAGE;
    monitor->metrics[METRIC_CPU_USAGE].name = strdup("CPU Usage");
    monitor->metrics[METRIC_CPU_USAGE].unit = strdup("%");
    monitor->metrics[METRIC_CPU_USAGE].min_value = 0.0;
    monitor->metrics[METRIC_CPU_USAGE].max_value = 100.0;

    monitor->metrics[METRIC_MEMORY_USAGE].type = METRIC_MEMORY_USAGE;
    monitor->metrics[METRIC_MEMORY_USAGE].name = strdup("Memory Usage");
    monitor->metrics[METRIC_MEMORY_USAGE].unit = strdup("MB");
    monitor->metrics[METRIC_MEMORY_USAGE].min_value = 0.0;
    monitor->metrics[METRIC_MEMORY_USAGE].max_value = DBL_MAX;

    monitor->metrics[METRIC_DISK_IO].type = METRIC_DISK_IO;
    monitor->metrics[METRIC_DISK_IO].name = strdup("Disk I/O");
    monitor->metrics[METRIC_DISK_IO].unit = strdup("MB/s");
    monitor->metrics[METRIC_DISK_IO].min_value = 0.0;
    monitor->metrics[METRIC_DISK_IO].max_value = DBL_MAX;

    monitor->metrics[METRIC_NETWORK_IO].type = METRIC_NETWORK_IO;
    monitor->metrics[METRIC_NETWORK_IO].name = strdup("Network I/O");
    monitor->metrics[METRIC_NETWORK_IO].unit = strdup("MB/s");
    monitor->metrics[METRIC_NETWORK_IO].min_value = 0.0;
    monitor->metrics[METRIC_NETWORK_IO].max_value = DBL_MAX;

    monitor->metrics[METRIC_THREAD_COUNT].type = METRIC_THREAD_COUNT;
    monitor->metrics[METRIC_THREAD_COUNT].name = strdup("Thread Count");
    monitor->metrics[METRIC_THREAD_COUNT].unit = strdup("");
    monitor->metrics[METRIC_THREAD_COUNT].min_value = 0.0;
    monitor->metrics[METRIC_THREAD_COUNT].max_value = DBL_MAX;

    monitor->metrics[METRIC_HANDLE_COUNT].type = METRIC_HANDLE_COUNT;
    monitor->metrics[METRIC_HANDLE_COUNT].name = strdup("Handle Count");
    monitor->metrics[METRIC_HANDLE_COUNT].unit = strdup("");
    monitor->metrics[METRIC_HANDLE_COUNT].min_value = 0.0;
    monitor->metrics[METRIC_HANDLE_COUNT].max_value = DBL_MAX;

    monitor->metrics[METRIC_CONTEXT_SWITCHES].type = METRIC_CONTEXT_SWITCHES;
    monitor->metrics[METRIC_CONTEXT_SWITCHES].name = strdup("Context Switches");
    monitor->metrics[METRIC_CONTEXT_SWITCHES].unit = strdup("/s");
    monitor->metrics[METRIC_CONTEXT_SWITCHES].min_value = 0.0;
    monitor->metrics[METRIC_CONTEXT_SWITCHES].max_value = DBL_MAX;

    monitor->metrics[METRIC_PAGE_FAULTS].type = METRIC_PAGE_FAULTS;
    monitor->metrics[METRIC_PAGE_FAULTS].name = strdup("Page Faults");
    monitor->metrics[METRIC_PAGE_FAULTS].unit = strdup("/s");
    monitor->metrics[METRIC_PAGE_FAULTS].min_value = 0.0;
    monitor->metrics[METRIC_PAGE_FAULTS].max_value = DBL_MAX;

    // 设置进程信息
    monitor->monitored_pid = getpid();
    monitor->start_time = time(NULL);

    // 获取进程名
    char exe_path[4096];
    ssize_t len = readlink("/proc/self/exe", exe_path, sizeof(exe_path) - 1);
    if (len > 0) {
        exe_path[len] = '\0';
        char* slash = strrchr(exe_path, '/');
        if (slash) {
            monitor->process_name = strdup(slash + 1);
        } else {
            monitor->process_name = strdup(exe_path);
        }
    }

    // 打开日志文件
    if (monitor->config.enable_logging && monitor->config.log_file_path) {
        monitor->log_file = fopen(monitor->config.log_file_path, "a");
        if (!monitor->log_file) {
            perror("Failed to open performance log file");
        }
    }

    return monitor;
}

/* 销毁性能监控器 */
void performance_monitor_destroy(performance_monitor_t* monitor) {
    if (!monitor) return;

    // 停止监控
    if (monitor->running) {
        performance_monitor_stop(monitor);
    }

    // 释放指标名称和单位
    for (int i = 0; i < METRIC_MAX; i++) {
        free(monitor->metrics[i].name);
        free(monitor->metrics[i].unit);
    }

    // 关闭日志文件
    if (monitor->log_file) {
        fclose(monitor->log_file);
    }

    // 释放进程名
    free(monitor->process_name);
    free(monitor);
}

/* 启动性能监控 */
bool performance_monitor_start(performance_monitor_t* monitor) {
    if (!monitor || monitor->running) return false;

    monitor->running = true;

    // 创建监控线程
    if (pthread_create(&monitor->monitor_thread, NULL, monitor_thread_func, monitor) != 0) {
        perror("Failed to create monitor thread");
        monitor->running = false;
        return false;
    }

    printf("Performance monitoring started for process %d (%s)\n",
           monitor->monitored_pid, monitor->process_name);
    return true;
}

/* 停止性能监控 */
void performance_monitor_stop(performance_monitor_t* monitor) {
    if (!monitor || !monitor->running) return;

    monitor->running = false;

    // 等待监控线程结束
    pthread_join(monitor->monitor_thread, NULL);

    printf("Performance monitoring stopped\n");
}

/* 监控线程函数 */
static void* monitor_thread_func(void* arg) {
    performance_monitor_t* monitor = (performance_monitor_t*)arg;
    struct timespec interval;
    interval.tv_sec = monitor->config.sampling_interval_ms / 1000;
    interval.tv_nsec = (monitor->config.sampling_interval_ms % 1000) * 1000000;

    while (monitor->running) {
        // 更新性能指标
        performance_monitor_update_metrics(monitor);

        // 实时显示
        if (monitor->config.enable_real_time_display) {
            performance_monitor_display_status(monitor);
        }

        // 日志记录
        if (monitor->config.enable_logging && monitor->log_file) {
            performance_monitor_log_metrics(monitor);
        }

        // 检查报警条件
        if (monitor->config.enable_alerts) {
            double cpu_usage = monitor->metrics[METRIC_CPU_USAGE].value;
            double memory_usage = monitor->metrics[METRIC_MEMORY_USAGE].value;

            if (cpu_usage > monitor->config.cpu_alert_threshold) {
                printf("ALERT: CPU usage %.2f%% exceeds threshold %.2f%%\n",
                       cpu_usage, monitor->config.cpu_alert_threshold);
            }

            if (memory_usage > monitor->config.memory_alert_threshold) {
                printf("ALERT: Memory usage %.2f MB exceeds threshold %.2f MB\n",
                       memory_usage, monitor->config.memory_alert_threshold);
            }
        }

        // 限制样本数量
        if (monitor->total_samples >= monitor->config.max_samples) {
            monitor->running = false;
            break;
        }

        // 等待下一个采样间隔
        nanosleep(&interval, NULL);
    }

    return NULL;
}

/* 更新性能指标 */
void performance_monitor_update_metrics(performance_monitor_t* monitor) {
    if (!monitor) return;

    monitor->metrics[METRIC_CPU_USAGE].value = get_cpu_usage();
    monitor->metrics[METRIC_MEMORY_USAGE].value = get_memory_usage();
    monitor->metrics[METRIC_DISK_IO].value = get_disk_io();
    monitor->metrics[METRIC_NETWORK_IO].value = get_network_io();
    monitor->metrics[METRIC_THREAD_COUNT].value = (double)get_thread_count();
    monitor->metrics[METRIC_HANDLE_COUNT].value = (double)get_handle_count();
    monitor->metrics[METRIC_CONTEXT_SWITCHES].value = (double)get_context_switches();
    monitor->metrics[METRIC_PAGE_FAULTS].value = (double)get_page_faults();

    // 更新统计信息
    for (int i = 0; i < METRIC_MAX; i++) {
        performance_metric_t* metric = &monitor->metrics[i];
        metric->last_update = time(NULL);
        metric->sample_count++;

        // 更新最小值和最大值
        if (metric->value < metric->min_value || metric->sample_count == 1) {
            metric->min_value = metric->value;
        }
        if (metric->value > metric->max_value || metric->sample_count == 1) {
            metric->max_value = metric->value;
        }

        // 更新平均值
        metric->avg_value = (metric->avg_value * (metric->sample_count - 1) + metric->value) / metric->sample_count;
    }

    monitor->total_samples++;
}

/* 显示性能状态 */
void performance_monitor_display_status(performance_monitor_t* monitor) {
    if (!monitor) return;

    time_t now = time(NULL);
    char time_str[64];
    strftime(time_str, sizeof(time_str), "%Y-%m-%d %H:%M:%S", localtime(&now));

    printf("\033[2J\033[H");  // 清屏并移动光标到顶部
    printf("=== PERFORMANCE MONITOR ===\n");
    printf("Time: %s\n", time_str);
    printf("Process: %s (PID: %d)\n", monitor->process_name, monitor->monitored_pid);
    printf("Samples: %lu\n", monitor->total_samples);
    printf("Runtime: %ld seconds\n", now - monitor->start_time);
    printf("\n");

    for (int i = 0; i < METRIC_MAX; i++) {
        performance_metric_t* metric = &monitor->metrics[i];
        printf("%-20s: %8.2f %s (Min: %8.2f, Max: %8.2f, Avg: %8.2f)\n",
               metric->name, metric->value, metric->unit,
               metric->min_value, metric->max_value, metric->avg_value);
    }

    printf("\nPress Ctrl+C to stop monitoring...\n");
}

/* 日志记录指标 */
void performance_monitor_log_metrics(performance_monitor_t* monitor) {
    if (!monitor || !monitor->log_file) return;

    time_t now = time(NULL);
    char time_str[64];
    strftime(time_str, sizeof(time_str), "%Y-%m-%d %H:%M:%S", localtime(&now));

    fprintf(monitor->log_file, "%s,", time_str);
    fprintf(monitor->log_file, "%lu,", monitor->total_samples);

    for (int i = 0; i < METRIC_MAX; i++) {
        fprintf(monitor->log_file, "%.2f,", monitor->metrics[i].value);
    }

    fprintf(monitor->log_file, "\n");
    fflush(monitor->log_file);
}

/* 获取指标值 */
double performance_monitor_get_metric_value(performance_monitor_t* monitor, metric_type_t type) {
    if (!monitor || type < 0 || type >= METRIC_MAX) return 0.0;
    return monitor->metrics[type].value;
}

/* 系统信息获取函数 */
static double get_cpu_usage(void) {
    static unsigned long long prev_user = 0, prev_nice = 0, prev_system = 0;
    static unsigned long long prev_idle = 0, prev_iowait = 0;
    static unsigned long long prev_irq = 0, prev_softirq = 0;
    static unsigned long long prev_steal = 0, prev_guest = 0;

    FILE* fp = fopen("/proc/stat", "r");
    if (!fp) return 0.0;

    char line[256];
    fgets(line, sizeof(line), fp);
    fclose(fp);

    unsigned long long user, nice, system, idle, iowait, irq, softirq, steal, guest;
    sscanf(line, "cpu %llu %llu %llu %llu %llu %llu %llu %llu %llu",
           &user, &nice, &system, &idle, &iowait, &irq, &softirq, &steal, &guest);

    unsigned long long total_diff = (user - prev_user) + (nice - prev_nice) +
                                   (system - prev_system) + (idle - prev_idle) +
                                   (iowait - prev_iowait) + (irq - prev_irq) +
                                   (softirq - prev_softirq) + (steal - prev_steal) +
                                   (guest - prev_guest);

    unsigned long busy_diff = (user - prev_user) + (nice - prev_nice) +
                             (system - prev_system) + (irq - prev_irq) +
                             (softirq - prev_softirq) + (steal - prev_steal);

    prev_user = user;
    prev_nice = nice;
    prev_system = system;
    prev_idle = idle;
    prev_iowait = iowait;
    prev_irq = irq;
    prev_softirq = softirq;
    prev_steal = steal;
    prev_guest = guest;

    if (total_diff == 0) return 0.0;
    return (double)busy_diff / total_diff * 100.0;
}

static double get_memory_usage(void) {
    FILE* fp = fopen("/proc/self/status", "r");
    if (!fp) return 0.0;

    char line[256];
    unsigned long vmrss = 0;

    while (fgets(line, sizeof(line), fp)) {
        if (strncmp(line, "VmRSS:", 6) == 0) {
            sscanf(line, "VmRSS: %lu kB", &vmrss);
            break;
        }
    }

    fclose(fp);
    return vmrss / 1024.0;  // 转换为MB
}

static double get_disk_io(void) {
    // 简化的磁盘IO监控
    static unsigned long prev_read = 0, prev_write = 0;

    FILE* fp = fopen("/proc/self/io", "r");
    if (!fp) return 0.0;

    char line[256];
    unsigned long read_bytes = 0, write_bytes = 0;

    while (fgets(line, sizeof(line), fp)) {
        if (strncmp(line, "read_bytes:", 11) == 0) {
            sscanf(line, "read_bytes: %lu", &read_bytes);
        } else if (strncmp(line, "write_bytes:", 12) == 0) {
            sscanf(line, "write_bytes: %lu", &write_bytes);
        }
    }

    fclose(fp);

    unsigned long total_io = (read_bytes - prev_read) + (write_bytes - prev_write);
    prev_read = read_bytes;
    prev_write = write_bytes;

    return total_io / (1024.0 * 1024.0);  // 转换为MB
}

static double get_network_io(void) {
    // 简化的网络IO监控
    return 0.0;  // 实际实现需要读取/proc/net/dev
}

static unsigned long get_thread_count(void) {
    char path[256];
    snprintf(path, sizeof(path), "/proc/%d/status", getpid());

    FILE* fp = fopen(path, "r");
    if (!fp) return 0;

    char line[256];
    unsigned long threads = 0;

    while (fgets(line, sizeof(line), fp)) {
        if (strncmp(line, "Threads:", 8) == 0) {
            sscanf(line, "Threads: %lu", &threads);
            break;
        }
    }

    fclose(fp);
    return threads;
}

static unsigned long get_handle_count(void) {
    char path[256];
    snprintf(path, sizeof(path), "/proc/%d/fd", getpid());

    DIR* dir = opendir(path);
    if (!dir) return 0;

    unsigned long count = 0;
    struct dirent* entry;

    while ((entry = readdir(dir)) != NULL) {
        if (strcmp(entry->d_name, ".") != 0 && strcmp(entry->d_name, "..") != 0) {
            count++;
        }
    }

    closedir(dir);
    return count;
}

static unsigned long get_context_switches(void) {
    // 简化的上下文切换监控
    return 0;  // 实际实现需要读取/proc/self/status
}

static unsigned long get_page_faults(void) {
    char path[256];
    snprintf(path, sizeof(path), "/proc/%d/stat", getpid());

    FILE* fp = fopen(path, "r");
    if (!fp) return 0;

    char line[1024];
    fgets(line, sizeof(line), fp);
    fclose(fp);

    // 解析/proc/[pid]/stat的第11和第12个字段
    unsigned long minflt = 0, majflt = 0;
    sscanf(line, "%*d %*s %*c %*d %*d %*d %*d %*d %*d %*d %lu %lu",
           &minflt, &majflt);

    return minflt + majflt;
}
```

## 7. 最佳实践与总结

### 7.1 现代C语言开发最佳实践

1. **构建系统选择**
   - 使用CMake进行跨平台构建
   - 集成包管理器（Conan、Vcpkg）
   - 配置CI/CD流水线

2. **代码质量保证**
   - 启用严格的编译器警告
   - 使用静态分析工具
   - 设置代码质量门禁

3. **调试与测试**
   - 集成内存调试工具
   - 实现自动化测试
   - 使用性能分析工具

4. **性能优化**
   - 进行缓存友好编程
   - 使用性能计数器
   - 实施监控和报警

### 7.2 工具链推荐

**构建工具**
- CMake: 跨平台构建
- Meson: 现代构建系统
- Ninja: 快速构建后端

**静态分析**
- Clang-Tidy: 代码质量检查
- Cppcheck: 深度静态分析
- SonarQube: 代码质量平台

**调试工具**
- GDB: 调试器
- Valgrind: 内存检查
- AddressSanitizer: 内存安全

**性能分析**
- perf: Linux性能工具
- VTune: Intel性能分析器
- Instruments: macOS性能工具

### 7.3 开发工作流

1. **开发环境设置**
   ```bash
   # 使用Docker容器化开发环境
   docker-compose up c-dev

   # 初始化构建
   cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Debug
   ```

2. **日常开发**
   ```bash
   # 构建项目
   cmake --build build

   # 运行测试
   cd build && ctest --output-on-failure

   # 静态分析
   cppcheck --enable=all src/
   ```

3. **性能优化**
   ```bash
   # 性能分析
   perf record -g ./build/your_program
   perf report

   # 内存检查
   valgrind --tool=memcheck ./build/your_program
   ```

### 7.4 总结

现代C语言开发已经发展为一个完整的生态系统，包含构建系统、静态分析、调试工具、性能分析等组件。通过合理使用这些工具，可以显著提升C语言项目的开发效率、代码质量和运行性能。

关键要点：
- 建立现代化的构建和测试流程
- 重视代码质量和安全性
- 系统化地进行性能优化
- 采用容器化和CI/CD实践
- 持续学习和应用新技术

通过本文的学习，您应该能够在实际的C语言项目中应用这些现代开发实践，构建高质量、高性能的系统软件。