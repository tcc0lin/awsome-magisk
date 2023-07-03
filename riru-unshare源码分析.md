### 一、前言
在之前的Magisk检测方式的文章中，提到过isolated process的概念，MagiskHide无法处理这种进程，因为它和zygote共同使用同一个namespace，如果对这类进程进行unmount的话，会导致所有app都无法正正常访问到su，对于这种情况，可以使用riru-unshare模块来处理使指定的独立进程不与zygote共享namespace
### 二、源码分析
riru-unshare是一个riru模块，看源码就从从main.cpp入手
```c
#ifndef RIRU_MODULE_LEGACY_INIT
RiruVersionedModuleInfo *init(Riru *riru) {
    auto core_max_api_version = riru->riruApiVersion;
    riru_api_version = core_max_api_version <= RIRU_MODULE_API_VERSION ? core_max_api_version : RIRU_MODULE_API_VERSION;
    module.moduleApiVersion = riru_api_version;

    riru_magisk_module_path = strdup(riru->magiskModulePath);
    if (riru_api_version >= 25) {
        riru_allow_unload = riru->allowUnload;
    }
    return &module;
}
#else
RiruVersionedModuleInfo *init(Riru *riru) {
    static int step = 0;
    step += 1;

    switch (step) {
        case 1: {
            auto core_max_api_version = riru->riruApiVersion;
            riru_api_version = core_max_api_version <= RIRU_MODULE_API_VERSION ? core_max_api_version : RIRU_MODULE_API_VERSION;
            if (riru_api_version < 25) {
                module.moduleInfo.unused = (void *) shouldSkipUid;
            } else {
                riru_allow_unload = riru->allowUnload;
            }
            if (riru_api_version >= 24) {
                module.moduleApiVersion = riru_api_version;
                riru_magisk_module_path = strdup(riru->magiskModulePath);
                return &module;
            } else {
                return (RiruVersionedModuleInfo *) &riru_api_version;
            }
        }
        case 2: {
            return (RiruVersionedModuleInfo *) &module.moduleInfo;
        }
        case 3:
        default: {
            return nullptr;
        }
    }
}
#endif
}
```
配置需要替换的函数
```c
static auto module = RiruVersionedModuleInfo{
        .moduleApiVersion = RIRU_MODULE_API_VERSION,
        .moduleInfo= RiruModuleInfo{
                .supportHide = true,
                .version = RIRU_MODULE_VERSION,
                .versionName = RIRU_MODULE_VERSION_NAME,
                .onModuleLoaded = nullptr,
                .forkAndSpecializePre = forkAndSpecializePre,
                .forkAndSpecializePost = forkAndSpecializePost,
                .forkSystemServerPre = nullptr,
                .forkSystemServerPost = nullptr,
                .specializeAppProcessPre = specializeAppProcessPre,
                .specializeAppProcessPost = specializeAppProcessPost
        }
};
```
主要是四个函数forkAndSpecializePre、forkAndSpecializePost、specializeAppProcessPre和specializeAppProcessPost
#### 1 forkAndSpecializePre
在forkAndSpecializePre阶段做doUnshare处理
```c
static void forkAndSpecializePre(
        JNIEnv *env, jclass clazz, jint *uid, jint *gid, jintArray *gids, jint *runtimeFlags,
        jobjectArray *rlimits, jint *mountExternal, jstring *seInfo, jstring *niceName,
        jintArray *fdsToClose, jintArray *fdsToIgnore, jboolean *is_child_zygote,
        jstring *instructionSet, jstring *appDataDir, jboolean *isTopApp, jobjectArray *pkgDataInfoList,
        jobjectArray *whitelistedDataInfoList, jboolean *bindMountAppDataDirs, jboolean *bindMountAppStorageDirs) {
    doUnshare(env, uid, mountExternal, niceName, *is_child_zygote);
}

static void doUnshare(JNIEnv *env, jint *uid, jint *mountExternal, jstring *niceName, bool is_child_zygote) {
    if (shouldSkipUid(*uid)) return;
    if (*mountExternal == 0) {
        *mountExternal = 1;
        ScopedUtfChars name(env, *niceName);
        is_app_zygote = is_child_zygote && is_app(*uid);
        nice_name_ = niceName;
        LOGI("unshare uid=%d name=%s app_zygote=%s", *uid, name.c_str(), is_app_zygote?"true":"false");
    }
}

static int shouldSkipUid(int uid) {
    int appid = uid % AID_USER_OFFSET;
    if (appid >= AID_APP_START && appid <= AID_APP_END) return false;
    if (appid >= AID_ISOLATED_START && appid <= AID_ISOLATED_END) return false;
    return true;
}
```
#### 2 forkAndSpecializePost
```c
static void forkAndSpecializePost(JNIEnv *env, jclass clazz, jint res) {
    if (res == 0) {
        if (is_app_zygote && nice_name_)
            SetProcessName(env, *nice_name_);
        clear_state();
        riru_set_unload_allowed(true);
    }
}

void SetProcessName(JNIEnv* env, jstring name) {
    jclass Process = env->FindClass("android/os/Process");
    jmethodID setArgV0 = env->GetStaticMethodID(Process, "setArgV0", "(Ljava/lang/String;)V");
    if (env->ExceptionCheck()) {
        env->ExceptionClear();
        LOGW("Process.setArgV0(String) not found");
    } else {
        env->CallStaticVoidMethod(Process, setArgV0, name);
        if (env->ExceptionCheck()) {
            env->ExceptionClear();
            LOGW("Process.setArgV0(String) threw exception");
        }
    }
    env->DeleteLocalRef(Process);
}

static void clear_state(){
    nice_name_ = nullptr;
    is_app_zygote = false;
}
```
#### 3 specializeAppProcessPre
```c
static void specializeAppProcessPre(
        JNIEnv *env, jclass clazz, jint *uid, jint *gid, jintArray *gids, jint *runtimeFlags,
        jobjectArray *rlimits, jint *mountExternal, jstring *seInfo, jstring *niceName,
        jboolean *startChildZygote, jstring *instructionSet, jstring *appDataDir,
        jboolean *isTopApp, jobjectArray *pkgDataInfoList, jobjectArray *whitelistedDataInfoList,
        jboolean *bindMountAppDataDirs, jboolean *bindMountAppStorageDirs) {
    doUnshare(env, uid, mountExternal, niceName, false);
}
```
#### 4 specializeAppProcessPost
```c
static void specializeAppProcessPost(JNIEnv *env, jclass clazz) {
    clear_state();
    riru_set_unload_allowed(true);
}
```