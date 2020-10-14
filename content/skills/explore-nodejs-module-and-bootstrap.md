---
title: "Node.js探索——启动与模块加载"
date: 2020-10-08T20:17:56+08:00
draft: false
comments: true
toc: false
tags: [Node.js]
---

版本：14.13.1

`NOde.js`的实际入口是`src\node_main.cc`，这里以`Windows`为例。

```c++
// node_main.cc
int wmain(int argc, wchar_t* wargv[]) {
  // Windows Server 2012 (not R2) is supported until 10/10/2023, so we allow it
  // to run in the experimental support tier.
  // ...
  argv[argc] = nullptr;
  return node::Start(argc, argv);
}
```

`Windows`下调用`wmain`类`unix`下调用`main`，调用`Start`启动`Node.js`

```c++
// src/node.cc
int Start(int argc, char** argv) {
    // 初始化
  InitializationResult result = InitializeOncePerProcess(argc, argv);
    // ...
}
```

开始初始化，调用`InitializeOncePerProcess`。

在`V8`初始化之前，会执行一步初始化操作

```c++
// node.cc
{
    result.exit_code =
        InitializeNodeWithArgs(&(result.args), &(result.exec_args), &errors);
    // ...
    }
  }
```

初始化函数`InitializeNodeWithArgs`主要是调用`RegisterBuiltinModules`注册内建模块，处理一些`Node.js`配置及参数相关的内容。

调用`RegisterBuiltinModules`注册核心模块，其中定义了一个宏，调用`NODE_BUILTIN_MODULES(`

```c++
// node_binding.cc
void RegisterBuiltinModules() {
#define V(modname) _register_##modname();
  NODE_BUILTIN_MODULES(V)
#undef V
}
```

展开

```c++
#define NODE_BUILTIN_MODULES(V)                                                \
  NODE_BUILTIN_STANDARD_MODULES(V)                                             \
  NODE_BUILTIN_OPENSSL_MODULES(V)                                              \
  NODE_BUILTIN_QUIC_MODULES(V)                                                 \
  NODE_BUILTIN_ICU_MODULES(V)                                                  \
  NODE_BUILTIN_PROFILER_MODULES(V)                                             \
  NODE_BUILTIN_DTRACE_MODULES(V)
```

最终展开后如下

```c++
// node_binding.cc
void RegisterBuiltinModules() {
	_register_async_wrap();
    _register_block_list();
    // _register_<module name>
}
```

这些单独模块的注册函数定义在每个`C++`模块文件里，比如`async_wrap.cc`的结尾。

```c++
// async_wrap.cc
NODE_MODULE_CONTEXT_AWARE_INTERNAL(async_wrap, node::AsyncWrap::Initialize)
```

其中`NODE_MODULE_CONTEXT_AWARE_INTERNAL`定义如下

```c++
// node_binding.cc
#define NODE_MODULE_CONTEXT_AWARE_INTERNAL(modname, regfunc)                   \
  NODE_MODULE_CONTEXT_AWARE_CPP(modname, regfunc, nullptr, NM_F_INTERNAL)
```

展开

```c++
// node_binding.cc
#define NODE_MODULE_CONTEXT_AWARE_CPP(modname, regfunc, priv, flags)           \
  static node::node_module _module = {                                         \
      NODE_MODULE_VERSION,                                                     \
      flags,                                                                   \
      nullptr,                                                                 \
      __FILE__,                                                                \
      nullptr,                                                                 \
      (node::addon_context_register_func)(regfunc),                            \
      NODE_STRINGIFY(modname),                                                 \
      priv,                                                                    \
      nullptr};                                                                \
  void _register_##modname() { node_module_register(&_module); }
```

每个`C++`模块的结尾，都注册了一个`_register_##modname`的函数，在注册`C++`模块时，经过一系列的宏展开，把所有模块的注册函数都执行一遍，其最终执行`node_module_register`。

`C++`模块的结构以及`node_module_register`实现如下

```c++
// node.h
struct node_module {
  int nm_version;
  unsigned int nm_flags;
  void* nm_dso_handle;
  const char* nm_filename;
  node::addon_register_func nm_register_func;
  node::addon_context_register_func nm_context_register_func;
  const char* nm_modname;
  void* nm_priv;
  struct node_module* nm_link;
};
```

```c++
// node_binding.cc
extern "C" void node_module_register(void* m) {
  struct node_module* mp = reinterpret_cast<struct node_module*>(m);

  if (mp->nm_flags & NM_F_INTERNAL) {
    mp->nm_link = modlist_internal;
    modlist_internal = mp;
  } else if (!node_is_initialized) {
    // "Linked" modules are included as part of the node project.
    // Like builtins they are registered *before* node::Init runs.
    mp->nm_flags = NM_F_LINKED;
    mp->nm_link = modlist_linked;
    modlist_linked = mp;
  } else {
    thread_local_modpending = mp;
  }
}
```

多个`C++`模块组成一个链表，并使用头插法连接。

通过这里可以知道，内建模块是注册到一个链表上，以供后续使用。

内建模块注册完毕以及一系列准备结束后，在函数的结尾，初始化`V8`。

```c++
// node.cc
per_process::v8_platform.Initialize(
    per_process::cli_options->v8_thread_pool_size);
V8::Initialize();
performance::performance_v8_start = PERFORMANCE_NOW();
per_process::v8_initialized = true;
```

初始化结束，根据`InitializeOncePerProcess(argc, argv)`返回的结果(包括初始化的`V8`)创建`NodeMainInstance`实例

```c++
// node.cc
Isolate::CreateParams params;
const std::vector<size_t>* indexes = nullptr;
const EnvSerializeInfo* env_info = nullptr;
bool force_no_snapshot =
    per_process::cli_options->per_isolate->no_node_snapshot;
if (!force_no_snapshot) {
  v8::StartupData* blob = NodeMainInstance::GetEmbeddedSnapshotBlob();
  if (blob != nullptr) {
    params.snapshot_blob = blob;
    indexes = NodeMainInstance::GetIsolateDataIndexes();
    env_info = NodeMainInstance::GetEnvSerializeInfo();
  }
}
uv_loop_configure(uv_default_loop(), UV_METRICS_IDLE_TIME);
NodeMainInstance main_instance(&params,
                               uv_default_loop(),
                               per_process::v8_platform.Platform(),
                               result.args,
                               result.exec_args,
                               indexes);
result.exit_code = main_instance.Run(env_info);
```

初始化`NodeMainInstance`

```cpp
// src/node_main_instance.cc
NodeMainInstance::NodeMainInstance(
    Isolate::CreateParams* params,
    uv_loop_t* event_loop,
    MultiIsolatePlatform* platform,
    const std::vector<std::string>& args,
    const std::vector<std::string>& exec_args,
    const std::vector<size_t>* per_isolate_data_indexes)
    : args_(args),
      exec_args_(exec_args),
      array_buffer_allocator_(ArrayBufferAllocator::Create()),
      isolate_(nullptr),
      platform_(platform),
      isolate_data_(nullptr),
      owns_isolate_(true) {
  params->array_buffer_allocator = array_buffer_allocator_.get();
  deserialize_mode_ = per_isolate_data_indexes != nullptr;
  if (deserialize_mode_) {
    const std::vector<intptr_t>& external_references =
        CollectExternalReferences();
    params->external_references = external_references.data();
  }

  isolate_ = Isolate::Allocate();
  CHECK_NOT_NULL(isolate_);
  // Register the isolate on the platform before the isolate gets initialized,
  // so that the isolate can access the platform during initialization.
  platform->RegisterIsolate(isolate_, event_loop);
  SetIsolateCreateParamsForNode(params);
  Isolate::Initialize(isolate_, *params);

  // If the indexes are not nullptr, we are not deserializing
  CHECK_IMPLIES(deserialize_mode_, params->external_references != nullptr);
  isolate_data_ = std::make_unique<IsolateData>(isolate_,
                                                event_loop,
                                                platform,
                                                array_buffer_allocator_.get(),
                                                per_isolate_data_indexes);
  IsolateSettings s;
  SetIsolateMiscHandlers(isolate_, s);
  if (!deserialize_mode_) {
    // If in deserialize mode, delay until after the deserialization is
    // complete.
    SetIsolateErrorHandlers(isolate_, s);
  }
}
```

可见`event loop`的创建是在这里完成的，之后它创建并初始化了一个`Isolate`实例，并在`Isolate`初始化之前，将`event loop`和`Isolate`注册到`platform`上。

完成一系列初始化之后，调用`NodeMainInstance::Run`。

这一步至关重要，`Environment`的创建，`V8`环境的启动以及`event loop`的执行都是在这里完成的。

```cpp
// node_main_instance.cc
int NodeMainInstance::Run(const EnvSerializeInfo* env_info) {
  Locker locker(isolate_);
  Isolate::Scope isolate_scope(isolate_);
  HandleScope handle_scope(isolate_);

  int exit_code = 0;
  DeleteFnPtr<Environment, FreeEnvironment> env =
      CreateMainEnvironment(&exit_code, env_info);

  CHECK_NOT_NULL(env);
  {
    Context::Scope context_scope(env->context());

    if (exit_code == 0) {
      LoadEnvironment(env.get());

      env->set_trace_sync_io(env->options()->trace_sync_io);

      {
        SealHandleScope seal(isolate_);
        bool more;
        env->performance_state()->Mark(
            node::performance::NODE_PERFORMANCE_MILESTONE_LOOP_START);
        do {
          uv_run(env->event_loop(), UV_RUN_DEFAULT);

          per_process::v8_platform.DrainVMTasks(isolate_);

          more = uv_loop_alive(env->event_loop());
          if (more && !env->is_stopping()) continue;

          if (!uv_loop_alive(env->event_loop())) {
            EmitBeforeExit(env.get());
          }

          // Emit `beforeExit` if the loop became alive either after emitting
          // event, or after running some callbacks.
          more = uv_loop_alive(env->event_loop());
        } while (more == true && !env->is_stopping());
        env->performance_state()->Mark(
            node::performance::NODE_PERFORMANCE_MILESTONE_LOOP_EXIT);
      }

      env->set_trace_sync_io(false);
      if (!env->is_stopping()) env->VerifyNoStrongBaseObjects();
      exit_code = EmitExit(env.get());
    }

    ResetStdio();

#if HAVE_INSPECTOR && defined(__POSIX__) && !defined(NODE_SHARED_MODE)
  struct sigaction act;
  memset(&act, 0, sizeof(act));
  for (unsigned nr = 1; nr < kMaxSignal; nr += 1) {
    if (nr == SIGKILL || nr == SIGSTOP || nr == SIGPROF)
      continue;
    act.sa_handler = (nr == SIGPIPE) ? SIG_IGN : SIG_DFL;
    CHECK_EQ(0, sigaction(nr, &act, nullptr));
  }
#endif

#if defined(LEAK_SANITIZER)
  __lsan_do_leak_check();
#endif
  }

  return exit_code;
}
```

首先根据`isolate`创建`Locker`，用来完成`isolate`隔离以及线程独占。

之后调用`CreateMainEnvironment`创建环境，初始化`env`，`Environment`可以理解为`Node.js`的运行环境。

这里做了大量的初始化工作，展开分析一下。

`CreateMainEnvironment`中创建了一个`Context`，然后初始化了一个`DeleteFnPtr`指针(本质上是` unique_ptr `)，之后通过这个指针初始化`Context`，之后调用`RunBootstrapping`，正式开始`Bootstrap`。

```c++
MaybeLocal<Value> Environment::RunBootstrapping() {
  EscapableHandleScope scope(isolate_);

  CHECK(!has_run_bootstrapping_code());

  if (BootstrapInternalLoaders().IsEmpty()) {
    return MaybeLocal<Value>();
  }

  Local<Value> result;
  if (!BootstrapNode().ToLocal(&result)) {
    return MaybeLocal<Value>();
  }

  // Make sure that no request or handle is created during bootstrap -
  // if necessary those should be done in pre-execution.
  // Usually, doing so would trigger the checks present in the ReqWrap and
  // HandleWrap classes, so this is only a consistency check.
  CHECK(req_wrap_queue()->IsEmpty());
  CHECK(handle_wrap_queue()->IsEmpty());

  set_has_run_bootstrapping_code(true);

  return scope.Escape(result);
}
```

除了一些检查逻辑外，`RunBootstrapping`主要是调用了`BootstrapInternalLoaders`和`BootstrapNode`，其中第一步`BootstrapInternalLoaders`完成了内建模块和核心`JS`模块的加载。

```c++
// node.cc
MaybeLocal<Value> Environment::BootstrapInternalLoaders() {
  EscapableHandleScope scope(isolate_);

  // Create binding loaders
  std::vector<Local<String>> loaders_params = {
      process_string(),
      FIXED_ONE_BYTE_STRING(isolate_, "getLinkedBinding"),
      FIXED_ONE_BYTE_STRING(isolate_, "getInternalBinding"),
      primordials_string()};
  std::vector<Local<Value>> loaders_args = {
      process_object(),
      NewFunctionTemplate(binding::GetLinkedBinding)
          ->GetFunction(context())
          .ToLocalChecked(),
      NewFunctionTemplate(binding::GetInternalBinding)
          ->GetFunction(context())
          .ToLocalChecked(),
      primordials()};

  // Bootstrap internal loaders
  Local<Value> loader_exports;
  if (!ExecuteBootstrapper(
           this, "internal/bootstrap/loaders", &loaders_params, &loaders_args)
           .ToLocal(&loader_exports)) {
    return MaybeLocal<Value>();
  }
  CHECK(loader_exports->IsObject());
  Local<Object> loader_exports_obj = loader_exports.As<Object>();
  Local<Value> internal_binding_loader =
      loader_exports_obj->Get(context(), internal_binding_string())
          .ToLocalChecked();
  CHECK(internal_binding_loader->IsFunction());
  set_internal_binding_loader(internal_binding_loader.As<Function>());
  Local<Value> require =
      loader_exports_obj->Get(context(), require_string()).ToLocalChecked();
  CHECK(require->IsFunction());
  set_native_module_require(require.As<Function>());

  return scope.Escape(loader_exports);
}
```

根据注释，流程分为了两步：`Create binding loaders`和`Bootstrap internal loaders`，第一步准备了两个参数`loaders_params`和`loaders_args`供第二步使用，两者的内容是相同的，区别是`params`是字符串形式的，`args`是真正的对象。其中，

- `process`参数就是`Node.js`中的全局对象`process`
- `getLinkedBinding`和`getInternalBinding`实现`JS`端获取`C++`模块，两者都是`V8`的`FunctionTemplate`，实现`JS`函数与`C++`函数的关联，以便`JS`调用`C++`函数。
- ` primordials`是`JS`常用的内置对象，`Error, Symbol, String, Boolean`等。

之前说到，内建模块在` InitializeNodeWithArgs `中被注册到一个链表上，然后有一个`GetInternalBinding`方法，用来取出内建模块。

```c++
// node_binding.cc
void GetInternalBinding(const FunctionCallbackInfo<Value>& args) {
  Environment* env = Environment::GetCurrent(args);

  CHECK(args[0]->IsString());

  Local<String> module = args[0].As<String>();
  node::Utf8Value module_v(env->isolate(), module);
  Local<Object> exports;

  node_module* mod = FindModule(modlist_internal, *module_v, NM_F_INTERNAL);
  if (mod != nullptr) {
    exports = InitModule(env, mod, module);
  } else if (!strcmp(*module_v, "constants")) {
    exports = Object::New(env->isolate());
    CHECK(
        exports->SetPrototype(env->context(), Null(env->isolate())).FromJust());
    DefineConstants(env->isolate(), exports);
  } else if (!strcmp(*module_v, "natives")) {
    exports = native_module::NativeModuleEnv::GetSourceObject(env->context());
    // Legacy feature: process.binding('natives').config contains stringified
    // config.gypi
    CHECK(exports
              ->Set(env->context(),
                    env->config_string(),
                    native_module::NativeModuleEnv::GetConfigString(
                        env->isolate()))
              .FromJust());
  } else {
    char errmsg[1024];
    snprintf(errmsg, sizeof(errmsg), "No such module: %s", *module_v);
    return THROW_ERR_INVALID_MODULE(env, errmsg);
  }

  args.GetReturnValue().Set(exports);
}
```

首先通过一个`FindModule`函数找到模块，

```c++
// node_binding.cc
inline struct node_module* FindModule(struct node_module* list,
                                      const char* name,
                                      int flag) {
  struct node_module* mp;

  for (mp = list; mp != nullptr; mp = mp->nm_link) {
    if (strcmp(mp->nm_modname, name) == 0) break;
  }

  CHECK(mp == nullptr || (mp->nm_flags & flag) != 0);
  return mp;
}
```

之后通过`InitModule`初始化模块，

```c++
// node_binding.cc
static Local<Object> InitModule(Environment* env,
                                node_module* mod,
                                Local<String> module) {
  // Internal bindings don't have a "module" object, only exports.
  Local<Function> ctor = env->binding_data_ctor_template()
                             ->GetFunction(env->context())
                             .ToLocalChecked();
  Local<Object> exports = ctor->NewInstance(env->context()).ToLocalChecked();
  CHECK_NULL(mod->nm_register_func);
  CHECK_NOT_NULL(mod->nm_context_register_func);
  Local<Value> unused = Undefined(env->isolate());
  mod->nm_context_register_func(exports, unused, env->context(), mod->nm_priv);
  return exports;
}
```

初始化时，调用传入的`mod`的`nm_context_register_func`方法注册模块，这里以`async_wrap`模块为例

```c++
// async_wrap.cc
void AsyncWrap::Initialize(Local<Object> target,
                           Local<Value> unused,
                           Local<Context> context,
                           void* priv) {
  Environment* env = Environment::GetCurrent(context);
  Isolate* isolate = env->isolate();
  HandleScope scope(isolate);

  env->SetMethod(target, "setupHooks", SetupHooks);
  env->SetMethod(target, "setCallbackTrampoline", SetCallbackTrampoline);
  env->SetMethod(target, "pushAsyncContext", PushAsyncContext);
  env->SetMethod(target, "popAsyncContext", PopAsyncContext);
  env->SetMethod(target, "executionAsyncResource", ExecutionAsyncResource);
  env->SetMethod(target, "clearAsyncIdStack", ClearAsyncIdStack);
  env->SetMethod(target, "queueDestroyAsyncId", QueueDestroyAsyncId);
  env->SetMethod(target, "enablePromiseHook", EnablePromiseHook);
  env->SetMethod(target, "disablePromiseHook", DisablePromiseHook);
  env->SetMethod(target, "registerDestroyHook", RegisterDestroyHook);
  // ...
}
```

调用初始化函数后，传入的`exports`上就挂载了`setupHooks`,`setCallbackTrampoline`等函数，之后返回`exports`。

至此，内建函数的装载就完成了。

参数准备完毕后，就调用`!ExecuteBootstrapper`执行`internal/bootstrap/loaders`，首先看一下`!ExecuteBootstrapper`方法。

```c++
// node.cc
MaybeLocal<Value> ExecuteBootstrapper(Environment* env,
                                      const char* id,
                                      std::vector<Local<String>>* parameters,
                                      std::vector<Local<Value>>* arguments) {
  EscapableHandleScope scope(env->isolate());
  MaybeLocal<Function> maybe_fn =
      NativeModuleEnv::LookupAndCompile(env->context(), id, parameters, env);

  Local<Function> fn;
  if (!maybe_fn.ToLocal(&fn)) {
    return MaybeLocal<Value>();
  }

  MaybeLocal<Value> result = fn->Call(env->context(),
                                      Undefined(env->isolate()),
                                      arguments->size(),
                                      arguments->data());

  // If there was an error during bootstrap, it must be unrecoverable
  // (e.g. max call stack exceeded). Clear the stack so that the
  // AsyncCallbackScope destructor doesn't fail on the id check.
  // There are only two ways to have a stack size > 1: 1) the user manually
  // called MakeCallback or 2) user awaited during bootstrap, which triggered
  // _tickCallback().
  if (result.IsEmpty()) {
    env->async_hooks()->clear_async_id_stack();
  }

  return scope.EscapeMaybe(result);
}
```

可以看到，这里调用`LookupAndCompile`， 将文件路径字符串传入当作`id`执行，并返回`maybe_fn`调用`call`执行真正意义上的第一个`JS`文件。

看一下`LookupAndCompile`，这里的主干代码由秋衣大佬做了修正，具体可以看[https://github.com/nodejs/node/pull/27160](https://github.com/nodejs/node/pull/27160)，就不看这个代理类的相关代码了，直接跳到`LookupAndCompile`。

```c++
// node_native_module.cc
MaybeLocal<Function> NativeModuleLoader::LookupAndCompile(
    Local<Context> context,
    const char* id,
    std::vector<Local<String>>* parameters,
    NativeModuleLoader::Result* result) {
  Isolate* isolate = context->GetIsolate();
  EscapableHandleScope scope(isolate);

  Local<String> source;
  if (!LoadBuiltinModuleSource(isolate, id).ToLocal(&source)) {
    return {};
  }

  std::string filename_s = std::string("node:") + id;
  Local<String> filename =
      OneByteString(isolate, filename_s.c_str(), filename_s.size());
  Local<Integer> line_offset = Integer::New(isolate, 0);
  Local<Integer> column_offset = Integer::New(isolate, 0);
  ScriptOrigin origin(filename, line_offset, column_offset, True(isolate));

  ScriptCompiler::CachedData* cached_data = nullptr;
  {
    // Note: The lock here should not extend into the
    // `CompileFunctionInContext()` call below, because this function may
    // recurse if there is a syntax error during bootstrap (because the fatal
    // exception handler is invoked, which may load built-in modules).
    Mutex::ScopedLock lock(code_cache_mutex_);
    auto cache_it = code_cache_.find(id);
    if (cache_it != code_cache_.end()) {
      // Transfer ownership to ScriptCompiler::Source later.
      cached_data = cache_it->second.release();
      code_cache_.erase(cache_it);
    }
  }

  const bool has_cache = cached_data != nullptr;
  ScriptCompiler::CompileOptions options =
      has_cache ? ScriptCompiler::kConsumeCodeCache
                : ScriptCompiler::kEagerCompile;
  ScriptCompiler::Source script_source(source, origin, cached_data);

  MaybeLocal<Function> maybe_fun =
      ScriptCompiler::CompileFunctionInContext(context,
                                               &script_source,
                                               parameters->size(),
                                               parameters->data(),
                                               0,
                                               nullptr,
                                               options);

  // This could fail when there are early errors in the native modules,
  // e.g. the syntax errors
  Local<Function> fun;
  if (!maybe_fun.ToLocal(&fun)) {
    // In the case of early errors, v8 is already capable of
    // decorating the stack for us - note that we use CompileFunctionInContext
    // so there is no need to worry about wrappers.
    return MaybeLocal<Function>();
  }

  // XXX(joyeecheung): this bookkeeping is not exactly accurate because
  // it only starts after the Environment is created, so the per_context.js
  // will never be in any of these two sets, but the two sets are only for
  // testing anyway.

  *result = (has_cache && !script_source.GetCachedData()->rejected)
                ? Result::kWithCache
                : Result::kWithoutCache;
  // Generate new cache for next compilation
  std::unique_ptr<ScriptCompiler::CachedData> new_cached_data(
      ScriptCompiler::CreateCodeCacheForFunction(fun));
  CHECK_NOT_NULL(new_cached_data);

  {
    Mutex::ScopedLock lock(code_cache_mutex_);
    // The old entry should've been erased by now so we can just emplace.
    // If another thread did the same thing in the meantime, that should not
    // be an issue.
    code_cache_.emplace(id, std::move(new_cached_data));
  }

  return scope.Escape(fun);
}
```

首先调用`LoadBuiltinModuleSource`加载文件代码

```c++
// node_native_module.cc
MaybeLocal<String> NativeModuleLoader::LoadBuiltinModuleSource(Isolate* isolate,
                                                               const char* id) {
#ifdef NODE_BUILTIN_MODULES_PATH
  std::string filename = OnDiskFileName(id);

  uv_fs_t req;
  uv_file file =
      uv_fs_open(nullptr, &req, filename.c_str(), O_RDONLY, 0, nullptr);
  CHECK_GE(req.result, 0);
  uv_fs_req_cleanup(&req);

  auto defer_close = OnScopeLeave([file]() {
    uv_fs_t close_req;
    CHECK_EQ(0, uv_fs_close(nullptr, &close_req, file, nullptr));
    uv_fs_req_cleanup(&close_req);
  });

  std::string contents;
  char buffer[4096];
  uv_buf_t buf = uv_buf_init(buffer, sizeof(buffer));

  while (true) {
    const int r =
        uv_fs_read(nullptr, &req, file, &buf, 1, contents.length(), nullptr);
    CHECK_GE(req.result, 0);
    uv_fs_req_cleanup(&req);
    if (r <= 0) {
      break;
    }
    contents.append(buf.base, r);
  }

  return String::NewFromUtf8(
      isolate, contents.c_str(), v8::NewStringType::kNormal, contents.length());
#else
  const auto source_it = source_.find(id);
  CHECK_NE(source_it, source_.end());
  return source_it->second.ToStringChecked(isolate);
#endif  // NODE_BUILTIN_MODULES_PATH
}
```

这里加载本地文件使用了`libuv`相关的函数，之后调用`CompileFunctionInContext`将之前的文件代码和含有`getLinkedBinding`等方法的参数包裹起来形成一个可执行函数，执行这个函数就等同于执行`internal/bootstrap/loaders`。

分析完成如何编译`internal/bootstrap/loaders`之后，进入这个文件看看。

注释写得很长很详细，这里概括一下。

这个文件创建了`native module`加载器，并且给其通过`binding loaders`调用内建函数的能立。而且指出这里和`CommonJS Module`和`ES Module`是不同的机制。`native module`就是指`lib/**/*.js`和`*deps/**/\*.js*`。所有核心模块都是通过`js2c.py`生成的`node_javascript.cc`编译到二进制文件中，因此调用时不需要消耗`I/O`。

文件首先定义了`process`上的`moduleLoadList`属性，用来表示已加载的模块。

```javascript
const moduleLoadList = [];
ObjectDefineProperty(process, 'moduleLoadList', {
  value: moduleLoadList,
  configurable: true,
  enumerable: true,
  writable: false
});
```

之后定义了`binding`和`_linkedBinding`方法，分别通过`internalBinding`方法和`getLinkedBinding`方法获取内建模块。

```javascript
process.binding = function binding(module) {
  module = String(module);
  // Deprecated specific process.binding() modules, but not all, allow
  // selective fallback to internalBinding for the deprecated ones.
  if (internalBindingWhitelist.has(module)) {
    return internalBinding(module);
  }
  // eslint-disable-next-line no-restricted-syntax
  throw new Error(`No such module: ${module}`);
};
process._linkedBinding = function _linkedBinding(module) {
  module = String(module);
  let mod = bindingObj[module];
  if (typeof mod !== 'object')
    mod = bindingObj[module] = getLinkedBinding(module);
  return mod;
};
```

之后创建了`NativeModule`类，提供了内部`JS`模块的抽象，用于加载被写入`node_javascript.cc`的`JS`模块。

之后解释了`require`的机制。

```javascript
const loaderExports = {
  internalBinding,
  NativeModule,
  require: nativeModuleRequire
};

function nativeModuleRequire(id) {
  if (id === loaderId) {
    return loaderExports;
  }

  const mod = NativeModule.map.get(id);
  // Can't load the internal errors module from here, have to use a raw error.
  // eslint-disable-next-line no-restricted-syntax
  if (!mod) throw new TypeError(`Missing internal module '${id}'`);
  return mod.compileForInternalLoader();
}
```

首先根据`id`找到模块，然后调用`compileForInternalLoader`通过`node_native_module_env.cc`的`CompileFunction`函数实现对模块的装裹，同时提供了6个参数，分别是`exports`,`require`,`module`,`process`,`internalBinding`,`primordials`。

至此，内建模块`built-in module`以及内部`JS`模块`native module`装载完毕。

第一步`BootstrapInternalLoaders`结束后，执行第二步`BootstrapNode`。

```c++
// node.cc
MaybeLocal<Value> Environment::BootstrapNode() {
  EscapableHandleScope scope(isolate_);

  Local<Object> global = context()->Global();
  // TODO(joyeecheung): this can be done in JS land now.
  global->Set(context(), FIXED_ONE_BYTE_STRING(isolate_, "global"), global)
      .Check();

  // process, require, internalBinding, primordials
  std::vector<Local<String>> node_params = {
      process_string(),
      require_string(),
      internal_binding_string(),
      primordials_string()};
  std::vector<Local<Value>> node_args = {
      process_object(),
      native_module_require(),
      internal_binding_loader(),
      primordials()};

  MaybeLocal<Value> result = ExecuteBootstrapper(
      this, "internal/bootstrap/node", &node_params, &node_args);

  if (result.IsEmpty()) {
    return scope.EscapeMaybe(result);
  }

  // TODO(joyeecheung): skip these in the snapshot building for workers.
  auto thread_switch_id =
      is_main_thread() ? "internal/bootstrap/switches/is_main_thread"
                       : "internal/bootstrap/switches/is_not_main_thread";
  result =
      ExecuteBootstrapper(this, thread_switch_id, &node_params, &node_args);

  if (result.IsEmpty()) {
    return scope.EscapeMaybe(result);
  }

  auto process_state_switch_id =
      owns_process_state()
          ? "internal/bootstrap/switches/does_own_process_state"
          : "internal/bootstrap/switches/does_not_own_process_state";
  result = ExecuteBootstrapper(
      this, process_state_switch_id, &node_params, &node_args);

  if (result.IsEmpty()) {
    return scope.EscapeMaybe(result);
  }

  Local<String> env_string = FIXED_ONE_BYTE_STRING(isolate_, "env");
  Local<Object> env_var_proxy;
  if (!CreateEnvVarProxy(context(), isolate_).ToLocal(&env_var_proxy) ||
      process_object()->Set(context(), env_string, env_var_proxy).IsNothing()) {
    return MaybeLocal<Value>();
  }

  return scope.EscapeMaybe(result);
}
```

首先在`context`上设置了一个`global`代理，它定义在`deps\v8\src\objects\js-objects.h`，并且可以通过在`Node`层调用`global`对象修改底层的`global`对象。

之后同样的，调用`ExecuteBootstrapper`，并将`env`，`internal/bootstrap/node`作为`id`，`node_params`以及`node_args`传入，不过这里参数有所改变，分别是`native_module_require`和`internal_binding_loader`。

`ExecuteBootstrapper`之前分析过了，这里直接看一下`lib\internal\bootstrap\node.js`。

同样的，这里注释很详细，大体概括一下：

```c++
// Hello, and welcome to hacking node.js!
//
// This file is invoked by `node::RunBootstrapping()` in `src/node.cc`, and is
// responsible for setting up node.js core before executing main scripts
// under `lib/internal/main/`.
//
// This file is expected not to perform any asynchronous operations itself
// when being executed - those should be done in either
// `lib/internal/bootstrap/pre_execution.js` or in main scripts. The majority
// of the code here focuses on setting up the global proxy and the process
// object in a synchronous manner.
// As special caution is given to the performance of the startup process,
// many dependencies are invoked lazily.
```

第一句很有趣，`Hello, and welcome to hacking node.js!`。

这个文件只做同步操作，并且指出处理异步操作的位置`lib/internal/bootstrap/pre_execution.js`，之后说明，此文件主要是设置`global`代理和`process`对象，并且该文件启动了主线程和工作线程，这个`worker threads`主要用在`CPU`密集的场景，并且每个`worker thread`拥有独立的`V8 ioslate`，`libuv`以及`event loop`，有机会再展开说。

之后启动了主线程以及工作线程，然后设置了`global`对象和`process`对象上的一些属性，包括`global.URL, global.clearInterval, global.setInterval`等等，最后设置了异常回调以及任务队列`TaskQueue`。

` CreateMainEnvironment `完成一大堆工作之后，调用`LoadEnvironment`。

```c++
// environment.cc
MaybeLocal<Value> LoadEnvironment(
    Environment* env,
    StartExecutionCallback cb,
    std::unique_ptr<InspectorParentHandle> removeme) {
  env->InitializeLibuv();
  env->InitializeDiagnostics();

  return StartExecution(env, cb);
}
```

`LoadEnvironment`首先是初始化`libuv`和异常诊断程序，然后执行启动`StartExecution`。

这个`StartExecution`函数实现的就是根据用户的启动参数，来以不同模式启动，并执行对应的启动文件。比如

```c++
// node.cc
std::string first_argv;
if (env->argv().size() > 1) {
  first_argv = env->argv()[1];
}
if (first_argv == "inspect") {
  return StartExecution(env, "internal/main/inspect");
}
if (per_process::cli_options->print_help) {
  return StartExecution(env, "internal/main/print_help");
}
if (env->options()->prof_process) {
  return StartExecution(env, "internal/main/prof_process");
}
// -e/--eval without -i/--interactive
if (env->options()->has_eval_string && !env->options()->force_repl) {
  return StartExecution(env, "internal/main/eval_string");
}
if (env->options()->syntax_check_only) {
  return StartExecution(env, "internal/main/check_syntax");
}
if (!first_argv.empty() && first_argv != "-") {
  return StartExecution(env, "internal/main/run_main_module");
}
if (env->options()->force_repl || uv_guess_handle(STDIN_FILENO) == UV_TTY) {
  return StartExecution(env, "internal/main/repl");
}
return StartExecution(env, "internal/main/eval_stdin");
```

其中`inspect`就是调试模式，下面也可以看到`repl`环境的启动模式。

而默认的不带参数的启动模式如下

```c++
if (!first_argv.empty() && first_argv != "-") {
  return StartExecution(env, "internal/main/run_main_module");
}
```

看这个文件之前，先分析`StartExecution`是如何执行这个文件的 

```c++
// node.cc
MaybeLocal<Value> StartExecution(Environment* env, const char* main_script_id) {
  EscapableHandleScope scope(env->isolate());
  CHECK_NOT_NULL(main_script_id);

  std::vector<Local<String>> parameters = {
      env->process_string(),
      env->require_string(),
      env->internal_binding_string(),
      env->primordials_string(),
      FIXED_ONE_BYTE_STRING(env->isolate(), "markBootstrapComplete")};

  std::vector<Local<Value>> arguments = {
      env->process_object(),
      env->native_module_require(),
      env->internal_binding_loader(),
      env->primordials(),
      env->NewFunctionTemplate(MarkBootstrapComplete)
          ->GetFunction(env->context())
          .ToLocalChecked()};

  return scope.EscapeMaybe(
      ExecuteBootstrapper(env, main_script_id, &parameters, &arguments));
}
```

发现还是一样，传入要执行的`JS`文件作为`ID`，然后老几样，`process, require, pprimordials, internal_binding_loader`，只是少了一个`getLinkedBinding`，多了一个`markBootstrapComplete`的标记而已，然后调用`ExecuteBootstrapper`。

之后看一下`lib\internal\main\run_main_module.js`。

```javascript
// lib\internal\main\run_main_module.js
'use strict';

const {
  prepareMainThreadExecution
} = require('internal/bootstrap/pre_execution');

prepareMainThreadExecution(true);

markBootstrapComplete();

// Note: this loads the module through the ESM loader if the module is
// determined to be an ES module. This hangs from the CJS module loader
// because we currently allow monkey-patching of the module loaders
// in the preloaded scripts through require('module').
// runMain here might be monkey-patched by users in --require.
// XXX: the monkey-patchability here should probably be deprecated.
require('internal/modules/cjs/loader').Module.runMain(process.argv[1]);
```

代码很短，主要是执行`prepareMainThreadExecution`，做最后初始化工作，并标记`Bootstrap`已完成。

然后通过引入`'internal/modules/cjs/loader`，调用`runMain`方法执行要启动的`JS`文件，就比如`node index.js`里的`index.js`。

其中有两步需要注意。

```javascript
// pre_execution.js
function initializeCJSLoader() {
  const CJSLoader = require('internal/modules/cjs/loader');
  CJSLoader.Module._initPaths();
  // TODO(joyeecheung): deprecate this in favor of a proper hook?
  CJSLoader.Module.runMain =
    require('internal/modules/run_main').executeUserEntryPoint;
}
// ...
function loadPreloadModules() {
  // For user code, we preload modules if `-r` is passed
  const preloadModules = getOptionValue('--require');
  if (preloadModules && preloadModules.length > 0) {
    const {
      Module: {
        _preloadModules
      },
    } = require('internal/modules/cjs/loader');
    _preloadModules(preloadModules);
  }
}
```

首先是初始化`CJSLoader`，它设置了`CJSLoader`的`runMain`方法实际上为`'internal/modules/run_main'`的`executeUserEntryPoint`。

然后是`loadPreloadModules`，这里通过`CJSLoader`的`_preloadModules`方法加载传入的`require`。

这里看一下`_preloadModules`。

```javascript
// lib\internal\modules\cjs\loader.js
Module._preloadModules = function(requests) {
  if (!ArrayIsArray(requests))
    return;

  // Preloaded modules have a dummy parent module which is deemed to exist
  // in the current working directory. This seeds the search path for
  // preloaded modules.
  const parent = new Module('internal/preload', null);
  try {
    parent.paths = Module._nodeModulePaths(process.cwd());
  } catch (e) {
    if (e.code !== 'ENOENT') {
      throw e;
    }
  }
  for (let n = 0; n < requests.length; n++)
    parent.require(requests[n]);
};
```

这里构造了一个被认为存在于当前工作路径的假的模块，这为预加载的模块提供索引路径，就是说如果我们引用`--require`的预加载模块，会通过这个路径进行查找。

OK，回去看` executeUserEntryPoint `，启动文件就是通过这个方法执行。

```javascript
// lib\internal\modules\run_main.js
'use strict';

const CJSLoader = require('internal/modules/cjs/loader');
const { Module, toRealPath, readPackageScope } = CJSLoader;
const { getOptionValue } = require('internal/options');
const path = require('path');

function resolveMainPath(main) {
  // Note extension resolution for the main entry point can be deprecated in a
  // future major.
  // Module._findPath is monkey-patchable here.
  let mainPath = Module._findPath(path.resolve(main), null, true);
  if (!mainPath)
    return;

  const preserveSymlinksMain = getOptionValue('--preserve-symlinks-main');
  if (!preserveSymlinksMain)
    mainPath = toRealPath(mainPath);

  return mainPath;
}

function shouldUseESMLoader(mainPath) {
  const userLoader = getOptionValue('--experimental-loader');
  if (userLoader)
    return true;
  const esModuleSpecifierResolution =
    getOptionValue('--experimental-specifier-resolution');
  if (esModuleSpecifierResolution === 'node')
    return true;
  // Determine the module format of the main
  if (mainPath && mainPath.endsWith('.mjs'))
    return true;
  if (!mainPath || mainPath.endsWith('.cjs'))
    return false;
  const pkg = readPackageScope(mainPath);
  return pkg && pkg.data.type === 'module';
}

function runMainESM(mainPath) {
  const esmLoader = require('internal/process/esm_loader');
  const { pathToFileURL } = require('internal/url');
  handleMainPromise(esmLoader.loadESM((ESMLoader) => {
    const main = path.isAbsolute(mainPath) ?
      pathToFileURL(mainPath).href : mainPath;
    return ESMLoader.import(main);
  }));
}

function handleMainPromise(promise) {
  // Handle a Promise from running code that potentially does Top-Level Await.
  // In that case, it makes sense to set the exit code to a specific non-zero
  // value if the main code never finishes running.
  function handler() {
    if (process.exitCode === undefined)
      process.exitCode = 13;
  }
  process.on('exit', handler);
  return promise.finally(() => process.off('exit', handler));
}

// For backwards compatibility, we have to run a bunch of
// monkey-patchable code that belongs to the CJS loader (exposed by
// `require('module')`) even when the entry point is ESM.
function executeUserEntryPoint(main = process.argv[1]) {
  const resolvedMain = resolveMainPath(main);
  const useESMLoader = shouldUseESMLoader(resolvedMain);
  if (useESMLoader) {
    runMainESM(resolvedMain || main);
  } else {
    // Module._load is the monkey-patchable CJS module loader.
    Module._load(main, null, true);
  }
}

module.exports = {
  executeUserEntryPoint,
  handleMainPromise,
};
```

这里的`executeUserEntryPoint`方法，首先是找到文件路径并判断模块类别，这里介绍一下`cjs`的导入方法。

首先会调用`Module`的`_load`方法。

```javascript
// lib\internal\modules\cjs\loader.js
Module._load = function(request, parent, isMain) {
  let relResolveCacheIdentifier;
  if (parent) {
    debug('Module._load REQUEST %s parent: %s', request, parent.id);
    // Fast path for (lazy loaded) modules in the same directory. The indirect
    // caching is required to allow cache invalidation without changing the old
    // cache key names.
    relResolveCacheIdentifier = `${parent.path}\x00${request}`;
    const filename = relativeResolveCache[relResolveCacheIdentifier];
    if (filename !== undefined) {
      const cachedModule = Module._cache[filename];
      if (cachedModule !== undefined) {
        updateChildren(parent, cachedModule, true);
        if (!cachedModule.loaded)
          return getExportsForCircularRequire(cachedModule);
        return cachedModule.exports;
      }
      delete relativeResolveCache[relResolveCacheIdentifier];
    }
  }

  const filename = Module._resolveFilename(request, parent, isMain);

  const cachedModule = Module._cache[filename];
  if (cachedModule !== undefined) {
    updateChildren(parent, cachedModule, true);
    if (!cachedModule.loaded) {
      const parseCachedModule = cjsParseCache.get(cachedModule);
      if (!parseCachedModule || parseCachedModule.loaded)
        return getExportsForCircularRequire(cachedModule);
      parseCachedModule.loaded = true;
    } else {
      return cachedModule.exports;
    }
  }

  const mod = loadNativeModule(filename, request);
  if (mod && mod.canBeRequiredByUsers) return mod.exports;

  // Don't call updateChildren(), Module constructor already does.
  const module = cachedModule || new Module(filename, parent);

  if (isMain) {
    process.mainModule = module;
    module.id = '.';
  }

  Module._cache[filename] = module;
  if (parent !== undefined) {
    relativeResolveCache[relResolveCacheIdentifier] = filename;
  }

  let threw = true;
  try {
    // Intercept exceptions that occur during the first tick and rekey them
    // on error instance rather than module instance (which will immediately be
    // garbage collected).
    if (getSourceMapsEnabled()) {
      try {
        module.load(filename);
      } catch (err) {
        rekeySourceMap(Module._cache[filename], err);
        throw err; /* node-do-not-add-exception-line */
      }
    } else {
      module.load(filename);
    }
    threw = false;
  } finally {
    if (threw) {
      delete Module._cache[filename];
      if (parent !== undefined) {
        delete relativeResolveCache[relResolveCacheIdentifier];
        const children = parent && parent.children;
        if (ArrayIsArray(children)) {
          const index = children.indexOf(module);
          if (index !== -1) {
            children.splice(index, 1);
          }
        }
      }
    } else if (module.exports &&
               !isProxy(module.exports) &&
               ObjectGetPrototypeOf(module.exports) ===
                 CircularRequirePrototypeWarningProxy) {
      ObjectSetPrototypeOf(module.exports, ObjectPrototype);
    }
  }

  return module.exports;
};
```

如果当前模块有父模块，会根据父模块找到当前模块的文件名。之后会检查模块是否有缓存，有则直接返回。下一步，会判断模块是否为`NativeModule`，如果是则直接导出。之后只会是自定义模块了，首先会根据文件名实例化一个`Module`对象并做缓存处理，然后调用`load`方法。

```javascript
// lib\internal\modules\cjs\loader.js
Module.prototype.load = function(filename) {
  debug('load %j for module %j', filename, this.id);

  assert(!this.loaded);
  this.filename = filename;
  this.paths = Module._nodeModulePaths(path.dirname(filename));

  const extension = findLongestRegisteredExtension(filename);
  // allow .mjs to be overridden
  if (filename.endsWith('.mjs') && !Module._extensions['.mjs']) {
    throw new ERR_REQUIRE_ESM(filename);
  }
  Module._extensions[extension](this, filename);
  this.loaded = true;

  const ESMLoader = asyncESM.ESMLoader;
  // Create module entry at load time to snapshot exports correctly
  const exports = this.exports;
  // Preemptively cache
  if ((module?.module === undefined ||
       module.module.getStatus() < kEvaluated) &&
      !ESMLoader.cjsCache.has(this))
    ESMLoader.cjsCache.set(this, exports);
};
```

这里会根据不同的文件扩展名调用` Module._extension `方法，分别是`.js .json .node`。

```javascript
// lib\internal\modules\cjs\loader.js
Module._extensions['.js'] = function(module, filename) {
  if (filename.endsWith('.js')) {
    const pkg = readPackageScope(filename);
    // Function require shouldn't be used in ES modules.
    if (pkg && pkg.data && pkg.data.type === 'module') {
      const parent = moduleParentCache.get(module);
      const parentPath = parent && parent.filename;
      const packageJsonPath = path.resolve(pkg.path, 'package.json');
      throw new ERR_REQUIRE_ESM(filename, parentPath, packageJsonPath);
    }
  }
  // If already analyzed the source, then it will be cached.
  const cached = cjsParseCache.get(module);
  let content;
  if (cached && cached.source) {
    content = cached.source;
    cached.source = undefined;
  } else {
    content = fs.readFileSync(filename, 'utf8');
  }
  module._compile(content, filename);
};
```

对于`.js`模块，会调用`fs.readFileSync`读取文件内容，有缓存则直接读缓存，然后调用`_compile`编译。

```javascript
// lib\internal\modules\cjs\loader.js
Module.prototype._compile = function(content, filename) {
  let moduleURL;
  let redirects;
  if (policy?.manifest) {
    moduleURL = pathToFileURL(filename);
    redirects = policy.manifest.getDependencyMapper(moduleURL);
    policy.manifest.assertIntegrity(moduleURL, content);
  }

  maybeCacheSourceMap(filename, content, this);
  const compiledWrapper = wrapSafe(filename, content, this);

  let inspectorWrapper = null;
  if (getOptionValue('--inspect-brk') && process._eval == null) {
    if (!resolvedArgv) {
      // We enter the repl if we're not given a filename argument.
      if (process.argv[1]) {
        try {
          resolvedArgv = Module._resolveFilename(process.argv[1], null, false);
        } catch {
          // We only expect this codepath to be reached in the case of a
          // preloaded module (it will fail earlier with the main entry)
          assert(ArrayIsArray(getOptionValue('--require')));
        }
      } else {
        resolvedArgv = 'repl';
      }
    }

    // Set breakpoint on module start
    if (resolvedArgv && !hasPausedEntry && filename === resolvedArgv) {
      hasPausedEntry = true;
      inspectorWrapper = internalBinding('inspector').callAndPauseOnStart;
    }
  }
  const dirname = path.dirname(filename);
  const require = makeRequireFunction(this, redirects);
  let result;
  const exports = this.exports;
  const thisValue = exports;
  const module = this;
  if (requireDepth === 0) statCache = new Map();
  if (inspectorWrapper) {
    result = inspectorWrapper(compiledWrapper, thisValue, exports,
                              require, module, filename, dirname);
  } else {
    result = compiledWrapper.call(thisValue, exports, require, module,
                                  filename, dirname);
  }
  hasLoadedAnyUserCJSModule = true;
  if (requireDepth === 0) statCache = null;
  return result;
};
```

首先会调用`wrapSafe`方法给源代码包装，传入`6`个参数`exports, require, mudole, __filename, __dirname`形成一个新函数，如果已经包装过，就直接调用`vm.runInThisContext`运行这个函数，否则就调用内建模块` contextify `里的`compileFunction`函数编译，最终返回一个可执行的函数体。

```javascript
// lib\internal\modules\cjs\loader.js
let wrap = function(script) {
  return Module.wrapper[0] + script + Module.wrapper[1];
};

const wrapper = [
  '(function (exports, require, module, __filename, __dirname) { ',
  '\n});'
];

function wrapSafe(filename, content, cjsModuleInstance) {
// ...
  try {
    compiled = compileFunction(
      content,
      filename,
      0,
      0,
      undefined,
      false,
      undefined,
      [],
      [
        'exports',
        'require',
        'module',
        '__filename',
        '__dirname',
      ]
    );
  } catch (err) {
    if (process.mainModule === cjsModuleInstance)
      enrichCJSError(err);
    throw err;
  }

  const { callbackMap } = internalBinding('module_wrap');
  callbackMap.set(compiled.cacheKey, {
    importModuleDynamically: async (specifier) => {
      const loader = asyncESM.ESMLoader;
      return loader.import(specifier, normalizeReferrerURL(filename));
    }
  });

  return compiled.function;
}
```

回到刚才的`_compile`，在包装编译完成之后，会将本体的`exports, require, module, filename, dirname`作为参数传入刚才编译后的函数，最终返回执行的结果。

这里看一下`require`方法的具体实现

```javascript
// // lib\internal\modules\cjs\loader.js
Module.prototype.require = function(id) {
  validateString(id, 'id');
  if (id === '') {
    throw new ERR_INVALID_ARG_VALUE('id', id,
                                    'must be a non-empty string');
  }
  requireDepth++;
  try {
    return Module._load(id, this, /* isMain */ false);
  } finally {
    requireDepth--;
  }
};
```

至此， 环境初始化完成，内建模块，`native`模块以及自定义模块加载完成，主文件编译完毕，在刚刚初始化的`Environment`上，启动`event loop`，以及`libuv`。

```c++
do {
  uv_run(env->event_loop(), UV_RUN_DEFAULT);
  per_process::v8_platform.DrainVMTasks(isolate_);
  more = uv_loop_alive(env->event_loop());
  if (more && !env->is_stopping()) continue;
  if (!uv_loop_alive(env->event_loop())) {
    EmitBeforeExit(env.get());
  }
  more = uv_loop_alive(env->event_loop());
} while (more == true && !env->is_stopping());
```
