### 一、市面现存的检测方式
#### 1 Magisk Detector
来源于[Magisk Detector](https://github.com/vvb2060/MagiskDetector)（现已停止维护），我们可以从官方的[细节文档](https://github.com/vvb2060/MagiskDetector/blob/master/README_ZH.md)看出它之前的设计思路
，目前从最新的代码上看，仅仅存在三种检测方式
```c
JNINativeMethod methods[] = {
        {"haveSu",         "()I", haveSu},
        {"haveMagiskHide", "()I", haveMagiskHide},
        {"haveMagicMount", "()I", haveMagicMount},
};
```
##### 1.1 su文件检测
- 检测方式

    ```c++
    static int scan_path() {
        char *path = getenv("PATH");
        char *p = strtok(path, ":");
        char supath[PATH_MAX];
        do {
            sprintf(supath, "%s/su", p);
            if (access(supath, F_OK) == 0) {
                LOGW("Found su at %s", supath);
                return 1;
            }
        } while ((p = strtok(NULL, ":")) != NULL);
        return 0;
    }
    ```
    代码比较少，很容易理解，通过获取系统环境变量path的值来确定当前有哪些可执行文件的目录，再依次遍历这些目录检测是否存在su文件，系统环境变量path内的路径通常是
    ```shell
    selene:/ $ echo $PATH
    /product/bin:/apex/com.android.runtime/bin:/apex/com.android.art/bin:/system_ext/bin:/system/bin:/system/xbin:/odm/bin:/vendor/bin:/vendor/xbin
    selene:/ $
    ```
- 思路

    之所以要检测这些可执行文件的目录是否存在su文件是因为正常情况下，root过的手机都会依照传统方式通过执行su命令来提权切换到root用户下，那这就依靠在这些可执行文件的目录下放置su文件。对于Magisk来说，同样也是在每次启动后去动态修改bin目录，先是将magisk、magiskinit放入自身文件下的bin目录，再将例如su、magiskhide等做magisk的软链，最后通过bind mount同步到真实的bin目录下达到修改bin的效果
    ```c++
    // native/jni/core/module.cpp
    static void inject_magisk_bins(root_node *system) {
        auto bin = system->child<inter_node>("bin");
        if (!bin) {
            bin = new inter_node("bin", "");
            system->insert(bin);
        }

        // Insert binaries
        bin->insert(new magisk_node("magisk"));
        bin->insert(new magisk_node("magiskinit"));

        // Also delete all applets to make sure no modules can override it
        for (int i = 0; applet_names[i]; ++i)
            delete bin->extract(applet_names[i]);
        for (int i = 0; init_applet[i]; ++i)
            delete bin->extract(init_applet[i]);
    }

    class magisk_node : public node_entry {
    public:
        explicit magisk_node(const char *name) : node_entry(name, DT_REG, this) {}

        void mount() override {
            const string &dir_name = parent()->node_path();
            if (name() == "magisk") {
                for (int i = 0; applet_names[i]; ++i) {
                    string dest = dir_name + "/" + applet_names[i];
                    VLOGD("create", "./magisk", dest.data());
                    xsymlink("./magisk", dest.data());
                }
            } else {
                for (int i = 0; init_applet[i]; ++i) {
                    string dest = dir_name + "/" + init_applet[i];
                    VLOGD("create", "./magiskinit", dest.data());
                    xsymlink("./magiskinit", dest.data());
                }
            }
            create_and_mount(MAGISKTMP + "/" + name());
        }
    };
    ```
    所以，可以在/system/bin目录下看到magisk所做的变动，通过这个方面来检测magisk
    ```shell
    selene:/ $ ls -al /system/bin |grep magisk
    -rwxr-xr-x  1 root root   170224 2023-06-16 17:00 magisk
    lrwxrwxrwx  1 root root        8 2023-06-16 17:00 magiskhide -> ./magisk
    -rwxr-xr-x  1 root root  3987848 2023-06-16 17:00 magiskinit
    lrwxrwxrwx  1 root root       12 2023-06-16 17:00 magiskpolicy -> ./magiskinit
    lrwxrwxrwx  1 root root        8 2023-06-16 17:00 resetprop -> ./magisk
    lrwxrwxrwx  1 root root        8 2023-06-16 17:00 su -> ./magisk
    lrwxrwxrwx  1 root root       12 2023-06-16 17:00 supolicy -> ./magiskinit
    ```
##### 1.2 Magisk模块篡改系统文件检测
- 检测方式

    ```c++
    static jint haveMagicMount(JNIEnv *env __unused, jclass clazz __unused) {
        dev_t data_dev = scan_mountinfo();
        if (data_dev == 0) return -1;
        return scan_maps(data_dev);
    }

    static dev_t scan_mountinfo() {
        int major = 0;
        int minor = 0;
        char line[PATH_MAX];
        char mountinfo[] = "/proc/self/mountinfo";
        int fd = sys_open(mountinfo, O_RDONLY, 0);
        if (fd < 0) {
            LOGE("cannot open %s", mountinfo);
            return 0;
        }
        FILE *fp = fdopen(fd, "r");
        if (fp == NULL) {
            LOGE("cannot open %s", mountinfo);
            close(fd);
            return 0;
        }
        // 遍历mountinfo文件，判断存在/ /data的行时拿它的设备号
        while (fgets(line, PATH_MAX - 1, fp) != NULL) {
            if (strstr(line, "/ /data ") != NULL) {
                sscanf(line, "%*d %*d %d:%d", &major, &minor);
            }
        }
        fclose(fp);
        // 根据major和minor创建设备号
        return makedev(major, minor);
    }

    static int scan_maps(dev_t data_dev) {
        int module = 0;
        char line[PATH_MAX];
        char maps[] = "/proc/self/maps";
        int fd = sys_open(maps, O_RDONLY, 0);
        if (fd < 0) {
            LOGE("cannot open %s", maps);
            return -1;
        }
        FILE *fp = fdopen(fd, "r");
        if (fp == NULL) {
            LOGE("cannot open %s", maps);
            close(fd);
            return -1;
        }
        while (fgets(line, PATH_MAX - 1, fp) != NULL) {
            // 在maps的内容里判断都否存在/data目录下的设备号
            if (strchr(line, '/') == NULL) continue;
            if (strstr(line, " /system/") != NULL ||
                strstr(line, " /vendor/") != NULL ||
                strstr(line, " /product/") != NULL ||
                strstr(line, " /system_ext/") != NULL) {
                int f;
                int s;
                char p[PATH_MAX];
                sscanf(line, "%*s %*s %*s %x:%x %*s %s", &f, &s, p);
                if (makedev(f, s) == data_dev) {
                    LOGW("Magisk module file %x:%x %s", f, s, p);
                    module++;
                }
            }
        }
        fclose(fp);
        return module;
    }
    ```
- 思路

    我理解是在Android系统中，类似system、vendor、product这些都属于系统相关的镜像，它们挂载到设备上时分区通常是只读的，而相对而言，data分区是可读写的，因此某些magisk模块会通过挂载的方式将例如system挂载到/data/system下面，从而完成对system分区的修改。
    >
    这样做的检测原理是例如当访问/system/build.prop时，实际上却是访问/data/system/build.prop，在上面的检测逻辑中是先获取挂载信息中/data目录相关的挂载设备号，例如
    ```shell
    selene:/ # cat /proc/7428/mountinfo |grep "/ /data"
    94628 94522 253:6 / /data rw,nosuid,nodev,noatime master:42 - f2fs /dev/block/dm-6 rw,lazytime,seclabel,background_gc=on,gc_merge,discard,no_heap,user_xattr,inline_xattr,acl,inline_data,inline_dentry,extent_cache,mode=adaptive,active_logs=6,reserve_root=54072,resuid=0,resgid=1065,inlinecrypt,alloc_mode=default,checkpoint_merge,fsync_mode=nobarrier
    136334 94522 0:36 / /data_mirror rw,nosuid,nodev,noexec,relatime master:43 - tmpfs tmpfs rw,seclabel,size=1906244k,nr_inodes=476561,mode=700,gid=1000
    141106 94628 0:155 / /data/data rw,nosuid,nodev,noexec,relatime - tmpfs tmpfs rw,seclabel,mode=751
    141107 94628 0:156 / /data/user rw,nosuid,nodev,noexec,relatime - tmpfs tmpfs rw,seclabel,mode=751
    141108 94628 0:157 / /data/user_de rw,nosuid,nodev,noexec,relatime - tmpfs tmpfs rw,seclabel,mode=751
    141111 94628 0:158 / /data/misc/profiles/cur rw,nosuid,nodev,noexec,relatime - tmpfs tmpfs rw,seclabel,mode=751 
    ```
    像253:6、0:36这样的主设备号:次设备号的结构，接着去maps中寻找是否system、vendor、product中是否包含相同设备号的，例如
    ```shell
    7cd43b5000-7cd43b6000 r--s 00000000 fd:06 924                            /data/resource-cache/vendor@overlay@MiuiFrameworkResOverlay.apk@idmap
    7cd4424000-7cd4425000 r--s 00000000 fd:02 2543                           /vendor/overlay/DevicesAndroidOverlay.apk
    7cd4425000-7cd4426000 r--s 00002000 fd:02 2543                           /vendor/overlay/DevicesAndroidOverlay.apk
    7cd4426000-7cd4427000 r--s 00000000 fd:06 920                            /data/resource-cache/vendor@overlay@DevicesAndroidOverlay.apk@idmap
    7cd445a000-7cd445c000 r--s 00000000 fd:02 2539                           /vendor/overlay/AospFrameworkResOverlay.apk
    7cd445c000-7cd445d000 r--s 00000000 fd:06 919                            /data/resource-cache/vendor@overlay@AospFrameworkResOverlay.apk@idmap
    7cd4497000-7cd4499000 r--s 00000000 fd:02 2541                           /vendor/overlay/DeviceAndroidConfig.apk
    7cd4499000-7cd449a000 r--s 00000000 fd:06 918                            /data/resource-cache/vendor@overlay@DeviceAndroidConfig.apk@idmap
    7cd44fa000-7cd44fb000 r--s 00000000 fd:02 2546                           /vendor/overlay/FrameworkResOverlay/FrameworkResOverlay.apk
    7cd44fb000-7cd44fc000 r--s 00002000 fd:02 2546                           /vendor/overlay/FrameworkResOverlay/FrameworkResOverlay.apk
    7cd44fc000-7cd44fd000 r--s 00000000 fd:06 917                            /data/resource-cache/vendor@overlay@FrameworkResOverlay@FrameworkResOverlay.apk@idmap
    7cd47d5000-7cd47e0000 r--p 00000000 fd:01 3517                           /system/lib64/vendor.mediatek.hardware.mms@1.2.so
    7cd47e0000-7cd47e8000 r-xp 0000b000 fd:01 3517                           /system/lib64/vendor.mediatek.hardware.mms@1.2.so
    7cd47e8000-7cd47ea000 r--p 00013000 fd:01 3517                           /system/lib64/vendor.mediatek.hardware.mms@1.2.so
    7cd47ea000-7cd47eb000 rw-p 00014000 fd:01 3517                           /system/lib64/vendor.mediatek.hardware.mms@1.2.so
    ```
    取出像01:3517、02:2541设备号来检测一致性
    >
    上述检测异常结果并没有在Android11上复现，待后续分析原因
##### 1.3 Magisk Hide开启检测
- 检测方式

    ```c++
    static int scan_status() {
        if (getppid() == 1) return -1;
        int pid = -1;
        char line[PATH_MAX];
        char maps[] = "/proc/self/status";
        int fd = sys_open(maps, O_RDONLY, 0);
        if (fd < 0) {
            LOGE("cannot open %s", maps);
            return -1;
        }
        FILE *fp = fdopen(fd, "r");
        if (fp == NULL) {
            LOGE("cannot open %s", maps);
            close(fd);
            return -1;
        }
        // 遍历status文件查看TracerPid的值为否为0
        while (fgets(line, PATH_MAX - 1, fp) != NULL) {
            if (strncmp(line, "TracerPid", 9) == 0) {
                pid = atoi(&line[10]);
                break;
            }
        }
        fclose(fp);
        return pid;
    }
    ```

- 思路

    检测逻辑比较简单，比较maps中的TracerPid是否为0，更有意思的是它的检测时机，首先看看AndroidManifest.xml
    ```shell
    # app/src/main/AndroidManifest.xml
    <application
        android:allowBackup="false"
        android:label="@string/app_name"
        android:supportsRtl="true"
        android:theme="@android:style/Theme.DeviceDefault"
        android:zygotePreloadName="io.github.vvb2060.magiskdetector.AppZygote"
        tools:ignore="AllowBackup,MissingApplicationIcon"
        tools:targetApi="q">
        
        ......

        <service
            android:name=".RemoteService"
            android:isolatedProcess="true"
            android:useAppZygote="true" />

    </application>
    ```
    首先是引用了zygotePreloadName这个属性，它是在Android 4.1后引入的，App可以通过该属性指定在Zygote启动时需要加载自定义的so并缓存到Zygote进程中。然后是isolatedProcess以及useAppZygote这两个属性，isolatedProcess是表示让service独立运行在进程中，而useAppZygote则从名字上能直接看出来，表明是否需要使用AppZygote模式
    >
    有了上面这些属性的开启，就能确保检测进程以AppZygote模式启动，并且也在Zygote启动时加载检测so文件，之所以要以AppZygote的模式启动，也是因为为了要规避Magisk Hide的影响，可以从文档中以下两个方面的解释来看
    - >Magisk Hide的实现核心是ptrace：magiskd跟踪zygote进程，监控fork和clone操作，即关注子进程创建及其线程创建。 被触发后读取/proc/pid/cmdline和uid判断是否为目标进程。 一般应用进程的cmdline在Java设置，此时已有主线程及JVM虚拟机工作线程。 在加载用户代码前，会至少有一次binder调用，使binder线程启动，此操作触发magiskhide卸载挂载
    - > 唯一的例外是app zygote，它和zygote一样通过socket通信，没有binder线程。在设置cmdline和加载用户代码之间没有线程启动， 因此，可以检测是否被ptrace来判断magiskhide的存在，即使不在隐藏列表中，只要开启功能，就会被发现

    可以从进程列表上看
    ```shell
    selene:/ # ps -ef|grep vvb
    u0_a244        6375    607 0 11:16:12 ?     00:00:01 io.github.vvb2060.magiskdetector
    u0_a244        7285    607 0 11:38:22 ?     00:00:00 io.github.vvb2060.magiskdetector_zygote
    u0_i0          7295   7285 1 11:38:22 ?     00:00:00 io.github.vvb2060.magiskdetector:io.github.vvb2060.magiskdetector.RemoteService
    root           7322   6346 3 11:38:35 pts/27 00:00:00 grep vvb
    ```
    启动了名为io.github.vvb2060.magiskdetector_zygote的AppZygote进程，而就是通过这个检测来判断TracerPid