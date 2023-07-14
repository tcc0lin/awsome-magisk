### 一、前言
同样的，作为一个riru模块，从该项目的简介中，就可以发现它的主要作用了
>Riru - MomoHider (aka IsolatedMagiskHider)
### 二、源码分析
从riru模块的入口main.cpp入手
```cpp
static auto module = RiruVersionedModuleInfo {
        .moduleApiVersion = RIRU_NEW_MODULE_API_VERSION,
        .moduleInfo = RiruModuleInfo {
                .supportHide = true,
                .version = RIRU_MODULE_VERSION_CODE,
                .versionName = RIRU_MODULE_VERSION_NAME,
                .onModuleLoaded = onModuleLoaded,
                .shouldSkipUid = nullptr,
                .forkAndSpecializePre = forkAndSpecializePre,
                .forkAndSpecializePost = forkAndSpecializePost,
                .forkSystemServerPre = nullptr,
                .forkSystemServerPost = nullptr,
                .specializeAppProcessPre = specializeAppProcessPre,
                .specializeAppProcessPost = specializeAppProcessPost
        }
};
```
配置了五个自定义方法
#### 1 onModuleLoaded
```c++
bool RegisterHook(const char* name, void* replace, void** backup) {
    int ret = xhook_register(".*\\libandroid_runtime.so$", name, replace, backup);
    if (ret != 0) {
        LOGE("Failed to hook %s", name);
        return true;
    }
    return false;
}

void RegisterHooks() {
    if (!hide_isolated_ && !magic_handle_app_zygote_) return;
    // 使用xhook
    xhook_enable_debug(1);
    xhook_enable_sigsegv_protection(0);
    bool failed = false;
#define HOOK(NAME, REPLACE) \
failed = failed || RegisterHook(#NAME, reinterpret_cast<void*>(REPLACE), reinterpret_cast<void**>(&orig_##NAME))
    // hook fork函数
    HOOK(fork, ForkReplace);
    if (hide_isolated_) {
        // hook unshare函数
        HOOK(unshare, UnshareReplace);
    }

#undef HOOK

    if (failed || xhook_refresh(0)) {
        LOGE("Failed to register hooks!");
        return;
    }
    xhook_clear();
}

void onModuleLoaded() {
    // 获取magisk模块目录
    LOGI("Magisk module dir is %s", module_dir_);
    // 获取magisk tmp目录
    magisk_tmp_ = ReadMagiskTmp();
    LOGI("Magisk temp path is %s", magisk_tmp_);
    // 读取配置文件
    hide_isolated_ = Exists(kIsolated);
    magic_handle_app_zygote_ = Exists(kMagicHandleAppZygote);
    use_nsholder_ = Exists(kSetNs);
    RegisterHooks();
}
```
在加载该模块时，主要做的事情是读取用户自定义的配置文件，并通过xhook hook了fork函数和unshare函数，具体hook操作如下
##### 1.1 ForkReplace
```cpp
pid_t ForkReplace() {
    int read_fd = -1, write_fd = -1;

#if !USE_NEW_APP_ZYGOTE_MAGIC
    // 创建管道，用于后续进程通信
    if (app_zygote_ && magic_handle_app_zygote_) {
        int pipe_fd[2];
        if (pipe(pipe_fd) == -1) {
            LOGE("Failed to create pipe for new app zygote: %s", strerror(errno));
        } else {
            read_fd = pipe_fd[0];
            write_fd = pipe_fd[1];
        }
    }
#endif
    // 调用原始fork方法
    pid_t pid = orig_fork();
    // pid<0，fork失败，清理管道
    if (pid < 0) {
#if !USE_NEW_APP_ZYGOTE_MAGIC
        // fork() failed, clean up
        if (read_fd != -1)
            close(read_fd);
        if (write_fd != -1)
            close(write_fd);
#endif
    } else if (pid == 0) {
        // child process
        // Do not hide here because the namespace not separated
        in_child_ = true;
#if !USE_NEW_APP_ZYGOTE_MAGIC
        if (read_fd != -1 && write_fd != -1) {
            close(read_fd);
            // 获取新pid
            pid_t new_pid = MagicHandleAppZygote();
            // 向管道中写入新pid值
            WriteIntAndClose(write_fd, new_pid);
        }
#endif
    } else {
#if !USE_NEW_APP_ZYGOTE_MAGIC
        // parent process
        if (read_fd != -1 && write_fd != -1) {
            close(write_fd);
            // 读取子进程写入的pid值
            pid = ReadIntAndClose(read_fd);
            LOGI("Zygote received new substitute pid %d", pid);
        }
#endif
    }
    return pid;
}

#if !USE_NEW_APP_ZYGOTE_MAGIC
pid_t MagicHandleAppZygote() {
    LOGI("Magic handling app zygote");
    // App zygote, fork a new process to run it (after forking magiskhide detachs)
    // This makes some detection not working (but also can be easily detected)
    pid_t pid = orig_fork();
    if (pid > 0) {
        // parent
        LOGI("Child zygote forked substitute %d", pid);
        exit(0);
    } else if (pid == 0) {
        pid = getpid();
    } else { // pid < 0
        LOGE("Failed to fork new process for app zygote");
    }
    return pid;
}
#endif
```
##### 1.2 UnshareReplace
#### 2 forkAndSpecializePre
#### 3 forkAndSpecializePost
#### 4 specializeAppProcessPre
#### 5 specializeAppProcessPost