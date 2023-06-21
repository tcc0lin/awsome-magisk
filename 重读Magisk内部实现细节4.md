### 前言
Magisk内部实现细节的第四篇，在前两篇着重讲了Magisk的三个重要功能的两个---su以及hide，这篇就来分析下最后一个重要功能---resetprop，这三个功能Magisk也分别导出了三个可执行文件
```c
// native/jni/core/applets.cpp
static main_fun applet_main[] = { su_client_main, resetprop_main, magiskhide_main, nullptr };

static int call_applet(int argc, char *argv[]) {
    // Applets
    string_view base = basename(argv[0]);
    for (int i = 0; applet_names[i]; ++i) {
        if (base == applet_names[i]) {
            // 根据可执行文件的名称执行具体的类方法
            return (*applet_main[i])(argc, argv);
        }
    }
#if ENABLE_INJECT
    if (str_starts(base, "app_process")) {
        return app_process_main(argc, argv);
    }
#endif
    fprintf(stderr, "%s: applet not found\n", base.data());
    return 1;
}
```

### 一、Magisk Resetprop入口
```c
// native/jni/resetprop/resetprop.cpp
int resetprop_main(int argc, char *argv[]) {
    log_cb.d = [](auto fmt, auto ap) -> int { return verbose ? vfprintf(stderr, fmt, ap) : 0; };

    bool prop_svc = true;
    bool persist = false;
    char *argv0 = argv[0];

    --argc;
    ++argv;

    // Parse flags and -- options
    while (argc && argv[0][0] == '-') {
        for (int idx = 1; true; ++idx) {
            switch (argv[0][idx]) {
            case '-':
                if (strcmp(argv[0], "--file") == 0 && argc == 2) {
                    load_prop_file(argv[1], prop_svc);
                    return 0;
                } else if (strcmp(argv[0], "--delete") == 0 && argc == 2) {
                    return delprop(argv[1], persist);
                } else if (strcmp(argv[0], "--help") == 0) {
                    usage(argv0);
                }
            case 'v':
                verbose = true;
                continue;
            case 'p':
                persist = true;
                continue;
            case 'n':
                prop_svc = false;
                continue;
            case '\0':
                break;
            case 'h':
            default:
                usage(argv0);
            }
            break;
        }
        --argc;
        ++argv;
    }

    switch (argc) {
    case 0:
        print_props(persist);
        return 0;
    case 1: {
        string prop = getprop(argv[0], persist);
        if (prop.empty())
            return 1;
        printf("%s\n", prop.data());
        return 0;
    }
    case 2:
        return setprop(argv[0], argv[1], prop_svc);
    default:
        usage(argv0);
    }
}
```
功能不多，除了开始对命令行参数的解析，再就是print_props、getprop、setprop，从字面上很容易了解它们的含义，先从print_props类看

#### 1 impl初始化
print_props也就是批量获取getprop
```c
// native/jni/resetprop/resetprop.cpp
static void print_props(bool persist) {
    getprops([](const char *name, const char *value, auto) {
        printf("[%s]: [%s]\n", name, value);
    }, nullptr, persist);
}

void getprops(void (*callback)(const char *, const char *, void *), void *cookie, bool persist) {
    get_impl()->getprops(callback, cookie, persist);
}
```
首先先初始化impl
```c
// native/jni/resetprop/resetprop.cpp
static sysprop_stub *get_impl() {
    static sysprop_stub *impl = nullptr;
    if (impl == nullptr) {
        // 判断/data/property/persistent_properties是否可读
        use_pb = access(PERSISTENT_PROPERTY_DIR "/persistent_properties", R_OK) == 0;
#ifdef APPLET_STUB_MAIN
        if (__system_properties_init()) {
            LOGE("resetprop: __system_properties_init error\n");
            exit(1);
        }
        impl = new resetprop();
#else
        // Load platform implementations
        // dlsym查找相关system_property方法
        load_functions();
        if (__system_properties_init()) {
            LOGW("resetprop: __system_properties_init error\n");
            impl = new sysprop();
        } else {
            impl = new resetprop();
        }
#endif
    }
    return impl;
}

// native/jni/external/systemproperties/system_property_api.cpp
int __system_properties_init() {
    // 传入/dev/__properties__
    return system_properties.Init(PROP_FILENAME) ? 0 : -1;
}

// native/jni/external/systemproperties/system_properties.cpp
bool SystemProperties::Init(const char* filename) {
  // This is called from __libc_init_common, and should leave errno at 0 (http://b/37248982).
  ErrnoRestorer errno_restorer;

  if (initialized_) {
    /* resetprop remove */
    // contexts_->ResetAccess();
    return true;
  }

  if (strlen(filename) >= PROP_FILENAME_MAX) {
    return false;
  }
  strcpy(property_filename_, filename);

  if (is_dir(property_filename_)) {
    if (access("/dev/__properties__/property_info", R_OK) == 0) {
      contexts_ = new (contexts_data_) ContextsSerialized();
      if (!contexts_->Initialize(false, property_filename_, nullptr)) {
        return false;
      }
    } else {
      contexts_ = new (contexts_data_) ContextsSplit();
      if (!contexts_->Initialize(false, property_filename_, nullptr)) {
        return false;
      }
    }
  } else {
    contexts_ = new (contexts_data_) ContextsPreSplit();
    if (!contexts_->Initialize(false, property_filename_, nullptr)) {
      return false;
    }
  }
  initialized_ = true;
  return true;
}
```

#### 2 getprop

#### 3 setprop