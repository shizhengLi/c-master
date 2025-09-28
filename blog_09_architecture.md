# C语言架构设计：模块化编程、接口设计与代码组织

## 引言

优秀的软件架构是构建大型、复杂、可维护系统的关键。在C语言中，虽然没有面向对象的语言特性，但通过良好的架构设计和编程模式，我们仍然可以构建出结构清晰、模块化、可扩展的系统。本文将深入探讨C语言的架构设计原则、模块化编程技术、接口设计模式和代码组织策略。

## 1. 模块化编程基础

### 1.1 模块定义与边界

```c
// 模块的基本概念
// 一个模块应该包含：
// 1. 明确的接口（头文件）
// 2. 隐藏的实现细节（源文件）
// 3. 清晰的责任边界

// 示例：日志模块
// log.h - 公共接口
#ifndef LOG_H
#define LOG_H

#include <stdarg.h>
#include <stdbool.h>

// 日志级别
typedef enum {
    LOG_LEVEL_ERROR = 0,
    LOG_LEVEL_WARN  = 1,
    LOG_LEVEL_INFO  = 2,
    LOG_LEVEL_DEBUG = 3,
    LOG_LEVEL_TRACE = 4
} log_level_t;

// 日志模块接口
typedef struct {
    void (*init)(const char* filename);
    void (*cleanup)(void);
    void (*write)(log_level_t level, const char* format, ...);
    void (*set_level)(log_level_t level);
    bool (*is_enabled)(log_level_t level);
} log_module_t;

// 获取日志模块实例
const log_module_t* get_log_module(void);

// 便利宏
#define LOG_ERROR(fmt, ...) get_log_module()->write(LOG_LEVEL_ERROR, fmt, ##__VA_ARGS__)
#define LOG_WARN(fmt, ...)  get_log_module()->write(LOG_LEVEL_WARN, fmt, ##__VA_ARGS__)
#define LOG_INFO(fmt, ...)  get_log_module()->write(LOG_LEVEL_INFO, fmt, ##__VA_ARGS__)
#define LOG_DEBUG(fmt, ...) get_log_module()->write(LOG_LEVEL_DEBUG, fmt, ##__VA_ARGS__)

#endif // LOG_H

// log.c - 私有实现
#include "log.h"
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <string.h>
#include <pthread.h>

// 私有数据结构
typedef struct {
    FILE* file;
    log_level_t level;
    pthread_mutex_t mutex;
    char time_buffer[64];
} log_context_t;

// 静态实例（单例模式）
static log_context_t g_log_context = {
    .file = NULL,
    .level = LOG_LEVEL_INFO,
    .mutex = PTHREAD_MUTEX_INITIALIZER
};

// 私有函数
static const char* level_to_string(log_level_t level) {
    static const char* levels[] = {"ERROR", "WARN", "INFO", "DEBUG", "TRACE"};
    return levels[level];
}

static void format_time(char* buffer, size_t size) {
    time_t now = time(NULL);
    struct tm* tm_info = localtime(&now);
    strftime(buffer, size, "%Y-%m-%d %H:%M:%S", tm_info);
}

// 接口实现
static void log_init(const char* filename) {
    pthread_mutex_lock(&g_log_context.mutex);

    if (g_log_context.file) {
        fclose(g_log_context.file);
    }

    if (filename) {
        g_log_context.file = fopen(filename, "a");
    } else {
        g_log_context.file = stdout;
    }

    pthread_mutex_unlock(&g_log_context.mutex);
}

static void log_cleanup(void) {
    pthread_mutex_lock(&g_log_context.mutex);

    if (g_log_context.file && g_log_context.file != stdout) {
        fclose(g_log_context.file);
        g_log_context.file = NULL;
    }

    pthread_mutex_unlock(&g_log_context.mutex);
}

static void log_write(log_level_t level, const char* format, ...) {
    if (level > g_log_context.level) {
        return;
    }

    pthread_mutex_lock(&g_log_context.mutex);

    if (!g_log_context.file) {
        pthread_mutex_unlock(&g_log_context.mutex);
        return;
    }

    format_time(g_log_context.time_buffer, sizeof(g_log_context.time_buffer));

    fprintf(g_log_context.file, "[%s] [%s] ",
            g_log_context.time_buffer, level_to_string(level));

    va_list args;
    va_start(args, format);
    vfprintf(g_log_context.file, format, args);
    va_end(args);

    fprintf(g_log_context.file, "\n");
    fflush(g_log_context.file);

    pthread_mutex_unlock(&g_log_context.mutex);
}

static void log_set_level(log_level_t level) {
    g_log_context.level = level;
}

static bool log_is_enabled(log_level_t level) {
    return level <= g_log_context.level;
}

// 模块接口实例
static const log_module_t g_log_module = {
    .init = log_init,
    .cleanup = log_cleanup,
    .write = log_write,
    .set_level = log_set_level,
    .is_enabled = log_is_enabled
};

const log_module_t* get_log_module(void) {
    return &g_log_module;
}
```

### 1.2 模块间通信模式

```c
// 模块间通信的几种模式

// 1. 函数调用模式
// network.h
#ifndef NETWORK_H
#define NETWORK_H

#include <stddef.h>

typedef struct {
    int socket;
    void* context;
} connection_t;

typedef struct {
    connection_t* (*connect)(const char* host, int port);
    int (*send)(connection_t* conn, const void* data, size_t size);
    int (*receive)(connection_t* conn, void* buffer, size_t size);
    void (*close)(connection_t* conn);
} network_module_t;

const network_module_t* get_network_module(void);

#endif // NETWORK_H

// 2. 回调模式
// event_system.h
#ifndef EVENT_SYSTEM_H
#define EVENT_SYSTEM_H

typedef enum {
    EVENT_TYPE_MOUSE_CLICK,
    EVENT_TYPE_KEY_PRESS,
    EVENT_TYPE_WINDOW_CLOSE,
    EVENT_TYPE_CUSTOM_START = 1000
} event_type_t;

typedef struct {
    event_type_t type;
    int x, y;
    int key;
    void* user_data;
} event_t;

typedef void (*event_handler_t)(const event_t* event);

typedef struct {
    void (*register_handler)(event_type_t type, event_handler_t handler);
    void (*unregister_handler)(event_type_t type, event_handler_t handler);
    void (*post_event)(const event_t* event);
    void (*process_events)(void);
} event_system_t;

const event_system_t* get_event_system(void);

#endif // EVENT_SYSTEM_H

// 3. 观察者模式
// observer.h
#ifndef OBSERVER_H
#define OBSERVER_H

typedef struct observer observer_t;
typedef struct subject subject_t;

// 观察者接口
struct observer {
    void (*update)(observer_t* self, subject_t* subject, void* data);
    void* context;
};

// 主题接口
struct subject {
    void (*attach)(subject_t* self, observer_t* observer);
    void (*detach)(subject_t* self, observer_t* observer);
    void (*notify)(subject_t* self, void* data);
    observer_t** observers;
    size_t observer_count;
    size_t observer_capacity;
};

// 工厂函数
subject_t* create_subject(void);
void destroy_subject(subject_t* subject);
observer_t* create_observer(void (*update)(observer_t*, subject_t*, void*), void* context);
void destroy_observer(observer_t* observer);

#endif // OBSERVER_H

// 4. 消息队列模式
// message_queue.h
#ifndef MESSAGE_QUEUE_H
#define MESSAGE_QUEUE_H

#include <pthread.h>

typedef struct {
    int type;
    int priority;
    void* data;
    void (*destructor)(void*);
} message_t;

typedef struct {
    message_t* messages;
    size_t size;
    size_t capacity;
    pthread_mutex_t mutex;
    pthread_cond_t not_empty;
    pthread_cond_t not_full;
} message_queue_t;

message_queue_t* create_message_queue(size_t capacity);
void destroy_message_queue(message_queue_t* queue);
int send_message(message_queue_t* queue, const message_t* message);
int receive_message(message_queue_t* queue, message_t* message, int timeout_ms);

#endif // MESSAGE_QUEUE_H
```

### 1.3 依赖注入与控制反转

```c
// 依赖注入模式示例
// database.h
#ifndef DATABASE_H
#define DATABASE_H

#include <stdbool.h>

typedef struct {
    void* connection;
    bool (*connect)(const char* url);
    void (*disconnect)(void);
    bool (*execute)(const char* query);
    bool (*query)(const char* query, void (*callback)(void* row, void* context), void* context);
} database_t;

// 数据库工厂
typedef database_t* (*database_factory_t)(const char* config);

#endif // DATABASE_H

// service.h
#ifndef SERVICE_H
#define SERVICE_H

#include "database.h"

typedef struct {
    database_t* db;
    void (*process_data)(struct service* self, const char* data);
    void (*cleanup)(struct service* self);
} service_t;

// 服务工厂，支持依赖注入
service_t* create_service(database_t* db);

#endif // SERVICE_H

// service.c
#include "service.h"

static void service_process_data(service_t* self, const char* data) {
    if (!self || !self->db || !data) return;

    // 使用注入的数据库依赖
    char query[256];
    snprintf(query, sizeof(query), "INSERT INTO data VALUES ('%s')", data);

    if (!self->db->execute(query)) {
        // 处理错误
    }
}

static void service_cleanup(service_t* self) {
    if (self) {
        if (self->db) {
            self->db->disconnect();
        }
        // 注意：这里不释放db，因为它是注入的依赖
    }
}

service_t* create_service(database_t* db) {
    if (!db) return NULL;

    service_t* service = malloc(sizeof(service_t));
    if (!service) return NULL;

    service->db = db;
    service->process_data = service_process_data;
    service->cleanup = service_cleanup;

    return service;
}

// 控制容器示例
typedef struct {
    database_t* db;
    service_t* service;
    void (*init)(struct container* self);
    void (*destroy)(struct container* self);
} container_t;

static void container_init(container_t* self) {
    // 创建依赖
    self->db = create_database("sqlite://test.db");
    if (!self->db) return;

    // 注入依赖
    self->service = create_service(self->db);
}

static void container_destroy(container_t* self) {
    if (self) {
        if (self->service) {
            self->service->cleanup(self->service);
            free(self->service);
        }
        if (self->db) {
            // 数据库的创建和销毁应该由容器管理
            self->db->disconnect();
            free(self->db);
        }
    }
}

container_t* create_container(void) {
    container_t* container = malloc(sizeof(container_t));
    if (!container) return NULL;

    container->db = NULL;
    container->service = NULL;
    container->init = container_init;
    container->destroy = container_destroy;

    container->init(container);
    return container;
}
```

## 2. 接口设计模式

### 2.1 抽象接口与多态

```c
// 使用函数指针实现多态
// shape.h
#ifndef SHAPE_H
#define SHAPE_H

#include <stdbool.h>

// 基础形状接口
typedef struct {
    void (*draw)(struct shape* self);
    void (*move)(struct shape* self, int x, int y);
    bool (*contains)(struct shape* self, int x, int y);
    void (*destroy)(struct shape* self);
    double (*area)(struct shape* self);
} shape_t;

// 工厂函数
shape_t* create_rectangle(int x, int y, int width, int height);
shape_t* create_circle(int x, int y, int radius);
shape_t* create_triangle(int x1, int y1, int x2, int y2, int x3, int y3);

#endif // SHAPE_H

// shape.c
#include "shape.h"
#include <stdlib.h>
#include <math.h>

// 矩形实现
typedef struct {
    shape_t base;
    int x, y;
    int width, height;
} rectangle_t;

static void rectangle_draw(shape_t* self) {
    rectangle_t* rect = (rectangle_t*)self;
    printf("Drawing rectangle at (%d, %d) size %dx%d\n",
           rect->x, rect->y, rect->width, rect->height);
}

static void rectangle_move(shape_t* self, int x, int y) {
    rectangle_t* rect = (rectangle_t*)self;
    rect->x = x;
    rect->y = y;
}

static bool rectangle_contains(shape_t* self, int x, int y) {
    rectangle_t* rect = (rectangle_t*)self;
    return x >= rect->x && x <= rect->x + rect->width &&
           y >= rect->y && y <= rect->y + rect->height;
}

static double rectangle_area(shape_t* self) {
    rectangle_t* rect = (rectangle_t*)self;
    return rect->width * rect->height;
}

static void rectangle_destroy(shape_t* self) {
    free(self);
}

shape_t* create_rectangle(int x, int y, int width, int height) {
    rectangle_t* rect = malloc(sizeof(rectangle_t));
    if (!rect) return NULL;

    rect->base.draw = rectangle_draw;
    rect->base.move = rectangle_move;
    rect->base.contains = rectangle_contains;
    rect->base.destroy = rectangle_destroy;
    rect->base.area = rectangle_area;

    rect->x = x;
    rect->y = y;
    rect->width = width;
    rect->height = height;

    return (shape_t*)rect;
}

// 圆形实现
typedef struct {
    shape_t base;
    int x, y;
    int radius;
} circle_t;

static void circle_draw(shape_t* self) {
    circle_t* circle = (circle_t*)self;
    printf("Drawing circle at (%d, %d) radius %d\n",
           circle->x, circle->y, circle->radius);
}

static void circle_move(shape_t* self, int x, int y) {
    circle_t* circle = (circle_t*)self;
    circle->x = x;
    circle->y = y;
}

static bool circle_contains(shape_t* self, int x, int y) {
    circle_t* circle = (circle_t*)self;
    int dx = x - circle->x;
    int dy = y - circle->y;
    return dx * dx + dy * dy <= circle->radius * circle->radius;
}

static double circle_area(shape_t* self) {
    circle_t* circle = (circle_t*)self;
    return M_PI * circle->radius * circle->radius;
}

static void circle_destroy(shape_t* self) {
    free(self);
}

shape_t* create_circle(int x, int y, int radius) {
    circle_t* circle = malloc(sizeof(circle_t));
    if (!circle) return NULL;

    circle->base.draw = circle_draw;
    circle->base.move = circle_move;
    circle->base.contains = circle_contains;
    circle->base.destroy = circle_destroy;
    circle->base.area = circle_area;

    circle->x = x;
    circle->y = y;
    circle->radius = radius;

    return (shape_t*)circle;
}

// 形状管理器
typedef struct {
    shape_t** shapes;
    size_t count;
    size_t capacity;
} shape_manager_t;

shape_manager_t* create_shape_manager(size_t initial_capacity);
void destroy_shape_manager(shape_manager_t* manager);
int add_shape(shape_manager_t* manager, shape_t* shape);
void draw_all_shapes(shape_manager_t* manager);
double total_area(shape_manager_t* manager);
```

### 2.2 插件架构设计

```c
// plugin.h
#ifndef PLUGIN_H
#define PLUGIN_H

#include <stdbool.h>

// 插件接口
typedef struct {
    const char* name;
    const char* version;
    bool (*init)(void);
    void (*cleanup)(void);
    int (*execute)(const char* command, void* result);
    void* (*get_config)(void);
} plugin_t;

// 插件管理器
typedef struct {
    plugin_t** plugins;
    size_t count;
    size_t capacity;
    void* context;
} plugin_manager_t;

plugin_manager_t* create_plugin_manager(void);
void destroy_plugin_manager(plugin_manager_t* manager);
bool load_plugin(plugin_manager_t* manager, const char* plugin_path);
bool unload_plugin(plugin_manager_t* manager, const char* plugin_name);
plugin_t* find_plugin(plugin_manager_t* manager, const char* name);

#endif // PLUGIN_H

// plugin_manager.c
#include "plugin.h"
#include <dlfcn.h>
#include <stdlib.h>
#include <string.h>

typedef struct {
    plugin_t* plugin;
    void* handle;
} loaded_plugin_t;

struct plugin_manager {
    loaded_plugin_t* plugins;
    size_t count;
    size_t capacity;
    void* context;
};

plugin_manager_t* create_plugin_manager(void) {
    plugin_manager_t* manager = malloc(sizeof(plugin_manager_t));
    if (!manager) return NULL;

    manager->plugins = NULL;
    manager->count = 0;
    manager->capacity = 0;
    manager->context = NULL;

    return manager;
}

bool load_plugin(plugin_manager_t* manager, const char* plugin_path) {
    if (!manager || !plugin_path) return false;

    // 扩展容量
    if (manager->count >= manager->capacity) {
        size_t new_capacity = manager->capacity == 0 ? 8 : manager->capacity * 2;
        loaded_plugin_t* new_plugins = realloc(manager->plugins,
                                               new_capacity * sizeof(loaded_plugin_t));
        if (!new_plugins) return false;

        manager->plugins = new_plugins;
        manager->capacity = new_capacity;
    }

    // 加载动态库
    void* handle = dlopen(plugin_path, RTLD_LAZY);
    if (!handle) {
        fprintf(stderr, "Failed to load plugin %s: %s\n", plugin_path, dlerror());
        return false;
    }

    // 获取插件入口点
    plugin_t* (*get_plugin)(void) = dlsym(handle, "get_plugin");
    if (!get_plugin) {
        fprintf(stderr, "Plugin %s has no get_plugin function\n", plugin_path);
        dlclose(handle);
        return false;
    }

    plugin_t* plugin = get_plugin();
    if (!plugin) {
        fprintf(stderr, "Plugin %s returned NULL\n", plugin_path);
        dlclose(handle);
        return false;
    }

    // 初始化插件
    if (plugin->init && !plugin->init()) {
        fprintf(stderr, "Failed to initialize plugin %s\n", plugin_path);
        dlclose(handle);
        return false;
    }

    // 添加到管理器
    loaded_plugin_t* loaded = &manager->plugins[manager->count];
    loaded->plugin = plugin;
    loaded->handle = handle;
    manager->count++;

    printf("Loaded plugin: %s v%s\n", plugin->name, plugin->version);
    return true;
}

bool unload_plugin(plugin_manager_t* manager, const char* plugin_name) {
    if (!manager || !plugin_name) return false;

    for (size_t i = 0; i < manager->count; i++) {
        loaded_plugin_t* loaded = &manager->plugins[i];
        if (strcmp(loaded->plugin->name, plugin_name) == 0) {
            // 清理插件
            if (loaded->plugin->cleanup) {
                loaded->plugin->cleanup();
            }

            // 卸载动态库
            dlclose(loaded->handle);

            // 移除条目
            for (size_t j = i; j < manager->count - 1; j++) {
                manager->plugins[j] = manager->plugins[j + 1];
            }
            manager->count--;

            printf("Unloaded plugin: %s\n", plugin_name);
            return true;
        }
    }

    return false;
}

// 示例插件
// example_plugin.c
#include "plugin.h"
#include <stdio.h>

static bool example_init(void) {
    printf("Example plugin initialized\n");
    return true;
}

static void example_cleanup(void) {
    printf("Example plugin cleaned up\n");
}

static int example_execute(const char* command, void* result) {
    printf("Example plugin executing: %s\n", command);
    return 0;
}

static void* example_get_config(void) {
    return NULL;
}

// 插件导出函数
__attribute__((visibility("default")))
plugin_t* get_plugin(void) {
    static plugin_t plugin = {
        .name = "example",
        .version = "1.0.0",
        .init = example_init,
        .cleanup = example_cleanup,
        .execute = example_execute,
        .get_config = example_get_config
    };

    return &plugin;
}
```

### 2.3 服务定位器模式

```c
// service_locator.h
#ifndef SERVICE_LOCATOR_H
#define SERVICE_LOCATOR_H

#include <stdbool.h>

// 服务类型
typedef enum {
    SERVICE_LOGGER,
    SERVICE_DATABASE,
    SERVICE_NETWORK,
    SERVICE_CONFIG,
    SERVICE_CACHE,
    SERVICE_MAX
} service_type_t;

// 服务接口
typedef struct service {
    void* instance;
    void (*destroy)(void* instance);
    const char* name;
} service_t;

// 服务定位器
typedef struct {
    service_t services[SERVICE_MAX];
    bool initialized[SERVICE_MAX];
    void* context;
} service_locator_t;

// 全局服务定位器
service_locator_t* get_service_locator(void);

// 服务注册和获取
bool register_service(service_type_t type, const char* name, void* instance,
                     void (*destroy)(void* instance));
void* get_service(service_type_t type);
void* get_service_by_name(const char* name);
bool unregister_service(service_type_t type);
void cleanup_all_services(void);

// 便利宏
#define GET_SERVICE(type) ((type##_t*)get_service(SERVICE_##type))
#define REGISTER_SERVICE(type, name, instance, destroy) \
    register_service(SERVICE_##type, name, instance, (void(*)(void*))destroy)

#endif // SERVICE_LOCATOR_H

// service_locator.c
#include "service_locator.h"
#include <stdlib.h>
#include <string.h>

static service_locator_t g_service_locator = {0};

service_locator_t* get_service_locator(void) {
    return &g_service_locator;
}

bool register_service(service_type_t type, const char* name, void* instance,
                     void (*destroy)(void* instance)) {
    if (type >= SERVICE_MAX || !name || !instance) {
        return false;
    }

    if (g_service_locator.initialized[type]) {
        // 服务已存在，先清理
        if (g_service_locator.services[type].destroy) {
            g_service_locator.services[type].destroy(
                g_service_locator.services[type].instance);
        }
    }

    g_service_locator.services[type].instance = instance;
    g_service_locator.services[type].destroy = destroy;
    g_service_locator.services[type].name = name;
    g_service_locator.initialized[type] = true;

    return true;
}

void* get_service(service_type_t type) {
    if (type >= SERVICE_MAX || !g_service_locator.initialized[type]) {
        return NULL;
    }

    return g_service_locator.services[type].instance;
}

void* get_service_by_name(const char* name) {
    if (!name) return NULL;

    for (int i = 0; i < SERVICE_MAX; i++) {
        if (g_service_locator.initialized[i] &&
            strcmp(g_service_locator.services[i].name, name) == 0) {
            return g_service_locator.services[i].instance;
        }
    }

    return NULL;
}

bool unregister_service(service_type_t type) {
    if (type >= SERVICE_MAX || !g_service_locator.initialized[type]) {
        return false;
    }

    if (g_service_locator.services[type].destroy) {
        g_service_locator.services[type].destroy(
            g_service_locator.services[type].instance);
    }

    g_service_locator.services[type].instance = NULL;
    g_service_locator.services[type].destroy = NULL;
    g_service_locator.services[type].name = NULL;
    g_service_locator.initialized[type] = false;

    return true;
}

void cleanup_all_services(void) {
    for (int i = 0; i < SERVICE_MAX; i++) {
        if (g_service_locator.initialized[i]) {
            if (g_service_locator.services[i].destroy) {
                g_service_locator.services[i].destroy(
                    g_service_locator.services[i].instance);
            }
            g_service_locator.initialized[i] = false;
        }
    }
}

// 使用示例
// main.c
#include "service_locator.h"
#include "log.h"
#include "database.h"

int main() {
    // 初始化服务定位器
    service_locator_t* locator = get_service_locator();

    // 创建并注册服务
    log_module_t* logger = (log_module_t*)get_log_module();
    logger->init("app.log");

    database_t* db = create_database("sqlite://app.db");

    REGISTER_SERVICE(LOGGER, "main_logger", logger, logger->cleanup);
    REGISTER_SERVICE(DATABASE, "main_db", db, db->disconnect);

    // 在其他模块中使用服务
    log_module_t* logger_service = GET_SERVICE(LOGGER);
    database_t* db_service = GET_SERVICE(DATABASE);

    if (logger_service) {
        logger_service->write(LOG_LEVEL_INFO, "Application started");
    }

    // 清理
    cleanup_all_services();
    return 0;
}
```

## 3. 分层架构设计

### 3.1 MVC模式实现

```c
// MVC架构在C语言中的实现

// model.h - 数据模型层
#ifndef MODEL_H
#define MODEL_H

#include <stdbool.h>

// 用户模型
typedef struct {
    int id;
    char username[64];
    char email[128];
    char password_hash[256];
    bool active;
    time_t created_at;
    time_t updated_at;
} user_model_t;

// 用户模型操作
typedef struct {
    bool (*save)(user_model_t* user);
    bool (*delete)(int user_id);
    user_model_t* (*find_by_id)(int user_id);
    user_model_t** (*find_all)(int* count);
    bool (*validate)(const user_model_t* user);
    void (*destroy)(user_model_t* user);
} user_model_ops_t;

// 工厂函数
user_model_t* create_user_model(void);
user_model_ops_t* get_user_model_ops(void);

#endif // MODEL_H

// view.h - 视图层
#ifndef VIEW_H
#define VIEW_H

#include "model.h"

// 视图接口
typedef struct {
    void (*render_user_list)(user_model_t** users, int count);
    void (*render_user_form)(const user_model_t* user);
    void (*render_error)(const char* error);
    void (*render_success)(const char* message);
    void (*show_loading)(bool show);
} view_t;

// 视图工厂
view_t* create_console_view(void);
view_t* create_web_view(void);
view_t* create_gui_view(void);

#endif // VIEW_H

// controller.h - 控制器层
#ifndef CONTROLLER_H
#define CONTROLLER_H

#include "model.h"
#include "view.h"

// 用户控制器
typedef struct {
    view_t* view;
    user_model_ops_t* model_ops;

    void (*handle_list_users)(struct user_controller* self);
    void (*handle_create_user)(struct user_controller* self, const char* username,
                               const char* email, const char* password);
    void (*handle_delete_user)(struct user_controller* self, int user_id);
    void (*handle_edit_user)(struct user_controller* self, int user_id,
                             const char* username, const char* email);
} user_controller_t;

// 控制器工厂
user_controller_t* create_user_controller(view_t* view, user_model_ops_t* model_ops);
void destroy_user_controller(user_controller_t* controller);

#endif // CONTROLLER_H

// controller.c
#include "controller.h"
#include <stdlib.h>
#include <string.h>

static void handle_list_users(user_controller_t* self) {
    if (!self) return;

    self->view->show_loading(true);

    int count = 0;
    user_model_t** users = self->model_ops->find_all(&count);

    self->view->show_loading(false);

    if (users) {
        self->view->render_user_list(users, count);

        // 清理
        for (int i = 0; i < count; i++) {
            self->model_ops->destroy(users[i]);
        }
        free(users);
    } else {
        self->view->render_error("Failed to load users");
    }
}

static void handle_create_user(user_controller_t* self, const char* username,
                               const char* email, const char* password) {
    if (!self || !username || !email || !password) return;

    user_model_t* user = create_user_model();
    if (!user) {
        self->view->render_error("Failed to create user model");
        return;
    }

    strncpy(user->username, username, sizeof(user->username) - 1);
    strncpy(user->email, email, sizeof(user->email) - 1);
    // 密码哈希处理
    strncpy(user->password_hash, password, sizeof(user->password_hash) - 1);
    user->active = true;

    if (self->model_ops->validate(user)) {
        if (self->model_ops->save(user)) {
            self->view->render_success("User created successfully");
        } else {
            self->view->render_error("Failed to save user");
        }
    } else {
        self->view->render_error("Invalid user data");
    }

    self->model_ops->destroy(user);
}

static void handle_delete_user(user_controller_t* self, int user_id) {
    if (!self || user_id <= 0) return;

    if (self->model_ops->delete(user_id)) {
        self->view->render_success("User deleted successfully");
    } else {
        self->view->render_error("Failed to delete user");
    }
}

static void handle_edit_user(user_controller_t* self, int user_id,
                             const char* username, const char* email) {
    if (!self || user_id <= 0) return;

    user_model_t* user = self->model_ops->find_by_id(user_id);
    if (!user) {
        self->view->render_error("User not found");
        return;
    }

    if (username) {
        strncpy(user->username, username, sizeof(user->username) - 1);
    }
    if (email) {
        strncpy(user->email, email, sizeof(user->email) - 1);
    }

    if (self->model_ops->validate(user)) {
        if (self->model_ops->save(user)) {
            self->view->render_success("User updated successfully");
        } else {
            self->view->render_error("Failed to update user");
        }
    } else {
        self->view->render_error("Invalid user data");
    }

    self->model_ops->destroy(user);
}

user_controller_t* create_user_controller(view_t* view, user_model_ops_t* model_ops) {
    if (!view || !model_ops) return NULL;

    user_controller_t* controller = malloc(sizeof(user_controller_t));
    if (!controller) return NULL;

    controller->view = view;
    controller->model_ops = model_ops;
    controller->handle_list_users = handle_list_users;
    controller->handle_create_user = handle_create_user;
    controller->handle_delete_user = handle_delete_user;
    controller->handle_edit_user = handle_edit_user;

    return controller;
}

void destroy_user_controller(user_controller_t* controller) {
    if (controller) {
        free(controller);
    }
}
```

### 3.2 领域驱动设计

```c
// 领域驱动设计在C语言中的实现

// domain.h - 领域模型
#ifndef DOMAIN_H
#define DOMAIN_H

#include <stdbool.h>
#include <time.h>

// 值对象：货币
typedef struct {
    double amount;
    char currency[4];  // USD, EUR, CNY等
} money_t;

// 值对象：地址
typedef struct {
    char street[128];
    char city[64];
    char state[32];
    char zip_code[16];
    char country[32];
} address_t;

// 实体：客户
typedef struct {
    int id;
    char name[128];
    address_t* address;
    money_t credit_limit;
    bool is_active;
    time_t created_at;
    // 业务方法
    bool (*can_place_order)(struct customer* customer, money_t amount);
    void (*add_credit)(struct customer* customer, money_t amount);
} customer_t;

// 实体：订单
typedef struct {
    int id;
    int customer_id;
    money_t total_amount;
    time_t order_date;
    char status[32];
    // 业务方法
    bool (*can_cancel)(struct order* order);
    void (*calculate_total)(struct order* order);
} order_t;

// 领域服务
typedef struct {
    bool (*validate_customer)(customer_t* customer);
    bool (*validate_order)(order_t* order);
    bool (*process_payment)(customer_t* customer, order_t* order);
    void (*send_confirmation)(customer_t* customer, order_t* order);
} order_service_t;

// 仓库接口
typedef struct {
    customer_t* (*find_customer_by_id)(int id);
    order_t* (*find_order_by_id)(int id);
    bool (*save_customer)(customer_t* customer);
    bool (*save_order)(order_t* order);
} repository_t;

// 工厂函数
customer_t* create_customer(const char* name, const address_t* address, money_t credit_limit);
order_t* create_order(int customer_id);
money_t create_money(double amount, const char* currency);
address_t* create_address(const char* street, const char* city, const char* state,
                          const char* zip_code, const char* country);
order_service_t* create_order_service(repository_t* repository);

#endif // DOMAIN_H

// domain.c
#include "domain.h"
#include <stdlib.h>
#include <string.h>
#include <stdio.h>

// 值对象实现
money_t create_money(double amount, const char* currency) {
    money_t money;
    money.amount = amount;
    strncpy(money.currency, currency ? currency : "USD", sizeof(money.currency) - 1);
    money.currency[sizeof(money.currency) - 1] = '\0';
    return money;
}

address_t* create_address(const char* street, const char* city, const char* state,
                         const char* zip_code, const char* country) {
    address_t* address = malloc(sizeof(address_t));
    if (!address) return NULL;

    strncpy(address->street, street ? street : "", sizeof(address->street) - 1);
    strncpy(address->city, city ? city : "", sizeof(address->city) - 1);
    strncpy(address->state, state ? state : "", sizeof(address->state) - 1);
    strncpy(address->zip_code, zip_code ? zip_code : "", sizeof(address->zip_code) - 1);
    strncpy(address->country, country ? country : "", sizeof(address->country) - 1);

    // 确保字符串终止
    address->street[sizeof(address->street) - 1] = '\0';
    address->city[sizeof(address->city) - 1] = '\0';
    address->state[sizeof(address->state) - 1] = '\0';
    address->zip_code[sizeof(address->zip_code) - 1] = '\0';
    address->country[sizeof(address->country) - 1] = '\0';

    return address;
}

// 实体方法实现
static bool customer_can_place_order(customer_t* customer, money_t amount) {
    if (!customer) return false;
    return customer->is_active && customer->credit_limit.amount >= amount.amount &&
           strcmp(customer->credit_limit.currency, amount.currency) == 0;
}

static void customer_add_credit(customer_t* customer, money_t amount) {
    if (!customer) return;
    if (strcmp(customer->credit_limit.currency, amount.currency) == 0) {
        customer->credit_limit.amount += amount.amount;
    }
}

customer_t* create_customer(const char* name, const address_t* address, money_t credit_limit) {
    customer_t* customer = malloc(sizeof(customer_t));
    if (!customer) return NULL;

    customer->id = 0;  // 将由数据库分配
    strncpy(customer->name, name ? name : "", sizeof(customer->name) - 1);
    customer->name[sizeof(customer->name) - 1] = '\0';
    customer->address = address ? create_address(address->street, address->city,
                                                 address->state, address->zip_code,
                                                 address->country) : NULL;
    customer->credit_limit = credit_limit;
    customer->is_active = true;
    customer->created_at = time(NULL);
    customer->can_place_order = customer_can_place_order;
    customer->add_credit = customer_add_credit;

    return customer;
}

// 领域服务实现
static bool validate_customer(customer_t* customer) {
    if (!customer) return false;
    if (strlen(customer->name) == 0) return false;
    if (!customer->address) return false;
    if (customer->credit_limit.amount <= 0) return false;
    return true;
}

static bool validate_order(order_t* order) {
    if (!order) return false;
    if (order->customer_id <= 0) return false;
    if (order->total_amount.amount <= 0) return false;
    return true;
}

static bool process_payment(customer_t* customer, order_t* order) {
    if (!customer || !order) return false;

    if (!customer->can_place_order(customer, order->total_amount)) {
        return false;
    }

    // 扣除信用额度
    customer->credit_limit.amount -= order->total_amount.amount;

    return true;
}

static void send_confirmation(customer_t* customer, order_t* order) {
    if (!customer || !order) return;

    printf("Order confirmation sent to customer %s for order %d, amount %.2f %s\n",
           customer->name, order->id, order->total_amount.amount, order->total_amount.currency);
}

order_service_t* create_order_service(repository_t* repository) {
    order_service_t* service = malloc(sizeof(order_service_t));
    if (!service) return NULL;

    service->validate_customer = validate_customer;
    service->validate_order = validate_order;
    service->process_payment = process_payment;
    service->send_confirmation = send_confirmation;

    return service;
}

// 应用服务
typedef struct {
    order_service_t* order_service;
    repository_t* repository;
} application_service_t;

bool place_order(application_service_t* app_service, int customer_id, money_t amount) {
    if (!app_service || customer_id <= 0 || amount.amount <= 0) {
        return false;
    }

    // 获取客户
    customer_t* customer = app_service->repository->find_customer_by_id(customer_id);
    if (!customer) {
        return false;
    }

    // 创建订单
    order_t* order = create_order(customer_id);
    order->total_amount = amount;

    // 验证
    if (!app_service->order_service->validate_customer(customer) ||
        !app_service->order_service->validate_order(order)) {
        free(customer);
        free(order);
        return false;
    }

    // 处理支付
    if (!app_service->order_service->process_payment(customer, order)) {
        free(customer);
        free(order);
        return false;
    }

    // 保存订单
    if (!app_service->repository->save_order(order)) {
        free(customer);
        free(order);
        return false;
    }

    // 保存客户
    app_service->repository->save_customer(customer);

    // 发送确认
    app_service->order_service->send_confirmation(customer, order);

    free(customer);
    free(order);
    return true;
}
```

### 3.3 事件驱动架构

```c
// event_driven.h - 事件驱动架构
#ifndef EVENT_DRIVEN_H
#define EVENT_DRIVEN_H

#include <stdbool.h>
#include <stddef.h>

// 事件类型
typedef enum {
    EVENT_USER_CREATED,
    EVENT_USER_UPDATED,
    EVENT_USER_DELETED,
    EVENT_ORDER_PLACED,
    EVENT_ORDER_CANCELLED,
    EVENT_PAYMENT_PROCESSED,
    EVENT_NOTIFICATION_SENT,
    EVENT_SYSTEM_ERROR,
    EVENT_CUSTOM_START = 1000
} event_type_t;

// 事件数据基类
typedef struct {
    event_type_t type;
    void* data;
    void (*destroy)(void* data);
    size_t data_size;
} event_t;

// 事件处理器
typedef void (*event_handler_t)(const event_t* event);

// 事件总线
typedef struct {
    event_handler_t* handlers[EVENT_CUSTOM_START + 100];  // 预留自定义事件空间
    bool initialized[EVENT_CUSTOM_START + 100];
    void* context;
} event_bus_t;

// 事件调度器
typedef struct {
    event_bus_t* bus;
    void* queue;
    bool running;
    void (*start)(struct event_scheduler* self);
    void (*stop)(struct event_scheduler* self);
    bool (*publish)(struct event_scheduler* self, const event_t* event);
    bool (*subscribe)(struct event_scheduler* self, event_type_t type, event_handler_t handler);
    bool (*unsubscribe)(struct event_scheduler* self, event_type_t type, event_handler_t handler);
} event_scheduler_t;

// 工厂函数
event_bus_t* create_event_bus(void);
void destroy_event_bus(event_bus_t* bus);
event_scheduler_t* create_event_scheduler(event_bus_t* bus);
void destroy_event_scheduler(event_scheduler_t* scheduler);

// 事件创建和发布
event_t* create_event(event_type_t type, void* data, size_t data_size, void (*destroy)(void*));
void destroy_event(event_t* event);
bool publish_event(event_scheduler_t* scheduler, const event_t* event);
bool subscribe_to_event(event_scheduler_t* scheduler, event_type_t type, event_handler_t handler);

#endif // EVENT_DRIVEN_H

// event_driven.c
#include "event_driven.h"
#include <stdlib.h>
#include <string.h>
#include <pthread.h>

// 事件总线实现
event_bus_t* create_event_bus(void) {
    event_bus_t* bus = malloc(sizeof(event_bus_t));
    if (!bus) return NULL;

    memset(bus, 0, sizeof(event_bus_t));
    return bus;
}

void destroy_event_bus(event_bus_t* bus) {
    if (bus) {
        free(bus);
    }
}

// 事件调度器实现
struct event_scheduler {
    event_bus_t* bus;
    pthread_t thread;
    pthread_mutex_t mutex;
    pthread_cond_t cond;
    event_t** event_queue;
    size_t queue_size;
    size_t queue_capacity;
    bool running;
    void (*start)(struct event_scheduler* self);
    void (*stop)(struct event_scheduler* self);
    bool (*publish)(struct event_scheduler* self, const event_t* event);
    bool (*subscribe)(struct event_scheduler* self, event_type_t type, event_handler_t handler);
    bool (*unsubscribe)(struct event_scheduler* self, event_type_t type, event_handler_t handler);
};

static void* event_thread_func(void* arg) {
    event_scheduler_t* scheduler = (event_scheduler_t*)arg;

    while (scheduler->running) {
        pthread_mutex_lock(&scheduler->mutex);

        while (scheduler->queue_size == 0 && scheduler->running) {
            pthread_cond_wait(&scheduler->cond, &scheduler->mutex);
        }

        if (!scheduler->running) {
            pthread_mutex_unlock(&scheduler->mutex);
            break;
        }

        // 获取事件
        event_t* event = scheduler->event_queue[0];

        // 移动队列
        for (size_t i = 0; i < scheduler->queue_size - 1; i++) {
            scheduler->event_queue[i] = scheduler->event_queue[i + 1];
        }
        scheduler->queue_size--;

        pthread_mutex_unlock(&scheduler->mutex);

        // 处理事件
        if (event->type < sizeof(scheduler->bus->handlers) / sizeof(event_handler_t) &&
            scheduler->bus->handlers[event->type]) {
            scheduler->bus->handlers[event->type](event);
        }

        destroy_event(event);
    }

    return NULL;
}

static void scheduler_start(event_scheduler_t* self) {
    if (!self || self->running) return;

    self->running = true;
    pthread_create(&self->thread, NULL, event_thread_func, self);
}

static void scheduler_stop(event_scheduler_t* self) {
    if (!self || !self->running) return;

    self->running = false;
    pthread_cond_signal(&self->cond);
    pthread_join(self->thread, NULL);
}

static bool scheduler_publish(event_scheduler_t* self, const event_t* event) {
    if (!self || !event || !self->running) return false;

    pthread_mutex_lock(&self->mutex);

    // 扩展队列
    if (self->queue_size >= self->queue_capacity) {
        size_t new_capacity = self->queue_capacity == 0 ? 16 : self->queue_capacity * 2;
        event_t** new_queue = realloc(self->event_queue, new_capacity * sizeof(event_t*));
        if (!new_queue) {
            pthread_mutex_unlock(&self->mutex);
            return false;
        }

        self->event_queue = new_queue;
        self->queue_capacity = new_capacity;
    }

    // 添加事件到队列
    self->event_queue[self->queue_size++] = (event_t*)event;
    pthread_cond_signal(&self->cond);

    pthread_mutex_unlock(&self->mutex);
    return true;
}

static bool scheduler_subscribe(event_scheduler_t* self, event_type_t type, event_handler_t handler) {
    if (!self || !handler || type >= sizeof(self->bus->handlers) / sizeof(event_handler_t)) {
        return false;
    }

    self->bus->handlers[type] = handler;
    self->bus->initialized[type] = true;
    return true;
}

static bool scheduler_unsubscribe(event_scheduler_t* self, event_type_t type, event_handler_t handler) {
    if (!self || type >= sizeof(self->bus->handlers) / sizeof(event_handler_t)) {
        return false;
    }

    if (self->bus->handlers[type] == handler) {
        self->bus->handlers[type] = NULL;
        self->bus->initialized[type] = false;
        return true;
    }

    return false;
}

event_scheduler_t* create_event_scheduler(event_bus_t* bus) {
    if (!bus) return NULL;

    event_scheduler_t* scheduler = malloc(sizeof(event_scheduler_t));
    if (!scheduler) return NULL;

    scheduler->bus = bus;
    scheduler->running = false;
    scheduler->queue_size = 0;
    scheduler->queue_capacity = 0;
    scheduler->event_queue = NULL;

    pthread_mutex_init(&scheduler->mutex, NULL);
    pthread_cond_init(&scheduler->cond, NULL);

    scheduler->start = scheduler_start;
    scheduler->stop = scheduler_stop;
    scheduler->publish = scheduler_publish;
    scheduler->subscribe = scheduler_subscribe;
    scheduler->unsubscribe = scheduler_unsubscribe;

    return scheduler;
}

void destroy_event_scheduler(event_scheduler_t* scheduler) {
    if (!scheduler) return;

    if (scheduler->running) {
        scheduler->stop(scheduler);
    }

    // 清理剩余事件
    for (size_t i = 0; i < scheduler->queue_size; i++) {
        destroy_event(scheduler->event_queue[i]);
    }

    free(scheduler->event_queue);
    pthread_mutex_destroy(&scheduler->mutex);
    pthread_cond_destroy(&scheduler->cond);
    free(scheduler);
}

// 事件创建
event_t* create_event(event_type_t type, void* data, size_t data_size, void (*destroy)(void*)) {
    event_t* event = malloc(sizeof(event_t));
    if (!event) return NULL;

    event->type = type;
    event->data = data;
    event->destroy = destroy;
    event->data_size = data_size;

    return event;
}

void destroy_event(event_t* event) {
    if (!event) return;

    if (event->destroy && event->data) {
        event->destroy(event->data);
    }
    free(event);
}

// 使用示例
// user_events.h
#ifndef USER_EVENTS_H
#define USER_EVENTS_H

#include "event_driven.h"

// 用户创建事件数据
typedef struct {
    int user_id;
    char username[64];
    char email[128];
} user_created_data_t;

// 用户更新事件数据
typedef struct {
    int user_id;
    char old_username[64];
    char new_username[64];
} user_updated_data_t;

// 事件工厂
event_t* create_user_created_event(int user_id, const char* username, const char* email);
event_t* create_user_updated_event(int user_id, const char* old_username, const char* new_username);

// 事件处理器
void handle_user_created(const event_t* event);
void handle_user_updated(const event_t* event);

#endif // USER_EVENTS_H

// user_events.c
#include "user_events.h"
#include <stdlib.h>
#include <string.h>

static void destroy_user_created_data(void* data) {
    free(data);
}

event_t* create_user_created_event(int user_id, const char* username, const char* email) {
    user_created_data_t* data = malloc(sizeof(user_created_data_t));
    if (!data) return NULL;

    data->user_id = user_id;
    strncpy(data->username, username ? username : "", sizeof(data->username) - 1);
    strncpy(data->email, email ? email : "", sizeof(data->email) - 1);
    data->username[sizeof(data->username) - 1] = '\0';
    data->email[sizeof(data->email) - 1] = '\0';

    return create_event(EVENT_USER_CREATED, data, sizeof(user_created_data_t), destroy_user_created_data);
}

static void destroy_user_updated_data(void* data) {
    free(data);
}

event_t* create_user_updated_event(int user_id, const char* old_username, const char* new_username) {
    user_updated_data_t* data = malloc(sizeof(user_updated_data_t));
    if (!data) return NULL;

    data->user_id = user_id;
    strncpy(data->old_username, old_username ? old_username : "", sizeof(data->old_username) - 1);
    strncpy(data->new_username, new_username ? new_username : "", sizeof(data->new_username) - 1);
    data->old_username[sizeof(data->old_username) - 1] = '\0';
    data->new_username[sizeof(data->new_username) - 1] = '\0';

    return create_event(EVENT_USER_UPDATED, data, sizeof(user_updated_data_t), destroy_user_updated_data);
}

void handle_user_created(const event_t* event) {
    if (!event || event->type != EVENT_USER_CREATED || !event->data) return;

    user_created_data_t* data = (user_created_data_t*)event->data;
    printf("User created event: ID=%d, Username=%s, Email=%s\n",
           data->user_id, data->username, data->email);

    // 可以在这里发送邮件、更新缓存等
}

void handle_user_updated(const event_t* event) {
    if (!event || event->type != EVENT_USER_UPDATED || !event->data) return;

    user_updated_data_t* data = (user_updated_data_t*)event->data;
    printf("User updated event: ID=%d, Old=%s, New=%s\n",
           data->user_id, data->old_username, data->new_username);

    // 可以在这里更新搜索索引、发送通知等
}
```

## 4. 代码组织与构建系统

### 4.1 项目结构设计

```
my_project/
├── src/
│   ├── core/                    # 核心模块
│   │   ├── common/
│   │   │   ├── types.h          # 通用类型定义
│   │   │   ├── utils.h          # 工具函数
│   │   │   ├── memory.h         # 内存管理
│   │   │   └── logging.h        # 日志系统
│   │   ├── container/
│   │   │   ├── list.h           # 链表
│   │   │   ├── hash_table.h     # 哈希表
│   │   │   ├── vector.h         # 动态数组
│   │   │   └── queue.h          # 队列
│   │   └── algorithm/
│   │       ├── sort.h           # 排序算法
│   │       ├── search.h         # 搜索算法
│   │       └── string.h         # 字符串算法
│   ├── modules/                 # 功能模块
│   │   ├── database/
│   │   │   ├── database.h
│   │   │   ├── database.c
│   │   │   └── sqlite/
│   │   ├── network/
│   │   │   ├── network.h
│   │   │   ├── network.c
│   │   │   └── http/
│   │   ├── ui/
│   │   │   ├── ui.h
│   │   │   ├── ui.c
│   │   │   └── widgets/
│   │   └── business/
│   │       ├── order/
│   │       ├── customer/
│   │       └── payment/
│   ├── platform/               # 平台相关
│   │   ├── linux/
│   │   ├── windows/
│   │   └── macos/
│   └── app/                     # 应用程序
│       ├── main.c
│       ├── config/
│       └── resources/
├── include/                     # 公共头文件
│   ├── my_project/
│   │   ├── core/
│   │   ├── modules/
│   │   └── platform/
├── tests/                       # 测试代码
│   ├── unit/
│   ├── integration/
│   └── performance/
├── docs/                        # 文档
│   ├── api/
│   ├── design/
│   └── user/
├── build/                       # 构建输出
├── third_party/                 # 第三方库
├── tools/                       # 构建工具
├── CMakeLists.txt               # CMake配置
├── Makefile                     # Make配置
└── README.md                    # 项目说明
```

### 4.2 构建系统配置

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.10)
project(MyProject VERSION 1.0.0 LANGUAGES C)

# 设置C标准
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)

# 编译选项
if(CMAKE_C_COMPILER_ID STREQUAL "GNU" OR CMAKE_C_COMPILER_ID STREQUAL "Clang")
    add_compile_options(-Wall -Wextra -Werror)
    add_compile_options(-O2)
    add_compile_options(-g)
endif()

# 查找依赖
find_package(Threads REQUIRED)
find_package(PkgConfig REQUIRED)

# 平台特定设置
if(WIN32)
    add_definitions(-DWIN32_LEAN_AND_MEAN)
    add_definitions(-DNOMINMAX)
elseif(UNIX)
    add_definitions(-D_POSIX_C_SOURCE=200112L)
endif()

# 包含目录
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include)
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/src)

# 第三方库
add_subdirectory(third_party)

# 核心库
add_library(core STATIC
    src/core/common/types.c
    src/core/common/utils.c
    src/core/common/memory.c
    src/core/common/logging.c
    src/core/container/list.c
    src/core/container/hash_table.c
    src/core/container/vector.c
    src/core/algorithm/sort.c
    src/core/algorithm/search.c
    src/core/algorithm/string.c
)

target_link_libraries(core Threads::Threads)

# 数据库模块
add_library(database STATIC
    src/modules/database/database.c
    src/modules/database/sqlite/sqlite_wrapper.c
)

target_link_libraries(database core)

# 网络模块
add_library(network STATIC
    src/modules/network/network.c
    src/modules/network/http/http_client.c
)

target_link_libraries(network core)

# UI模块
add_library(ui STATIC
    src/modules/ui/ui.c
    src/modules/ui/widgets/button.c
    src/modules/ui/widgets/window.c
)

target_link_libraries(ui core)

# 业务模块
add_library(business STATIC
    src/modules/business/order/order_service.c
    src/modules/business/customer/customer_service.c
    src/modules/business/payment/payment_service.c
)

target_link_libraries(business core database)

# 可执行文件
add_executable(my_app
    src/app/main.c
    src/app/config/config_loader.c
    src/app/resources/resource_manager.c
)

target_link_libraries(my_app core database network ui business)

# 安装规则
install(TARGETS my_app DESTINATION bin)
install(DIRECTORY include/ DESTINATION include)
install(FILES README.md DESTINATION share/doc/my_app)

# 测试
enable_testing()
add_subdirectory(tests)

# 打包配置
set(CPACK_PACKAGE_NAME "MyProject")
set(CPACK_PACKAGE_VERSION "1.0.0")
set(CPACK_PACKAGE_DESCRIPTION "My C Project")
set(CPACK_PACKAGE_CONTACT "your.email@example.com")
include(CPack)
```

### 4.3 模块间依赖管理

```c
// dependency_manager.h - 依赖管理器
#ifndef DEPENDENCY_MANAGER_H
#define DEPENDENCY_MANAGER_H

#include <stdbool.h>

// 模块依赖
typedef struct {
    const char* module_name;
    const char* version;
    const char** dependencies;
    size_t dependency_count;
} module_dependency_t;

// 模块接口
typedef struct {
    const char* name;
    const char* version;
    bool (*init)(void);
    void (*cleanup)(void);
    const module_dependency_t* dependencies;
    size_t dependency_count;
} module_t;

// 依赖管理器
typedef struct {
    module_t** modules;
    size_t module_count;
    size_t module_capacity;
    bool (*register_module)(struct dependency_manager* self, const module_t* module);
    bool (*unregister_module)(struct dependency_manager* self, const char* name);
    bool (*resolve_dependencies)(struct dependency_manager* self);
    bool (*initialize_modules)(struct dependency_manager* self);
    void (*cleanup_modules)(struct dependency_manager* self);
} dependency_manager_t;

// 工厂函数
dependency_manager_t* create_dependency_manager(void);
void destroy_dependency_manager(dependency_manager_t* manager);

// 模块注册宏
#define REGISTER_MODULE(manager, module) \
    (manager)->register_module(manager, module)

// 模块定义宏
#define DEFINE_MODULE(name, version, init_func, cleanup_func, ...) \
    static const module_dependency_t name##_dependencies[] = { __VA_ARGS__ }; \
    static const module_t name##_module = { \
        .name = #name, \
        .version = version, \
        .init = init_func, \
        .cleanup = cleanup_func, \
        .dependencies = name##_dependencies, \
        .dependency_count = sizeof(name##_dependencies) / sizeof(module_dependency_t) \
    }; \
    __attribute__((constructor)) \
    static void name##_register(void) { \
        dependency_manager_t* dm = get_dependency_manager(); \
        if (dm) { \
            dm->register_module(dm, &name##_module); \
        } \
    }

#endif // DEPENDENCY_MANAGER_H

// dependency_manager.c
#include "dependency_manager.h"
#include <stdlib.h>
#include <string.h>
#include <stdio.h>

static dependency_manager_t* g_dependency_manager = NULL;

dependency_manager_t* get_dependency_manager(void) {
    if (!g_dependency_manager) {
        g_dependency_manager = create_dependency_manager();
    }
    return g_dependency_manager;
}

static bool manager_register_module(dependency_manager_t* self, const module_t* module) {
    if (!self || !module) return false;

    // 检查模块是否已注册
    for (size_t i = 0; i < self->module_count; i++) {
        if (strcmp(self->modules[i]->name, module->name) == 0) {
            return false;
        }
    }

    // 扩展容量
    if (self->module_count >= self->module_capacity) {
        size_t new_capacity = self->module_capacity == 0 ? 16 : self->module_capacity * 2;
        module_t** new_modules = realloc(self->modules, new_capacity * sizeof(module_t*));
        if (!new_modules) return false;

        self->modules = new_modules;
        self->module_capacity = new_capacity;
    }

    // 添加模块
    self->modules[self->module_count++] = (module_t*)module;
    printf("Module registered: %s v%s\n", module->name, module->version);

    return true;
}

static bool manager_unregister_module(dependency_manager_t* self, const char* name) {
    if (!self || !name) return false;

    for (size_t i = 0; i < self->module_count; i++) {
        if (strcmp(self->modules[i]->name, name) == 0) {
            // 调用清理函数
            if (self->modules[i]->cleanup) {
                self->modules[i]->cleanup();
            }

            // 移除模块
            for (size_t j = i; j < self->module_count - 1; j++) {
                self->modules[j] = self->modules[j + 1];
            }
            self->module_count--;

            printf("Module unregistered: %s\n", name);
            return true;
        }
    }

    return false;
}

static bool manager_resolve_dependencies(dependency_manager_t* self) {
    if (!self) return false;

    // 简化的拓扑排序
    bool resolved = true;
    size_t iteration_count = 0;
    const size_t max_iterations = self->module_count * 2;

    while (resolved && iteration_count < max_iterations) {
        resolved = false;
        iteration_count++;

        for (size_t i = 0; i < self->module_count; i++) {
            module_t* module = self->modules[i];
            if (!module) continue;

            // 检查依赖是否已满足
            bool dependencies_satisfied = true;
            for (size_t j = 0; j < module->dependency_count; j++) {
                const module_dependency_t* dep = &module->dependencies[j];
                bool dependency_found = false;

                for (size_t k = 0; k < self->module_count; k++) {
                    if (self->modules[k] &&
                        strcmp(self->modules[k]->name, dep->module_name) == 0) {
                        dependency_found = true;
                        break;
                    }
                }

                if (!dependency_found) {
                    dependencies_satisfied = false;
                    break;
                }
            }

            if (dependencies_satisfied) {
                // 标记为已解析
                module = NULL;
                resolved = true;
            }
        }
    }

    return iteration_count < max_iterations;
}

static bool manager_initialize_modules(dependency_manager_t* self) {
    if (!self) return false;

    for (size_t i = 0; i < self->module_count; i++) {
        if (self->modules[i] && self->modules[i]->init) {
            if (!self->modules[i]->init()) {
                fprintf(stderr, "Failed to initialize module: %s\n", self->modules[i]->name);
                return false;
            }
            printf("Module initialized: %s\n", self->modules[i]->name);
        }
    }

    return true;
}

static void manager_cleanup_modules(dependency_manager_t* self) {
    if (!self) return;

    // 反向清理
    for (size_t i = self->module_count; i > 0; i--) {
        if (self->modules[i - 1] && self->modules[i - 1]->cleanup) {
            self->modules[i - 1]->cleanup();
            printf("Module cleaned up: %s\n", self->modules[i - 1]->name);
        }
    }
}

dependency_manager_t* create_dependency_manager(void) {
    dependency_manager_t* manager = malloc(sizeof(dependency_manager_t));
    if (!manager) return NULL;

    manager->modules = NULL;
    manager->module_count = 0;
    manager->module_capacity = 0;
    manager->register_module = manager_register_module;
    manager->unregister_module = manager_unregister_module;
    manager->resolve_dependencies = manager_resolve_dependencies;
    manager->initialize_modules = manager_initialize_modules;
    manager->cleanup_modules = manager_cleanup_modules;

    return manager;
}

void destroy_dependency_manager(dependency_manager_t* manager) {
    if (manager) {
        manager->cleanup_modules(manager);
        free(manager->modules);
        free(manager);
    }
}
```

## 5. 设计模式在C语言中的应用

### 5.1 工厂模式

```c
// factory_pattern.h
#ifndef FACTORY_PATTERN_H
#define FACTORY_PATTERN_H

#include <stdbool.h>

// 产品接口
typedef struct {
    void (*operation)(struct product* self);
    void (*destroy)(struct product* self);
    const char* type;
} product_t;

// 工厂接口
typedef struct {
    product_t* (*create_product)(struct factory* self, const char* type);
    void (*destroy)(struct factory* self);
} factory_t;

// 具体产品
typedef struct {
    product_t base;
    int value;
} concrete_product_a_t;

typedef struct {
    product_t base;
    double value;
} concrete_product_b_t;

// 具体工厂
typedef struct {
    factory_t base;
    // 工厂特定数据
} concrete_factory_t;

// 工厂函数
factory_t* create_concrete_factory(void);
product_t* create_product_a(void);
product_t* create_product_b(void);

#endif // FACTORY_PATTERN_H

// factory_pattern.c
#include "factory_pattern.h"
#include <stdlib.h>
#include <stdio.h>

// 具体产品A实现
static void product_a_operation(product_t* self) {
    concrete_product_a_t* product = (concrete_product_a_t*)self;
    printf("Product A operation with value: %d\n", product->value);
}

static void product_a_destroy(product_t* self) {
    free(self);
}

product_t* create_product_a(void) {
    concrete_product_a_t* product = malloc(sizeof(concrete_product_a_t));
    if (!product) return NULL;

    product->base.operation = product_a_operation;
    product->base.destroy = product_a_destroy;
    product->base.type = "ProductA";
    product->value = 42;

    return (product_t*)product;
}

// 具体产品B实现
static void product_b_operation(product_t* self) {
    concrete_product_b_t* product = (concrete_product_b_t*)self;
    printf("Product B operation with value: %.2f\n", product->value);
}

static void product_b_destroy(product_t* self) {
    free(self);
}

product_t* create_product_b(void) {
    concrete_product_b_t* product = malloc(sizeof(concrete_product_b_t));
    if (!product) return NULL;

    product->base.operation = product_b_operation;
    product->base.destroy = product_b_destroy;
    product->base.type = "ProductB";
    product->value = 3.14;

    return (product_t*)product;
}

// 具体工厂实现
static product_t* factory_create_product(factory_t* self, const char* type) {
    if (!type) return NULL;

    if (strcmp(type, "A") == 0) {
        return create_product_a();
    } else if (strcmp(type, "B") == 0) {
        return create_product_b();
    }

    return NULL;
}

static void factory_destroy(factory_t* self) {
    free(self);
}

factory_t* create_concrete_factory(void) {
    concrete_factory_t* factory = malloc(sizeof(concrete_factory_t));
    if (!factory) return NULL;

    factory->base.create_product = factory_create_product;
    factory->base.destroy = factory_destroy;

    return (factory_t*)factory;
}
```

### 5.2 单例模式

```c
// singleton_pattern.h
#ifndef SINGLETON_PATTERN_H
#define SINGLETON_PATTERN_H

#include <stdbool.h>
#include <pthread.h>

// 单例接口
typedef struct {
    void (*method)(struct singleton* self);
    int value;
} singleton_t;

// 获取单例实例
singleton_t* get_singleton(void);
void destroy_singleton(void);

#endif // SINGLETON_PATTERN_H

// singleton_pattern.c
#include "singleton_pattern.h"
#include <stdlib.h>
#include <stdio.h>

static singleton_t* g_singleton = NULL;
static pthread_mutex_t g_singleton_mutex = PTHREAD_MUTEX_INITIALIZER;

static void singleton_method(singleton_t* self) {
    printf("Singleton method called, value: %d\n", self->value);
}

singleton_t* get_singleton(void) {
    if (g_singleton) {
        return g_singleton;
    }

    pthread_mutex_lock(&g_singleton_mutex);

    if (!g_singleton) {
        singleton_t* singleton = malloc(sizeof(singleton_t));
        if (singleton) {
            singleton->method = singleton_method;
            singleton->value = 100;
            g_singleton = singleton;
            printf("Singleton created\n");
        }
    }

    pthread_mutex_unlock(&g_singleton_mutex);
    return g_singleton;
}

void destroy_singleton(void) {
    pthread_mutex_lock(&g_singleton_mutex);

    if (g_singleton) {
        free(g_singleton);
        g_singleton = NULL;
        printf("Singleton destroyed\n");
    }

    pthread_mutex_unlock(&g_singleton_mutex);
}
```

### 5.3 观察者模式

```c
// observer_pattern.h
#ifndef OBSERVER_PATTERN_H
#define OBSERVER_PATTERN_H

#include <stdbool.h>

// 观察者接口
typedef struct observer observer_t;
typedef struct subject subject_t;

struct observer {
    void (*update)(observer_t* self, subject_t* subject, void* data);
    void* context;
};

// 主题接口
struct subject {
    void (*attach)(subject_t* self, observer_t* observer);
    void (*detach)(subject_t* self, observer_t* observer);
    void (*notify)(subject_t* self, void* data);
    observer_t** observers;
    size_t observer_count;
    size_t observer_capacity;
};

// 工厂函数
subject_t* create_subject(void);
void destroy_subject(subject_t* subject);
observer_t* create_observer(void (*update)(observer_t*, subject_t*, void*), void* context);
void destroy_observer(observer_t* observer);

#endif // OBSERVER_PATTERN_H

// observer_pattern.c
#include "observer_pattern.h"
#include <stdlib.h>
#include <stdio.h>

static void subject_attach(subject_t* self, observer_t* observer) {
    if (!self || !observer) return;

    // 扩展容量
    if (self->observer_count >= self->observer_capacity) {
        size_t new_capacity = self->observer_capacity == 0 ? 8 : self->observer_capacity * 2;
        observer_t** new_observers = realloc(self->observers, new_capacity * sizeof(observer_t*));
        if (!new_observers) return;

        self->observers = new_observers;
        self->observer_capacity = new_capacity;
    }

    // 添加观察者
    self->observers[self->observer_count++] = observer;
    printf("Observer attached to subject\n");
}

static void subject_detach(subject_t* self, observer_t* observer) {
    if (!self || !observer) return;

    // 查找并移除观察者
    for (size_t i = 0; i < self->observer_count; i++) {
        if (self->observers[i] == observer) {
            // 移动数组
            for (size_t j = i; j < self->observer_count - 1; j++) {
                self->observers[j] = self->observers[j + 1];
            }
            self->observer_count--;
            printf("Observer detached from subject\n");
            return;
        }
    }
}

static void subject_notify(subject_t* self, void* data) {
    if (!self) return;

    // 通知所有观察者
    for (size_t i = 0; i < self->observer_count; i++) {
        if (self->observers[i] && self->observers[i]->update) {
            self->observers[i]->update(self->observers[i], self, data);
        }
    }
}

subject_t* create_subject(void) {
    subject_t* subject = malloc(sizeof(subject_t));
    if (!subject) return NULL;

    subject->attach = subject_attach;
    subject->detach = subject_detach;
    subject->notify = subject_notify;
    subject->observers = NULL;
    subject->observer_count = 0;
    subject->observer_capacity = 0;

    return subject;
}

void destroy_subject(subject_t* subject) {
    if (subject) {
        free(subject->observers);
        free(subject);
    }
}

observer_t* create_observer(void (*update)(observer_t*, subject_t*, void*), void* context) {
    observer_t* observer = malloc(sizeof(observer_t));
    if (!observer) return NULL;

    observer->update = update;
    observer->context = context;

    return observer;
}

void destroy_observer(observer_t* observer) {
    if (observer) {
        free(observer);
    }
}
```

## 6. 最佳实践与注意事项

### 6.1 架构设计原则

1. **单一职责原则**：每个模块应该只有一个变化的原因
2. **开放封闭原则**：对扩展开放，对修改封闭
3. **里氏替换原则**：子类应该能够替换其基类
4. **接口隔离原则**：不应该强迫客户端依赖它们不需要的接口
5. **依赖倒置原则**：高层模块不应该依赖低层模块，两者都应该依赖抽象

### 6.2 模块化最佳实践

1. **明确边界**：每个模块应该有清晰的接口和实现边界
2. **依赖管理**：使用依赖注入减少模块间耦合
3. **接口设计**：设计稳定、向后兼容的接口
4. **错误处理**：统一错误处理机制
5. **资源管理**：明确资源的生命周期和所有权

### 6.3 性能考虑

1. **减少依赖开销**：避免过度的抽象层次
2. **批量操作**：减少模块间通信频率
3. **缓存友好**：优化数据布局和访问模式
4. **延迟初始化**：按需创建和初始化模块
5. **内存管理**：合理使用内存池和对象池

## 7. 总结

C语言虽然没有面向对象的语法特性，但通过良好的架构设计，我们仍然可以构建出结构清晰、模块化、可扩展的系统。关键在于：

1. **接口设计**：使用函数指针和结构体实现多态
2. **模块化**：清晰的模块边界和职责分离
3. **依赖管理**：使用依赖注入和服务定位器
4. **设计模式**：适配C语言的经典设计模式
5. **代码组织**：合理的项目结构和构建系统

通过这些技术，我们可以在C语言中实现现代软件架构的理念，构建出高质量、可维护的系统。记住，好的架构比好的代码更重要，它能够适应需求变化，降低维护成本，提高开发效率。