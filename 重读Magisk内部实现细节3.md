### 前言
Magisk内部实现细节的第三篇，主要通过源码来了解下Magisk Hide的原理，这部分代码在native/jni/magiskhide当中

### 一、Magisk Hide入口
不管是在Magisk Manager中管理Magisk Hide
```kotlin
object MagiskHide : BaseSettingsItem.Toggle() {
    override val title = R.string.magiskhide.asText()
    override val description = R.string.settings_magiskhide_summary.asText()
    override var value = Config.magiskHide
        set(value) = setV(value, field, { field = it }) {
            val cmd = if (it) "enable" else "disable"
            Shell.su("magiskhide $cmd").submit { cb ->
                if (cb.isSuccess) Config.magiskHide = it
                else field = !it
            }
        }
}
```
还是通过adb shell来管理Magisk Hide
```shell
(base)  大慈大悲观世音菩萨  ~  adb shell
selene:/ $ su
selene:/ # magiskhide status
MagiskHide is not enabled
1|selene:/ # magiskhide enable
selene:/ # magiskhide status
MagiskHide is enabled
selene:/ #
```
其底层都是通过magiskhide这个二进制文件来触发的，而magiskhide的入口是
```c
// native/jni/magiskhide/magiskhide.cpp
// 入口函数
int magiskhide_main(int argc, char *argv[]) {
    if (argc < 2)
        usage(argv[0]);

    // CLI backwards compatibility
    const char *opt = argv[1];
    if (opt[0] == '-' && opt[1] == '-')
        opt += 2;

    int req;
    // 选择触发的指令
    if (opt == "enable"sv)
        req = LAUNCH_MAGISKHIDE;
    else if (opt == "disable"sv)
        req = STOP_MAGISKHIDE;
    else if (opt == "add"sv)
        req = ADD_HIDELIST;
    else if (opt == "rm"sv)
        req = RM_HIDELIST;
    else if (opt == "ls"sv)
        req = LS_HIDELIST;
    else if (opt == "status"sv)
        req = HIDE_STATUS;
    else if (opt == "exec"sv && argc > 2) {
        xunshare(CLONE_NEWNS);
        xmount(nullptr, "/", nullptr, MS_PRIVATE | MS_REC, nullptr);
        hide_unmount();
        execvp(argv[2], argv + 2);
        exit(1);
    }
#if 0 && !ENABLE_INJECT
    else if (opt == "test"sv)
        test_proc_monitor();
#endif
    else
        usage(argv[0]);
    // 同样需要和daemon进行交互
    // Send request
    int fd = connect_daemon();
    write_int(fd, MAGISKHIDE);
    write_int(fd, req);
    if (req == ADD_HIDELIST || req == RM_HIDELIST) {
        write_string(fd, argv[2]);
        write_string(fd, argv[3] ? argv[3] : "");
    }

    // Get response
    int code = read_int(fd);
    switch (code) {
    case DAEMON_SUCCESS:
        break;
    case HIDE_NOT_ENABLED:
        fprintf(stderr, "MagiskHide is not enabled\n");
        goto return_code;
    case HIDE_IS_ENABLED:
        fprintf(stderr, "MagiskHide is enabled\n");
        goto return_code;
    case HIDE_ITEM_EXIST:
        fprintf(stderr, "Target already exists in hide list\n");
        goto return_code;
    case HIDE_ITEM_NOT_EXIST:
        fprintf(stderr, "Target does not exist in hide list\n");
        goto return_code;
    case HIDE_NO_NS:
        fprintf(stderr, "Your kernel doesn't support mount namespace\n");
        goto return_code;
    case HIDE_INVALID_PKG:
        fprintf(stderr, "Invalid package / process name\n");
        goto return_code;
    case ROOT_REQUIRED:
        fprintf(stderr, "Root is required for this operation\n");
        goto return_code;
    case DAEMON_ERROR:
    default:
        fprintf(stderr, "Daemon error\n");
        return DAEMON_ERROR;
    }

    if (req == LS_HIDELIST) {
        string res;
        for (;;) {
            read_string(fd, res);
            if (res.empty())
                break;
            printf("%s\n", res.data());
        }
    }

return_code:
    return req == HIDE_STATUS ? (code == HIDE_IS_ENABLED ? 0 : 1) : code != DAEMON_SUCCESS;
```
而对于daemon进程来说，处理magiskhide传来的指令，具体的处理逻辑还是在magiskhide.cpp中
```c
// native/jni/core/daemon.cpp
static void request_handler(int client, int req_code, ucred cred) {
    switch (req_code) {
        case MAGISKHIDE:
            magiskhide_handler(client, &cred);
            break;
        ......

// native/jni/magiskhide/magiskhide.cpp
void magiskhide_handler(int client, ucred *cred) {
    int req = read_int(client);
    int res = DAEMON_ERROR;

    ......

    switch (req) {
    // magiskhide启动
    case LAUNCH_MAGISKHIDE:
        res = launch_magiskhide(true);
        break;
    // magiskhide关闭
    case STOP_MAGISKHIDE:
        res = stop_magiskhide();
        break;
    // 新增需要隐藏的app
    case ADD_HIDELIST:
        res = add_list(client);
        break;
    // 移除
    case RM_HIDELIST:
        res = rm_list(client);
        break;
    case LS_HIDELIST:
        ls_list(client);
        return;
    case HIDE_STATUS:
        res = hide_enabled() ? HIDE_IS_ENABLED : HIDE_NOT_ENABLED;
        break;
#if ENABLE_INJECT
    case REMOTE_CHECK_HIDE:
        res = check_uid_map(client);
        break;
    case REMOTE_DO_HIDE:
        kill(cred->pid, SIGSTOP);
        write_int(client, 0);
        hide_daemon(cred->pid);
        close(client);
        return;
#endif
    }

    write_int(client, res);
    close(client);
}
```

### 二、Magisk Hide指令分析
#### 1 LAUNCH_MAGISKHIDE
```c
// native/jni/magiskhide/hide_utils.cpp
// 开启magiskhide
int launch_magiskhide(bool late_props) {
    // 锁申请
    mutex_guard lock(hide_state_lock);

    // 判断全局变量hide_state的值，如果已经启动直接返回
    if (hide_state)
        return HIDE_IS_ENABLED;

    // 检测是否有访问namespace的权限
    if (access("/proc/self/ns/mnt", F_OK) != 0)
        return HIDE_NO_NS;

    // 复制procfp
    if (procfp == nullptr && (procfp = opendir("/proc")) == nullptr)
        return DAEMON_ERROR;

    LOGI("* Enable MagiskHide\n");
    // 初始化hide_set并杀死相关进程
    // Initialize the hide list
    if (!init_list())
        return DAEMON_ERROR;
    // 替换prop属性
    hide_sensitive_props();
    if (late_props)
        // 针对vendor.boot.verifiedbootstate进行替换
        hide_late_sensitive_props();

#if !ENABLE_INJECT
    // Start monitoring
    // 创建监控线程monitor_thread
    if (new_daemon_thread(&proc_monitor))
        return DAEMON_ERROR;
#endif
    // 更新当前magiskhide状态
    hide_state = true;
    // 更新settings里的magiskhide配置
    update_hide_config();
    // 释放锁
    // Unlock here or else we'll be stuck in deadlock
    lock.unlock();
    // 更新uid_proc_map，需要隐藏的app的uid对应进程名
    update_uid_map();
    return DAEMON_SUCCESS;
}
```
#### 2 STOP_MAGISKHIDE
```c
// native/jni/magiskhide/hide_utils.cpp
int stop_magiskhide() {
    mutex_guard g(hide_state_lock);

    if (hide_state) {
        LOGI("* Disable MagiskHide\n");
        // 清理工作
        uid_proc_map.clear();
        hide_set.clear();
#if !ENABLE_INJECT
        // 向monitor_thread发送自定义信号SIGTERMTHRD
        pthread_kill(monitor_thread, SIGTERMTHRD);
#endif
    }
    // 更新当前magiskhide状态
    hide_state = false;
    // 更新settings里的magiskhide配置
    update_hide_config();
    return DAEMON_SUCCESS;
}
```
#### 3 ADD_HIDELIST
```c
// native/jni/magiskhide/hide_utils.cpp
int add_list(int client) {
    string pkg = read_string(client);
    string proc = read_string(client);
    int ret = add_list(pkg.data(), proc.data());
    if (ret == DAEMON_SUCCESS)
        // 更新uid_proc_map
        update_uid_map();
    return ret;
}

static int add_list(const char *pkg, const char *proc) {
    if (proc[0] == '\0')
        proc = pkg;

    if (!validate(pkg, proc))
        return HIDE_INVALID_PKG;

    for (auto &hide : hide_set)
        if (hide.first == pkg && hide.second == proc)
            return HIDE_ITEM_EXIST;

    // Add to database
    char sql[4096];
    // 写入hidelist数据表
    snprintf(sql, sizeof(sql),
            "INSERT INTO hidelist (package_name, process) VALUES('%s', '%s')", pkg, proc);
    char *err = db_exec(sql);
    db_err_cmd(err, return DAEMON_ERROR);

    {
        // Critical region
        mutex_guard lock(hide_state_lock);
        // 更新hide_set
        add_hide_set(pkg, proc);
    }

    return DAEMON_SUCCESS;
}
```
可以看出，首先在Magisk Hide中有三个存储结构用来做Magisk Hide的管理工作
- hide_set: 存储需要隐藏功能的包名-进程名
- uid_proc_map: 根据hide_set集合来存储对应App的uid以及进程名映射
- hidelist: 数据表，供展示时使用

其次，可以看到在Magisk Hide启动时会额外启动monitor_thread这个线程，而这个就是Magisk Hide隐藏功能的核心

### 三、Magisk Hide原理
跟进monitor_thread
#### 1 信号处理
```c
// native/jni/magiskhide/proc_monitor.cpp

// 设置该线程为monitor_thread，并于后续清理
monitor_thread = pthread_self();

// Backup original mask
// 获取当前线程的信号掩码保存在orin_mask
sigset_t orig_mask;
pthread_sigmask(SIG_SETMASK, nullptr, &orig_mask);

// 清空信号集并初始化
sigset_t unblock_set;
sigemptyset(&unblock_set);
sigaddset(&unblock_set, SIGTERMTHRD);
sigaddset(&unblock_set, SIGIO);
sigaddset(&unblock_set, SIGALRM);

// 设置信号处理函数集合
struct sigaction act{};
sigfillset(&act.sa_mask);
act.sa_handler = SIG_IGN;
sigaction(SIGTERMTHRD, &act, nullptr);
sigaction(SIGIO, &act, nullptr);
sigaction(SIGALRM, &act, nullptr);

// 防止信号积压处理
// Temporary unblock to clear pending signals
pthread_sigmask(SIG_UNBLOCK, &unblock_set, nullptr);
pthread_sigmask(SIG_SETMASK, &orig_mask, nullptr);

// 使用term_thread来处理SIGTERMTHRD信号
act.sa_handler = term_thread;
sigaction(SIGTERMTHRD, &act, nullptr);
// 使用inotify_event处理SIGIO信号
act.sa_handler = inotify_event;
sigaction(SIGIO, &act, nullptr);
// 使用check_zygote处理SIGALRM信号
act.sa_handler = [](int){ check_zygote(); };
sigaction(SIGALRM, &act, nullptr);

setup_inotify();

static void setup_inotify() {
    // 创建inotify实例时指定了IN_CLOEXEC标志位，表示将inotify实例设置为 close-on-exec 模式。
    // 在close-on-exec模式下，当进程调用exec函数时，inotify实例会自动关闭
    inotify_fd = xinotify_init1(IN_CLOEXEC);
    if (inotify_fd < 0)
        return;

    // Setup inotify asynchronous I/O
    // 设置inotify文件描述符的异步通知和所有权
    fcntl(inotify_fd, F_SETFL, O_ASYNC);
    struct f_owner_ex ex = {
        .type = F_OWNER_TID,
        .pid = gettid()
    };
    fcntl(inotify_fd, F_SETOWN_EX, &ex);

    // 监控/data/system的写入并关闭事件
    // Monitor packages.xml
    inotify_add_watch(inotify_fd, "/data/system", IN_CLOSE_WRITE);

    // 监控app_process的被访问的事件，也就是监控App
    // Monitor app_process
    if (access(APP_PROC "32", F_OK) == 0) {
        inotify_add_watch(inotify_fd, APP_PROC "32", IN_ACCESS);
        if (access(APP_PROC "64", F_OK) == 0)
            inotify_add_watch(inotify_fd, APP_PROC "64", IN_ACCESS);
    } else {
        inotify_add_watch(inotify_fd, APP_PROC, IN_ACCESS);
    }
}
```
这个部分主要做的事是
- 设置信号处理函数，信号分别是SIGTERMTHRD、SIGIO、SIGALRM
- 启动inotify，fd写入inotify_fd，监控/system/bin/app_process的access事件，重点在于packages.xml文件的写入
#### 2 ptrace Zygote
```c
check_zygote();
if (!is_zygote_done()) {
    // 如果获取到zygote，则每250ms发送SIGALRM信号触发check_zygote
    // Periodic scan every 250ms
    timeval val { .tv_sec = 0, .tv_usec = 250000 };
    itimerval interval { .it_interval = val, .it_value = val };
    setitimer(ITIMER_REAL, &interval, nullptr);
}

static void check_zygote() {
    crawl_procfs([](int pid) -> bool {
        char buf[512];
        snprintf(buf, sizeof(buf), "/proc/%d/cmdline", pid);
        if (FILE *f = fopen(buf, "re")) {
            fgets(buf, sizeof(buf), f);
            if (strncmp(buf, "zygote", 6) == 0 && parse_ppid(pid) == 1)
                new_zygote(pid);
            fclose(f);
        }
        return true;
    });
    if (is_zygote_done()) {
        // Stop periodic scanning
        timeval val { .tv_sec = 0, .tv_usec = 0 };
        itimerval interval { .it_interval = val, .it_value = val };
        setitimer(ITIMER_REAL, &interval, nullptr);
    }
}

static DIR *procfp;
// procfp在之前已经被赋值成/proc目录
void crawl_procfs(const function<bool(int)> &fn) {
    // 指针重置到目录起始位置
    rewinddir(procfp);
    crawl_procfs(procfp, fn);
}

// 遍历proc目录，获取zygote的pid
void crawl_procfs(DIR *dir, const function<bool(int)> &fn) {
    struct dirent *dp;
    int pid;
    while ((dp = readdir(dir))) {
        pid = parse_int(dp->d_name);
        if (pid > 0 && !fn(pid))
            break;
    }
}

static void new_zygote(int pid) {
    struct stat st;
    // 读取zygote挂载的namespace信息
    if (read_ns(pid, &st))
        return;

    // 更新或者存储st到zygote_map
    auto it = zygote_map.find(pid);
    if (it != zygote_map.end()) {
        // Update namespace info
        it->second = st;
        return;
    }

    LOGD("proc_monitor: ptrace zygote PID=[%d]\n", pid);
    zygote_map[pid] = st;
    // ptrace attach到zygote进程
    xptrace(PTRACE_ATTACH, pid);
    // 等待zygote进程状态变化 
    waitpid(pid, nullptr, __WALL | __WNOTHREAD);
    监控zygote fork/vfork/exit事件
    xptrace(PTRACE_SETOPTIONS, pid, nullptr,
            PTRACE_O_TRACEFORK | PTRACE_O_TRACEVFORK | PTRACE_O_TRACEEXIT);
    // 恢复zygote进程执行
    xptrace(PTRACE_CONT, pid);
}
```
这一部分的作用是轮询判断zygote进程是否启动以及ptrace attach到zygote以便于监控到zygote的fork操作（引导启动App进程）
#### 3 子进程信号处理
```c
for (int status;;) {
    // 解除信号阻塞，获取信号
    pthread_sigmask(SIG_UNBLOCK, &unblock_set, nullptr);
    // 获取待处理的pid
    const int pid = waitpid(-1, &status, __WALL | __WNOTHREAD);
    if (pid < 0) {
        if (errno == ECHILD) {
            // Nothing to wait yet, sleep and wait till signal interruption
            LOGD("proc_monitor: nothing to monitor, wait for signal\n");
            struct timespec ts = {
                .tv_sec = INT_MAX,
                .tv_nsec = 0
            };
            nanosleep(&ts, nullptr);
        }
        continue;
    }

    pthread_sigmask(SIG_SETMASK, &orig_mask, nullptr);

    if (!WIFSTOPPED(status) /* Ignore if not ptrace-stop */)
        DETACH_AND_CONT;
    // 获取pid的信号和事件类型
    int event = WEVENT(status);
    int signal = WSTOPSIG(status);

    if (signal == SIGTRAP && event) {
        unsigned long msg;
        xptrace(PTRACE_GETEVENTMSG, pid, nullptr, &msg);
        // 处理zygote消息
        if (zygote_map.count(pid)) {
            // Zygote event
            switch (event) {
                case PTRACE_EVENT_FORK:
                case PTRACE_EVENT_VFORK:
                    PTRACE_LOG("zygote forked: [%lu]\n", msg);
                    // 表示收到的是zygote消息，监控到zygote fork子进程
                    // 此时设置attaches map中app pid的值为true
                    attaches[msg] = true;
                    break;
                case PTRACE_EVENT_EXIT:
                    PTRACE_LOG("zygote exited with status: [%lu]\n", msg);
                    [[fallthrough]];
                default:
                    zygote_map.erase(pid);
                    DETACH_AND_CONT;
            }
        } else {
            // 处理用户App消息
            switch (event) {
                // 表示收到的是子进程的信号，有新的App启动，开始执行隐藏操作
                case PTRACE_EVENT_CLONE:
                    PTRACE_LOG("create new threads: [%lu]\n", msg);
                    if (attaches[pid] && check_pid(pid))
                        continue;
                    break;
                case PTRACE_EVENT_EXEC:
                case PTRACE_EVENT_EXIT:
                    PTRACE_LOG("exit or execve\n");
                    [[fallthrough]];
                default:
                    DETACH_AND_CONT;
            }
        }
        xptrace(PTRACE_CONT, pid);
    } else if (signal == SIGSTOP) {
        // 收到暂停信号，继续监控
        if (!attaches[pid]) {
            // Double check if this is actually a process
            attaches[pid] = is_process(pid);
        }
        if (attaches[pid]) {
            // This is a process, continue monitoring
            PTRACE_LOG("SIGSTOP from child\n");
            xptrace(PTRACE_SETOPTIONS, pid, nullptr,
                    PTRACE_O_TRACECLONE | PTRACE_O_TRACEEXEC | PTRACE_O_TRACEEXIT);
            xptrace(PTRACE_CONT, pid);
        } else {
            // This is a thread, do NOT monitor
            PTRACE_LOG("SIGSTOP from thread\n");
            DETACH_AND_CONT;
        }
    } else {
        // 恢复执行
        // Not caused by us, resend signal
        xptrace(PTRACE_CONT, pid, nullptr, signal);
        PTRACE_LOG("signal [%d]\n", signal);
    }
}

static bool check_pid(int pid) {
    char path[128];
    char cmdline[1024];
    struct stat st;

    sprintf(path, "/proc/%d", pid);
    if (stat(path, &st)) {
        // Process died unexpectedly, ignore
        detach_pid(pid);
        return true;
    }
    // 获取进程pid
    int uid = st.st_uid;

    // UID hasn't changed
    if (uid == 0)
        return false;

    // 读取/proc/pid/cmdline的内容到cmdline
    sprintf(path, "/proc/%d/cmdline", pid);
    if (auto f = open_file(path, "re")) {
        fgets(cmdline, sizeof(cmdline), f.get());
    } else {
        // Process died unexpectedly, ignore
        detach_pid(pid);
        return true;
    }
    
    // 必须是用户进程
    if (cmdline == "zygote"sv || cmdline == "zygote32"sv || cmdline == "zygote64"sv ||
        cmdline == "usap32"sv || cmdline == "usap64"sv)
        return false;

    // 如果非需要隐藏的进程，忽略
    if (!is_hide_target(uid, cmdline, 95))
        goto not_target;

    // 同上
    // Ensure ns is separated
    read_ns(pid, &st);
    for (auto &zit : zygote_map) {
        if (zit.second.st_ino == st.st_ino &&
            zit.second.st_dev == st.st_dev) {
            // ns not separated, abort
            LOGW("proc_monitor: skip [%s] PID=[%d] UID=[%d]\n", cmdline, pid, uid);
            goto not_target;
        }
    }

    // Detach but the process should still remain stopped
    // The hide daemon will resume the process after hiding it
    LOGI("proc_monitor: [%s] PID=[%d] UID=[%d]\n", cmdline, pid, uid);
    detach_pid(pid, SIGSTOP);
    hide_daemon(pid);
    return true;

not_target:
    PTRACE_LOG("[%s] is not our target\n", cmdline);
    detach_pid(pid);
    return true;
}

void hide_daemon(int pid) {
    if (fork_dont_care() == 0) {
        // 关键隐藏动作
        hide_unmount(pid);
        // Send resume signal
        kill(pid, SIGCONT);
        _exit(0);
    }
}

void hide_unmount(int pid) {
    // 切换目标pid的namespace
    if (pid > 0 && switch_mnt_ns(pid))
        return;

    LOGD("hide: handling PID=[%d]\n", pid);

    char val;
    // 读取selinux的模式
    int fd = xopen(SELINUX_ENFORCE, O_RDONLY);
    xxread(fd, &val, sizeof(val));
    close(fd);
    // Permissive
    // 如果是宽容模式，则限制访问
    if (val == '0') {
        chmod(SELINUX_ENFORCE, 0640);
        chmod(SELINUX_POLICY, 0440);
    }

    vector<string> targets;

    // Unmount dummy skeletons and /sbin links
    // android11中为/dev/xxxx
    targets.push_back(MAGISKTMP);
    parse_mnt("/proc/self/mounts", [&](mntent *mentry) {
        if (TMPFS_MNT(system) || TMPFS_MNT(vendor) || TMPFS_MNT(product) || TMPFS_MNT(system_ext))
            targets.emplace_back(mentry->mnt_dir);
        return true;
    });

    for (auto &s : reversed(targets))
        lazy_unmount(s.data());
    targets.clear();

    // Unmount all Magisk created mounts
    parse_mnt("/proc/self/mounts", [&](mntent *mentry) {
        if (strstr(mentry->mnt_fsname, BLOCKDIR))
            targets.emplace_back(mentry->mnt_dir);
        return true;
    });

    for (auto &s : reversed(targets))
        lazy_unmount(s.data());
}

```
核心步骤，上一步已经ptrace attach到zygote，当监听到App启动了，切换到目标进程的namespace并执行umount操作