# 1 structure of node source code
- `deps/v8`
- `deps/uv/src` :about system call
  - `unix`
  - `win`
  - [others]
- `src`
  - `node.cc`

# 2 what happens when `require("fs")`
- `internal/main/run_main_module`[anonymous]
- `intermal/modules/run_main.js/executeUserEntryPoint`
- (if-else)Module._load(main,null,true)  (Internal/modules/cjs/loader)
- module.load(filename)
- module._extension[]()
- module._compile(content, filename)[------anonymous]
- [inspectorWrapper]
- makeRequireFunction - > mod.require(path)
- Module.require
- Module._load()
- JS`open`

# 3 when libuv api are invoked
## 3.1 program start
- `node::Start`->`InitializeOncePerProcess`->`uv_setup_args`
  - **location**: `uv/src/unix/aix.c`
  - **mutex**
    - `init_process_title_mutex_once`->`uv_mutex_init`
    - `uv_once`
    - `uv_mutex_lock` and `uv_mutex_unlock`
  - **malloc**:`new_argv = uv__malloc`
- `node::Start`->`uv_loop_configure`
  - **location**:uv/src/uv_common.c
  - uv_loop_configure
  - uv_default_loop
- `node::Start`->`Run->InitializeLibuv()`

## 3.2 execute user's JS code
### 3.2.1 when and how?
>- location: `src/node.cc`
>- function: `MaybeLocal<Value> StartExecution(env, main_script_id)` -> `ExecuteBootstrapper(env, id, arguments)`
- func parameters
  - `Environment* env`
  - `id` => `main_script_id` => `"internal/main/run_main_module"`
  - `std::vector<Local<Value>>* arguments`
    - `env->process`
    - `env->builtin_module_require`
    - `env->internal_binding_loader`
    - `env->primordials`
- variables
  - `Local<Function> fn`
    - `Local`:related to GC
    - `Function`: V8_EXPORT, a JS function object  
  - `MaybeLocal<value> result`
    - `MaybeLocal`: a wrapper around `Local`
    - `Value`:superclass of all JS values and objects
- execute
  - `MaybeLocal<Value> result = fn->Call()`
    - ```C
      fn->Call(env->context(),
               Undefined(env->isolate()),
               arguments->size(),
               arguments->data());
      ```
    - ```C
      has_pending_exception = !ToLocal<Value>(
        i::Execution::Call(isolate, self, recv_obj, argc, args), &result);
      ```
- jump into user's JS code
  - `fn->Call`: execute user code
    - `require`
    - `open`


### 3.2.2 src of `uv_fs_open()`
#### 3.2.2.1 execute JS code: `fs.open()`
  - `uv_fs_open()`:deps/uv/src/unix/fs.c
  - ```C
    int uv_fs_open(uv_loop_t* loop,
               uv_fs_t* req,
               const char* path,
               int flags,
               int mode,
               uv_fs_cb cb) {
      INIT(OPEN);
      PATH;
      req->flags = flags;
      req->mode = mode;
      POST;
    }
    ```
  - ```C
    #deifne INIT(subtype)
      do{
        if(req == NULL)
          return UV_EINVAL;
        UV_REQ_INIT(req, UV_FS);
        req->fs_type = UV_FS_ ## subtype;
        req->ptr = NULL;
        req->loop = loop;
        req->path = NULL;
        req->new_path = NULL;
        req->bufs = NULL;
        req->cb = cb;
      }
      while(0)
    ```
    - `UV_[xx]_MAP`:specify the mapping relationship by `#define`
      - `UV_EINVAL`:UV_ERRNO_MAP
      - `UV_FS`:UV_REQ_TYPE_MAP
    - `OPEN`-`subtype`-`req->fs_type`:invoke various system function, such as `open`, `close`
  - ```C
    #define POST
      do {
        if (cb != NULL) {
          uv__req_register(loop, req);
          uv__work_submit(
            loop,
            &req->work_req,
            UV__WORK_FAST_IO,
            uv__fs_work,
            uv__fs_done
          );
          return 0;
        }
        else {
          uv__fs_work(&req->work_req);
          return req->result;
        }
      }
      while(0)
    ```
    - `uv__work_submit()`
      - ```C
        void uv__work_submit(uv_loop_t* loop,
                             struct uv__work* w,
                             enum uv__work_kind kind,
                             void (*work)(struct uv__work* w),
                             void (*done)(struct uv__work* w, int status)) {
          uv_once(&once, init_once);
          w->loop = loop;
          w->work = work;
          w->done = done;
          post(&w->wq, kind);
        }
        ```
    - `uv__fs_work`
      - execute req acorrding to fs_type

#### 3.2.2.2 `req` in node
> `uv_req_s` is abstract base class of all request
> ```C
> struct uv_req_s {
>   UV_REQ_FIELDS
> };
> ```
- `uv_req_t`-`uv_req_s`: `UV_REQ_FIELDS`
  - `uv_fs_t`-`uv_fs_s`
  - `uv_write_t`-`uv_write_s`

## 3.3 event loop
> **function**:`int uv_run(uv_loop_t* loop, uv_run_mode mode)`
> **location**:`deps/uv/src/unix/core.c`
### 3.3.1 libuv data structure
- **location**:`uv/include/uv.h`
#### 3.3.1.1 `uv_loop_t loop`->`uv_loop_s`
```C
struct uv_loop_s {
  /* User data - use this for whatever. */
  void* data;
  /* Loop reference counting. */
  unsigned int active_handles;
  void* handle_queue[2];
  union {
    void* unused[2];
    unsigned int count;
  } active_reqs;
  /* Internal flag to signal loop stop. */
  unsigned int stop_flag;
  UV_LOOP_PRIVATE_FIELDS
};
```
- `active_handles` and `active_reqs` define `uv__loop_alive(loop)`
- `UV_LOOP_PRIVATE_FIELDS` related to  `loop->time` and `uv__update_time(loop)`
> ***note***: structure `UV_[xx]_[xx]_FIELDS` defined in libuv
#### 3.3.1.2 `uv_run_mode`: 
```C
typedef enum {
  UV_RUN_DEFAULT = 0,
  UV_RUN_ONCE,
  UV_RUN_NOWAIT
} uv_run_mode;
```
#### 3.3.1.3 `heap_node` and `heap`
```C
struct heap_node {
  struct heap_node* left;
  struct heap_node* right;
  struct heap_node* parent;
};
```
```C
struct heap {
  struct heap_node* min;
  unsigned int nelts;
};
```
#### 3.3.1.4 `uv_timer_t`->`uv_timer_s`
```C
struct uv_timer_s {
  UV_HANDLE_FIELDS
  UV_TIMER_PRIVATE_FIELDS
};
```
#### 3.3.1.5 `uv__io_t`->`uv__io_s`
```C
struct uv__io_s {
  uv__io_cb cb;
  void* pending_queue[2];
  void* watcher_queue[2];
  unsigned int pevents; /* Pending event mask i.e. mask at next tick. */
  unsigned int events;  /* Current event mask. */
  int fd;
  UV_IO_PRIVATE_PLATFORM_FIELDS
};
```

### 3.3.2 libuv functions
### 3.3.2.1 `uv__run_pending(uv_loop_t* loop)`

### 3.3.3 loop and env
- `loop = env->event_loop()`
  - ```C
    DeleteFnPtr<Environment, FreeEnvironment> env = CreateMainEnvironment(&exit_code);
    ```
- in `CreateMainEnvironment()`

# 4 Threads in node.js
- node::WorkerThreadsTaskRunner
# Other
- event define: handle->timer_cb
- struct
- ```C
  #define UV_HANDLE_FIELDS    \
    /* public */              \
    void* data;               \
    /* read-only */           \
    uv_loop_t* loop;          \
    uv_handle_type type;      \
    /* private */             \
    uv_close_cb close_cb;     \
    void* handle_queue[2];    \
    union {                   \
      int fd;                 \
      void* reserved[4];      \
    } u;                      \
    UV_HANDLE_PRIVATE_FIELDS  \
  ```

- uv.h
  - uv_check_s -> UV_CHECK_PRIVATE_FIELDS



  


