### 前言
相信Magisk对于移动安全从业者来说都不陌生了，我虽然也是一个版本接着一个版本的使用，但是始终没有去摸透Magisk的生态，希望借助之后想写的Magisk系列的文章来深度学习下Magisk，也正如Magisk在其主页所说的那样
>Magisk is a suite of open source software for customizing Android, supporting devices higher than Android 6.0.
Some highlight features:
>
>- MagiskSU: Provide root access for applications
>- Magisk Modules: Modify read-only partitions by installing modules
>- MagiskBoot: The most complete tool for unpacking and repacking Android boot images
>- Zygisk: Run code in every Android applications' processes

Magisk作为一套工具包，它的实现原理（包括它的su实现、boot patch、module机制等等）都是很值得去阅读理解的

### 一、预备知识
- boot.img的组成
- android启动流程
- android secure体系
- linux存储
- ......

### 二、什么是Root？
Android平台的基础是Linux内核，每个应用被严格控制运行在自己的沙盒中，不能越过边界，但是拥有Root权限就意味着你可以绕过内核的权限校验去任意执行你想要的功能。
### 三、Android是如何限制Root的？
在Android4.3之前的版本中，android会给每一个应用分配一个独一无二的ID（也就是user-ID，也称为UID），所以每个应用都有自己的权限边界，这个时候想要拥有Root权限的话，可以通过set-user-ID-root的机制执行su二进制文件来进行提权或者通过setgid/setuid来让自己拥有更多的权限。

但是自从Android4.3推出[Security Enhancements in Android 4.3](https://source.android.com/docs/security/enhancements/enhancements43)之后，堵住了setgid/setuid入口，引入了全新的安全体系（基于强制访问控制(MAC)的SELinux），构成了SEAndroid，进一步定义Android应用沙盒的边界，运行在单独的进程中，所以每个应用都有自己的权限边界，这样即使是进程具有root的能力，SELinux依然可以通过创建安全策略(sepolicy)来限制root进程的能力范围来增强Android的安全性。而在Android4.4之后推出的[Security Enhancements in Android 4.4](https://source.android.com/docs/security/enhancements/enhancements44)，进一步要求Android打开了SELinux的Enforcing模式。

上面这种改变也是因为过去的Android安全机制是基于DAC（自主访问控制）来实现的，其原理就是：进程理论上所拥有的权限与执行它的用户的权限相同，DAC使用了ACL（Access Control List，访问控制列表）来给非管理者用户提供不同的权限，而root用户对文件系统有完全自由的控制权，因此，想办法把自己的权限提升到root用户就可以完成任何事情。而正是因为这种宽松的管理方式，促使MAC（强制访问控制）的诞生，MAC核心思想：即任何进程想在SELinux系统中干任何事情，都必须先在安全策略配置文件中赋予权限，MAC不再像DAC一样简单的把进程分为root/others等，而是每个进程（Subject，主体）和文件（Object，客体）都配置了一个类型（Type），当一个进程去操控（读写等）一个文件时，系统会检测该进程类型是否有对该文件类型的操作权限

例如
```
(base)  ✘ 大慈大悲观世音菩萨  ~/Projects/Android_boot_image_editor   master  as
selene:/ $ ps -efZ|grep miui
u:r:miuibooster:s0             root            943      1 0 17:43:34 ?     00:00:00 miuibooster
u:r:platform_app:s0:c512,c768  u0_a122        1597    611 0 17:43:43 ?     00:00:01 com.miui.miwallpaper
u:r:platform_app:s0:c512,c768  u0_a102        1950    610 0 17:43:44 ?     00:00:06 com.miui.home
u:r:untrusted_app:s0:c234,c256,c512,c768 u0_a234 2212 610 0 17:43:45 ?     00:00:00 com.miui.weather2
u:r:platform_app:s0:c512,c768  u0_a163        2572    611 0 17:43:49 ?     00:00:00 com.miui.voiceassist
u:r:system_app:s0              system         2616    610 0 17:43:49 ?     00:00:00 com.miui.contentcatcher
u:r:system_app:s0              system         2667    610 0 17:43:49 ?     00:00:02 com.miui.daemon
u:r:system_app:s0              system         2794    610 0 17:43:49 ?     00:00:00 com.miui.face
u:r:untrusted_app:s0:c512,c768 u0_a78         2902    610 0 17:43:50 ?     00:00:00 com.miui.personalassistant
u:r:system_app:s0              system         2971    610 0 17:43:50 ?     00:00:00 com.miui.notification:remote
```
可以看到，像miuibooster、platform_app这样即表示进程归属的type，而具体的type的权限可以从[官方的te文件中查找](https://android.googlesource.com/platform/external/sepolicy/+/jb-mr1-dev/app.te)

因此，在Android4.4之后，获取Root面临的困难是先DAC、后MAC的访问权限控制，市面上通用的做法是修改sepolicy获得一个不受限制的SELinux context，当拥有这个context之后，就可以修改init.rc启动类似su daemon的服务，这样保障了系统运行时后台随时都存在一个拥有root权限的服务，剩下需要做的就只是考虑该如何和这个daemon进行通信

### 四、Magisk是如何工作的？
有了上面对于android权限访问控制体系以及现阶段Root实现方式的了解，我们大概能猜到Magisk是如何实现Root的了？那么Magisk它具体的实现包括

- 如何修改sepolicy、init.rc？
- 如何做到systemless的？
- 如何适配多种版本、机型？
- 是否具备扩展功能？
- ......

是如何实现的呢，接下来通过源码来分析下

下面正式开始分析Magisk的内部工作原理（大家都知道Magisk在v24.1之后推出了Zygisk的模式，为了避免新增部分影响我们对于原始流程的分析，因此我选择先忽略掉这部分，以前一个版本v23.0的源码来作为样本阅读）

#### 1 patch boot
Magisk Manager做的第一步就是对boot的修补，所以第一步就从Magisk Manager的修补boot页面开始追下来
```kt
// com/topjohnwu/magisk/ui/install/InstallFragment.kt
// 对应layout
class InstallFragment : BaseUIFragment<InstallViewModel, FragmentInstallMd2Binding>() {
    override val layoutRes = R.layout.fragment_install_md2
    override val viewModel by viewModel<InstallViewModel>()
    ......
}
// layout/fragment_install_md2.xml
// layout中的开始按钮，对应的方法是InstallViewModel中的install方法
<Button
    style="@style/WidgetFoundation.Button.Text"
    gone="@{viewModel.step != 1}"
    isEnabled="@{viewModel.method == @id/method_patch ? viewModel.data != null : viewModel.method != -1}"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:onClick="@{() -> viewModel.install()}"
    android:text="@string/install_start"
    app:icon="@drawable/ic_forth_md2"
    app:iconGravity="textEnd" />
// com/topjohnwu/magisk/ui/install/InstallViewModel.kt
// 引导出FlashFragment
fun install() {
    when (method) {
        R.id.method_patch -> FlashFragment.patch(data!!).navigate()
        R.id.method_direct -> FlashFragment.flash(false).navigate()
        R.id.method_inactive_slot -> FlashFragment.flash(true).navigate()
        else -> error("Unknown value")
    }
    state = State.LOADING
}
// com/topjohnwu/magisk/ui/flash/FlashFragment.kt
// 类似onCreate方法，触发startFlashing
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    super.onViewCreated(view, savedInstanceState)
    defaultOrientation = activity.requestedOrientation
    activity.requestedOrientation = ActivityInfo.SCREEN_ORIENTATION_NOSENSOR
    viewModel.startFlashing()
}
// com/topjohnwu/magisk/ui/flash/FlashViewModel.kt
Const.Value.PATCH_FILE -> {
    uri ?: return@launch
    showReboot = false
    MagiskInstaller.Patch(uri, outItems, logItems).exec()
}
// com/topjohnwu/magisk/core/tasks/MagiskInstaller.kt
// patchFile也就是传入的原生boot.img
protected fun doPatchFile(patchFile: Uri) = extractFiles() && handleFile(patchFile)
```
进入到关键类：MagiskInstaller
```kt
private fun extractFiles(): Boolean {
    ......
    // 创建/data/data/package_name/install目录
    installDir = File(context.filesDir.parent, "install")
    installDir.deleteRecursively()
    installDir.mkdirs()

    try {
        // Extract binaries
        // 从stub或者full中获取so文件
        if (isRunningAsStub) {
            val zf = ZipFile(DynAPK.current(context))
            zf.entries().asSequence().filter {
                !it.isDirectory && it.name.startsWith("lib/${Const.CPU_ABI_32}/")
            }.forEach {
                val n = it.name.substring(it.name.lastIndexOf('/') + 1)
                val name = n.substring(3, n.length - 3)
                val dest = File(installDir, name)
                zf.getInputStream(it).writeTo(dest)
            }
        } else {
            // 获取lib库中的so文件
            val libs = Const.NATIVE_LIB_DIR.listFiles { _, name ->
                name.startsWith("lib") && name.endsWith(".so")
            } ?: emptyArray()
            for (lib in libs) {
                // 重命名so文件并做软链，例如libmagiskboot.so->magiskboot
                // 并软链到/data/data/package_name/install/magiskboot
                val name = lib.name.substring(3, lib.name.length - 3)
                Os.symlink(lib.path, "$installDir/$name")
            }
        }

        // Extract scripts
        // 从asset目录中抽出三个shell脚本
        for (script in listOf("util_functions.sh", "boot_patch.sh", "addon.d.sh")) {
            val dest = File(installDir, script)
            context.assets.open(script).writeTo(dest)
        }
        // Extract chromeos tools
        // 同理
        File(installDir, "chromeos").mkdir()
        for (file in listOf("futility", "kernel_data_key.vbprivk", "kernel.keyblock")) {
            val name = "chromeos/$file"
            val dest = File(installDir, name)
            context.assets.open(name).writeTo(dest)
        }
    } catch (e: Exception) {
        console.add("! Unable to extract files")
        Timber.e(e)
        return false
    }
    ......
}
```
extractFiles这一步做的功能就是准备资源，把so文件变成可执行文件以及准备好shell脚本，根据Apk对应下的目录可看到文件如下：
```shell
(base)  大慈大悲观世音菩萨  ~/Downloads/Magisk-v23.0 (1)  ll assets
total 88
-rw-rw-rw-@ 1 tcc0lin  staff   3.4K  1  1  1981 addon.d.sh
-rw-rw-rw-@ 1 tcc0lin  staff   5.3K  1  1  1981 boot_patch.sh
drwxr-xr-x@ 5 tcc0lin  staff   160B  5 31 09:08 chromeos
-rw-rw-rw-@ 1 tcc0lin  staff   4.6K  1  1  1981 uninstaller.sh
-rw-rw-rw-@ 1 tcc0lin  staff    22K  1  1  1981 util_functions.sh
(base)  大慈大悲观世音菩萨  ~/Downloads/Magisk-v23.0 (1)  ll lib/armeabi-v7a
total 4472
-rw-rw-rw-@ 1 tcc0lin  staff   1.4M  1  1  1981 libbusybox.so
-rw-rw-rw-@ 1 tcc0lin  staff   102K  1  1  1981 libmagisk32.so
-rw-rw-rw-@ 1 tcc0lin  staff   166K  1  1  1981 libmagisk64.so
-rw-rw-rw-@ 1 tcc0lin  staff   272K  1  1  1981 libmagiskboot.so
-rw-rw-rw-@ 1 tcc0lin  staff   302K  1  1  1981 libmagiskinit.so
```
下一步是对传入的boot.img的具体处理
```kt
private fun handleFile(uri: Uri): Boolean {
    val outStream: OutputStream
    var outFile: MediaStoreUtils.UriFile? = null

    // Process input file
    try {
        uri.inputStream().buffered().use { src ->
            src.mark(500)
            ......
            // 随机给新boot起名
            val alpha = "abcdefghijklmnopqrstuvwxyz"
            val alphaNum = "$alpha${alpha.toUpperCase(Locale.ROOT)}0123456789"
            val random = SecureRandom()
            val filename = StringBuilder("magisk_patched-${BuildConfig.VERSION_CODE}_").run {
                for (i in 1..5) {
                    append(alphaNum[random.nextInt(alphaNum.length)])
                }
                toString()
            }

            outStream = if (magic.contentEquals("ustar".toByteArray())) {
                // tar file
                outFile = MediaStoreUtils.getFile("$filename.tar", true)
                processTar(src, outFile!!.uri.outputStream())
            } else {
                // raw image
                // 以处理boot.img为例
                srcBoot = installDirFile("boot.img")
                console.add("- Copying image to cache")
                src.cleanPump(SuFileOutputStream.open(srcBoot))
                outFile = MediaStoreUtils.getFile("$filename.img", true)
                outFile!!.uri.outputStream()
            }
        }
    } catch (e: IOException) {
        console.add("! Process error")
        outFile?.delete()
        Timber.e(e)
        return false
    }

    // Patch file
    if (!patchBoot()) {
        outFile!!.delete()
        return false
    }
    ......
}

private fun patchBoot(): Boolean {
    var isSigned = false
    if (srcBoot.let { it !is SuFile || !it.isCharacter }) {
        try {
            SuFileInputStream.open(srcBoot).use {
                // AVB验证检测
                if (SignBoot.verifySignature(it, null)) {
                    isSigned = true
                    console.add("- Boot image is signed with AVB 1.0")
                }
            }
        } catch (e: IOException) {
            console.add("! Unable to check signature")
            Timber.e(e)
            return false
        }
    }

    // 新建new-boot.img
    val newBoot = installDirFile("new-boot.img")
    if (!useRootDir) {
        // Create output files before hand
        newBoot.createNewFile()
        File(installDir, "stock_boot.img").createNewFile()
    }

    // 修补boot的执行脚本
    val cmds = arrayOf(
        "cd $installDir",
        "KEEPFORCEENCRYPT=${Config.keepEnc} " +
        "KEEPVERITY=${Config.keepVerity} " +
        "RECOVERYMODE=${Config.recovery} " +
        "sh boot_patch.sh $srcBoot")

    if (!cmds.sh().isSuccess)
        return false

    // 环境清理
    val job = shell.newJob().add("./magiskboot cleanup", "cd /")

    // boot.img签名
    if (isSigned) {
        console.add("- Signing boot image with verity keys")
        val signed = File.createTempFile("signed", ".img", context.cacheDir)
        try {
            val src = SuFileInputStream.open(newBoot).buffered()
            val out = signed.outputStream().buffered()
            withStreams(src, out) { _, _ ->
                SignBoot.doSignature(null, null, src, out, "/boot")
            }
        } catch (e: IOException) {
            console.add("! Unable to sign image")
            Timber.e(e)
            return false
        }
        job.add("cat $signed > $newBoot", "rm -f $signed")
    }
    job.exec()
    return true
}
```
从上面的分析流程可以看出来，虽然流程涉及到的代码比较复杂，但是单从最终涉及到文件来看，实际上就是依赖于lib中的so文件以及boot_patch.sh脚本，其他的关于AVB验证和非正常boot格式的都可以先忽略
##### 1.1 boot_patch分块解析
从boot_patch.sh的注释说明中能很清晰的看出可以分成六块内容
- Initialization
    ```sh
    # 判断是否已经加载完资源
    if [ -z $SOURCEDMODE ]; then
    # Switch to the location of the script file
    cd "$(getdir "${BASH_SOURCE:-$0}")"
    # Load utility functions
    # 执行util_functions.sh加载函数到当前脚本，同时配置环境变量
    . ./util_functions.sh
    # Check if 64-bit
    api_level_arch_detect
    fi

    # 获取传入的boot.img
    BOOTIMAGE="$1"
    [ -e "$BOOTIMAGE" ] || abort "$BOOTIMAGE does not exist!"

    # 通过nanddump指令将boot.img dump成字符
    # Dump image for MTD/NAND character device boot partitions
    if [ -c "$BOOTIMAGE" ]; then
    nanddump -f boot.img "$BOOTIMAGE"
    BOOTNAND="$BOOTIMAGE"
    BOOTIMAGE=boot.img
    fi

    # Flags
    [ -z $KEEPVERITY ] && KEEPVERITY=false
    [ -z $KEEPFORCEENCRYPT ] && KEEPFORCEENCRYPT=false
    [ -z $RECOVERYMODE ] && RECOVERYMODE=false
    export KEEPVERITY
    export KEEPFORCEENCRYPT

    chmod -R 755 .
    ```
- Unpack
    ```sh
    CHROMEOS=false

    ui_print "- Unpacking boot image"
    # 调用magiskboot来解包boot.img
    ./magiskboot unpack "$BOOTIMAGE"

    case $? in
    0 ) ;;
    1 )
        abort "! Unsupported/Unknown image format"
        ;;
    2 )
        ui_print "- ChromeOS boot image detected"
        CHROMEOS=true
        ;;
    * )
        abort "! Unable to unpack boot image"
        ;;
    esac

    [ -f recovery_dtbo ] && RECOVERYMODE=true
    ```
    在unpack阶段，引入了先前资源准备阶段的一个关键so文件：magiskboot
- Ramdisk Restores
    ```sh
    # Test patch status and do restore
    ui_print "- Checking ramdisk status"
    if [ -e ramdisk.cpio ]; then
        ./magiskboot cpio ramdisk.cpio test
        STATUS=$?
    else
        # Stock A only system-as-root
        STATUS=0
    fi
    case $((STATUS & 3)) in
    0 )  # Stock boot
        ui_print "- Stock boot image detected"
        SHA1=$(./magiskboot sha1 "$BOOTIMAGE" 2>/dev/null)
        cat $BOOTIMAGE > stock_boot.img
        # 这个阶段主要是复制ramdisk，作用是为了保存原生init
        cp -af ramdisk.cpio ramdisk.cpio.orig 2>/dev/null
        ;;
    1 )  # Magisk patched
        ui_print "- Magisk patched boot image detected"
        # Find SHA1 of stock boot image
        [ -z $SHA1 ] && SHA1=$(./magiskboot cpio ramdisk.cpio sha1 2>/dev/null)
        ./magiskboot cpio ramdisk.cpio restore
        cp -af ramdisk.cpio ramdisk.cpio.orig
        rm -f stock_boot.img
        ;;
    2 )  # Unsupported
        ui_print "! Boot image patched by unsupported programs"
        abort "! Please restore back to stock boot image"
        ;;
    esac
    ```
- Ramdisk Patches
    上一阶段做了ramdisk的备份，这个阶段开始对ramdisk进行patch
    ```sh
    ui_print "- Patching ramdisk"

    echo "KEEPVERITY=$KEEPVERITY" > config
    echo "KEEPFORCEENCRYPT=$KEEPFORCEENCRYPT" >> config
    echo "RECOVERYMODE=$RECOVERYMODE" >> config
    [ ! -z $SHA1 ] && echo "SHA1=$SHA1" >> config

    # 压缩magisk成xz
    # Compress to save precious ramdisk space
    ./magiskboot compress=xz magisk32 magisk32.xz
    ./magiskboot compress=xz magisk64 magisk64.xz
    $IS64BIT && SKIP64="" || SKIP64="#"
    # 对ramdisk进行修改，添加入magisk定制的magiskinit以及magisk，后面会着重讲到
    ./magiskboot cpio ramdisk.cpio \
    "add 0750 init magiskinit" \
    "mkdir 0750 overlay.d" \
    "mkdir 0750 overlay.d/sbin" \
    "add 0644 overlay.d/sbin/magisk32.xz magisk32.xz" \
    "$SKIP64 add 0644 overlay.d/sbin/magisk64.xz magisk64.xz" \
    "patch" \
    "backup ramdisk.cpio.orig" \
    "mkdir 000 .backup" \
    "add 000 .backup/.magisk config"

    rm -f ramdisk.cpio.orig config magisk*.xz
    ```
- Binary Patches
    这一步是对二进制文件的patch，boot.img中的二进制文件主要指的是kernel以及对应的dtb文件
    ```sh
    for dt in dtb kernel_dtb extra; do
        [ -f $dt ] && ./magiskboot dtb $dt patch && ui_print "- Patch fstab in $dt"
    done

    if [ -f kernel ]; then
        # 专门针对三星设备的patch
        # Remove Samsung RKP
        # 绕过三星内核保护，参考文章：https://www.anquanke.com/post/id/85627
        ./magiskboot hexpatch kernel \
        49010054011440B93FA00F71E9000054010840B93FA00F7189000054001840B91FA00F7188010054 \
        A1020054011440B93FA00F7140020054010840B93FA00F71E0010054001840B91FA00F7181010054

        # Remove Samsung defex
        # 三星设备防护，参考文章：https://www.99mediasector.com/how-to-disable-defex-security-to-root-samsung-galaxy-phones-oreo/
        # Before: [mov w2, #-221]   (-__NR_execve)
        # After:  [mov w2, #-32768]
        ./magiskboot hexpatch kernel 821B8012 E2FF8F12

        # Force kernel to load rootfs
        # skip_initramfs -> want_initramfs
        # skip_initramfs属于内核启动参数cmdline中，与system-as-root有关，将skip_initramfs更改为want_initramfs，实际上可以认为是去除skip_initramfs这个参数
        ./magiskboot hexpatch kernel \
        736B69705F696E697472616D667300 \
        77616E745F696E697472616D667300
    fi
    ```
- Repack & Flash
    ```sh
    ui_print "- Repacking boot image"
    # 上面已经修改好了ramdisk和kernel、dtb，现在开始重新打包成boot.img
    ./magiskboot repack "$BOOTIMAGE" || abort "! Unable to repack boot image"

    # Sign chromeos boot
    $CHROMEOS && sign_chromeos

    # Restore the original boot partition path
    [ -e "$BOOTNAND" ] && BOOTIMAGE="$BOOTNAND"

    # Reset any error code
    true
    ```

##### 1.2 magiskboot的作用
boot_patch脚本中多次出现了magiskboot，下面根据上述提到的方法来看看其具体的实现，位置在native/jni/magiskboot
###### 1.2.1 main

头文件中定义了boot_patch中所使用到的方法，由main.cpp来处理action
```c++
// native/jni/magiskboot/magiskboot.hpp
#define HEADER_FILE     "header"
#define KERNEL_FILE     "kernel"
#define RAMDISK_FILE    "ramdisk.cpio"
#define SECOND_FILE     "second"
#define EXTRA_FILE      "extra"
#define KER_DTB_FILE    "kernel_dtb"
#define RECV_DTBO_FILE  "recovery_dtbo"
#define DTB_FILE        "dtb"
#define NEW_BOOT        "new-boot.img"

int unpack(const char *image, bool skip_decomp = false, bool hdr = false);
void repack(const char *src_img, const char *out_img, bool skip_comp = false);
int split_image_dtb(const char *filename);
int hexpatch(const char *image, const char *from, const char *to);
int cpio_commands(int argc, char *argv[]);
int dtb_commands(int argc, char *argv[]);

uint32_t patch_verity(void *buf, uint32_t size);
uint32_t patch_encryption(void *buf, uint32_t size);
bool check_env(const char *name);
// native/jni/magiskboot/main.cpp
int main(int argc, char *argv[]) {
    cmdline_logging();
    umask(0);

    if (argc < 2)
        usage(argv[0]);

    // Skip '--' for backwards compatibility
    string_view action(argv[1]);
    if (str_starts(action, "--"))
        action = argv[1] + 2;

    if (action == "cleanup") {
        ......
    } else if (argc > 2 && action == "repack") {
        if (argv[2] == "-n"sv) {
            if (argc == 3)
                usage(argv[0]);
            repack(argv[3], argv[4] ? argv[4] : NEW_BOOT, true);
        } else {
            repack(argv[2], argv[3] ? argv[3] : NEW_BOOT);
        }
    } else if (argc > 2 && action == "decompress") {
        decompress(argv[2], argv[3]);
    } else if (argc > 2 && str_starts(action, "compress")) {
        compress(action[8] == '=' ? &action[9] : "gzip", argv[2], argv[3]);
    } else if (argc > 4 && action == "hexpatch") {
        return hexpatch(argv[2], argv[3], argv[4]);
    } else if (argc > 2 && action == "cpio"sv) {
        if (cpio_commands(argc - 2, argv + 2))
            usage(argv[0]);
    } else if (argc > 3 && action == "dtb") {
        if (dtb_commands(argc - 2, argv + 2))
            usage(argv[0]);
    } else {
        usage(argv[0]);
    }
    return 0;
}
```
###### 1.2.2 unpack
uppack方法依赖于对boot.img镜像的解析得到的boot_img结构体，再根据具体偏移量dump出对应文件
```c++
// native/jni/magiskboot/bootimg.cpp
int unpack(const char *image, bool skip_decomp, bool hdr) {
    // 对传入的boot.img镜像转换成boot_img结构体，当生成boot_img结构体后
    // 就可以根据不同类型文件的addr和size来进行文件的dump
    boot_img boot(image);
    // Dump kernel
    dump(boot.kernel, boot.hdr->kernel_size(), KERNEL_FILE);
    // Dump kernel_dtb
    dump(boot.kernel_dtb, boot.hdr->kernel_dt_size, KER_DTB_FILE);
    // Dump ramdisk
    dump(boot.ramdisk, boot.hdr->ramdisk_size(), RAMDISK_FILE);
    // Dump second
    dump(boot.second, boot.hdr->second_size(), SECOND_FILE);
    ......
}
```
这里着重看看转化成boot_img结构体的过程
```c++
// native/jni/magiskboot/bootimg.cpp
boot_img::boot_img(const char *image) {
    // 将boot.img文件映射到只读内存区域，同时会赋值给map_addr和map_size
    mmap_ro(image, map_addr, map_size);
    fprintf(stderr, "Parsing boot image: [%s]\n", image);
    // 根据不同文件的magic header来区分
    for (uint8_t *addr = map_addr; addr < map_addr + map_size; ++addr) {
        // 检查文件类型
        format_t fmt = check_fmt(addr, map_size);
        switch (fmt) {
        case CHROMEOS:
            // chromeos require external signing
            flags[CHROMEOS_FLAG] = true;
            addr += 65535;
            break;
        case DHTB:
            flags[DHTB_FLAG] = true;
            flags[SEANDROID_FLAG] = true;
            fprintf(stderr, "DHTB_HDR\n");
            addr += sizeof(dhtb_hdr) - 1;
            break;
        case BLOB:
            flags[BLOB_FLAG] = true;
            fprintf(stderr, "TEGRA_BLOB\n");
            addr += sizeof(blob_hdr) - 1;
            break;
        case AOSP:
        case AOSP_VENDOR:
            // AOSP为kernel相关
            parse_image(addr, fmt);
            return;
        default:
            break;
        }
    }
    exit(1);
}
// native/jni/magiskboot/format.cpp
format_t check_fmt(const void *buf, size_t len) {
    if (CHECKED_MATCH(CHROMEOS_MAGIC)) {
        return CHROMEOS;
    } else if (CHECKED_MATCH(BOOT_MAGIC)) {
        return AOSP;
    } else if (CHECKED_MATCH(VENDOR_BOOT_MAGIC)) {
        return AOSP_VENDOR;
    } 
    ......
}
// native/jni/magiskboot/format.hpp
#define BOOT_MAGIC      "ANDROID!"
#define VENDOR_BOOT_MAGIC "VNDRBOOT"
#define CHROMEOS_MAGIC  "CHROMEOS"
#define GZIP1_MAGIC     "\x1f\x8b"
#define GZIP2_MAGIC     "\x1f\x9e"
#define LZOP_MAGIC      "\x89""LZO"
#define XZ_MAGIC        "\xfd""7zXZ"
#define BZIP_MAGIC      "BZh"
```
解析kernel以及ramdisk size的过程
```c++
// native/jni/magiskboot/bootimg.cpp
void boot_img::parse_image(uint8_t *addr, format_t type) {
    // 首先获取到boot.img的header结构体
    auto hp = reinterpret_cast<boot_img_hdr*>(addr);
    if (type == AOSP_VENDOR) {
        fprintf(stderr, "VENDOR_BOOT_HDR\n");
        hdr = new dyn_img_vnd_v3(addr);
    } else if (hp->page_size >= 0x02000000) {
        fprintf(stderr, "PXA_BOOT_HDR\n");
        hdr = new dyn_img_pxa(addr);
    } else {
        ......
        // 根据header的版本选择对应的处理方式
        switch (hp->header_version) {
        case 1:
            hdr = new dyn_img_v1(addr);
            break;
        case 2:
            hdr = new dyn_img_v2(addr);
            break;
        case 3:
            hdr = new dyn_img_v3(addr);
            break;
        default:
            hdr = new dyn_img_v0(addr);
            break;
        }
    }
    ......
}

// Default to hdr v2，代码仿照android原生bootimg.h，也就是根据官方的逻辑来获取size，参考https://android.googlesource.com/platform/system/tools/mkbootimg/+/refs/heads/master/include/bootimg/bootimg.h
using boot_img_hdr = boot_img_hdr_v2;
```
上面可以看到在处理boot.img header阶段时有不同版本的header，截止到android13时boot.img header一共存有5个版本：[Boot Image Header](https://source.android.com/docs/core/architecture/bootloader/boot-image-header#header-v0)，header之所以会产生迭代是因为boot.img中的文件是一直存在变化的，包括v1版本出现的recovery image，v2版本出现的dtb等等

执行完unpack方法后就得到了boot.img解包后目录，目录下的文件可参考
```c++
// native/jni/magiskboot/magiskboot.hpp
#define HEADER_FILE     "header"
#define KERNEL_FILE     "kernel"
#define RAMDISK_FILE    "ramdisk.cpio"
#define SECOND_FILE     "second"
#define EXTRA_FILE      "extra"
#define KER_DTB_FILE    "kernel_dtb"
#define RECV_DTBO_FILE  "recovery_dtbo"
#define DTB_FILE        "dtb"
#define NEW_BOOT        "new-boot.img"
```
###### 1.2.3 cpio
在boot_patch中cpio方法主要使用到的地方在于ramdisk patch，也就是下面这个脚本
```sh
./magiskboot cpio ramdisk.cpio \
"add 0750 init magiskinit" \
"mkdir 0750 overlay.d" \
"mkdir 0750 overlay.d/sbin" \
"add 0644 overlay.d/sbin/magisk32.xz magisk32.xz" \
"$SKIP64 add 0644 overlay.d/sbin/magisk64.xz magisk64.xz" \
"patch" \
"backup ramdisk.cpio.orig" \
"mkdir 000 .backup" \
"add 000 .backup/.magisk config"
```
cpio的入口是cpio_commands
```c++
if (cmdv[0] == "test"sv) {
    exit(cpio.test());
} else if (cmdv[0] == "restore"sv) {
    cpio.restore();
} else if (cmdv[0] == "sha1"sv) {
    char *sha1 = cpio.sha1();
    if (sha1) printf("%s\n", sha1);
    return 0;
} else if (cmdv[0] == "compress"sv){
    cpio.compress();
} else if (cmdv[0] == "decompress"sv){
    cpio.decompress();
} else if (cmdv[0] == "patch"sv) {
    cpio.patch();
} else if (cmdc == 2 && cmdv[0] == "exists"sv) {
    exit(!cpio.exists(cmdv[1]));
} else if (cmdc == 2 && cmdv[0] == "backup"sv) {
    cpio.backup(cmdv[1]);
} else if (cmdc >= 2 && cmdv[0] == "rm"sv) {
    bool r = cmdc > 2 && cmdv[1] == "-r"sv;
    cpio.rm(cmdv[1 + r], r);
} else if (cmdc == 3 && cmdv[0] == "mv"sv) {
    cpio.mv(cmdv[1], cmdv[2]);
} else if (cmdv[0] == "extract"sv) {
    if (cmdc == 3) {
        return !cpio.extract(cmdv[1], cmdv[2]);
    } else {
        cpio.extract();
        return 0;
    }
} else if (cmdc == 3 && cmdv[0] == "mkdir"sv) {
    cpio.mkdir(strtoul(cmdv[1], nullptr, 8), cmdv[2]);
} else if (cmdc == 3 && cmdv[0] == "ln"sv) {
    cpio.ln(cmdv[1], cmdv[2]);
} else if (cmdc == 4 && cmdv[0] == "add"sv) {
    cpio.add(strtoul(cmdv[1], nullptr, 8), cmdv[2], cmdv[3]);
} else {
    return 1;
}
```
根据cpio给出的指令解析，可以将所执行的shell命令划分为如下
```shell
"add 0750 init magiskinit" \
"mkdir 0750 overlay.d" \
"mkdir 0750 overlay.d/sbin" \
"add 0644 overlay.d/sbin/magisk32.xz magisk32.xz" \
"add 0644 overlay.d/sbin/magisk64.xz magisk64.xz" \
"patch" \
"backup ramdisk.cpio.orig" \
"mkdir 000 .backup" \
"add 000 .backup/.magisk config"
```
也就是add、mkdir、patch、backup这几个方法，在执行方法前，会先进行cpio文件的加载
```c++
// native/jni/magiskboot/ramdisk.cpp
char *incpio = argv[0];
++argv;
--argc;

magisk_cpio cpio;
if (access(incpio, R_OK) == 0)
    cpio.load_cpio(incpio);
// native/jni/utils/cpio.cpp
void cpio_rw::load_cpio(const char *file) {
    char *buf;
    size_t sz;
    // 读取ramdisk到只读内存区域
    mmap_ro(file, buf, sz);
    fprintf(stderr, "Loading cpio: [%s]\n", file);
    // 加载
    load_cpio(buf, sz);
    // 释放上面的内存区域
    munmap(buf, sz);
}

void cpio_rw::load_cpio(const char *buf, size_t sz) {
    size_t pos = 0;
    const cpio_newc_header *header;
    unique_ptr<cpio_entry> entry;
    while (pos < sz) {
        header = reinterpret_cast<const cpio_newc_header *>(buf + pos);
        entry = make_unique<cpio_entry>(header);
        pos += sizeof(*header);
        string_view name_view(buf + pos);
        pos += x8u(header->namesize);
        pos_align(pos);
        if (name_view == "." || name_view == "..")
            continue;
        if (name_view == "TRAILER!!!")
            break;
        entry->filename = name_view;
        entry->data = xmalloc(entry->filesize);
        memcpy(entry->data, buf + pos, entry->filesize);
        pos += entry->filesize;
        // 这步是关键，等于是将文件都存入数据当中，便于后续来做替换以及插入
        entries[entry->filename] = std::move(entry);
        pos_align(pos);
    }
}
```
load完成之后，得到了magisk_cpio的实例，下面就可以根据脚本来进行patch
- add
    ```c++
    // native/jni/utils/cpio.cpp
    void cpio_rw::add(mode_t mode, const char *name, const char *file) {
        void *buf;
        size_t sz;
        // 映射到只读区域
        mmap_ro(file, buf, sz);
        // 创建entry对象
        auto e = new cpio_entry(name, S_IFREG | mode);
        // 配置属性
        e->filesize = sz;
        e->data = xmalloc(sz);
        // 将buf存入data属性
        memcpy(e->data, buf, sz);
        munmap(buf, sz);
        // 插入ramdisk中
        insert(e);
        fprintf(stderr, "Add entry [%s] (%04o)\n", name, mode);
    }

    void cpio_rw::insert(cpio_entry *e) {
        // 判断文件是否存在
        auto ex = entries.extract(e->filename);
        // 存在即替换
        if (!ex) {
            entries[e->filename].reset(e);
        } else {
            // 不存在直接插入
            ex.key() = e->filename;
            ex.mapped().reset(e);
            entries.insert(std::move(ex));
        }
    }
    ```
- mkdir
    ```c++
    // 类似linux mkdir
    void cpio_rw::mkdir(mode_t mode, const char *name) {
        insert(new cpio_entry(name, S_IFDIR | mode));
        fprintf(stderr, "Create directory [%s] (%04o)\n", name, mode);
    }
    ```
- patch
    ```c++
    // 根据传入的KEEPVERITY等来判断是否解fstab锁，默认是需要解锁的
    void magisk_cpio::patch() {
        bool keepverity = check_env("KEEPVERITY");
        bool keepforceencrypt = check_env("KEEPFORCEENCRYPT");
        fprintf(stderr, "Patch with flag KEEPVERITY=[%s] KEEPFORCEENCRYPT=[%s]\n",
                keepverity ? "true" : "false", keepforceencrypt ? "true" : "false");

        for (auto it = entries.begin(); it != entries.end();) {
            auto cur = it++;
            bool fstab = (!keepverity || !keepforceencrypt) &&
                        S_ISREG(cur->second->mode) &&
                        !str_starts(cur->first, ".backup") &&
                        !str_contains(cur->first, "twrp") &&
                        !str_contains(cur->first, "recovery") &&
                        str_contains(cur->first, "fstab");
            if (!keepverity) {
                if (fstab) {
                    fprintf(stderr, "Found fstab file [%s]\n", cur->first.data());
                    cur->second->filesize = patch_verity(cur->second->data, cur->second->filesize);
                } else if (cur->first == "verity_key") {
                    rm(cur);
                    continue;
                }
            }
            if (!keepforceencrypt) {
                if (fstab) {
                    cur->second->filesize = patch_encryption(cur->second->data, cur->second->filesize);
                }
            }
        }
    }
    ```
- backup
    ```c++
    // native/jni/magiskboot/ramdisk.cpp
    void magisk_cpio::backup(const char *orig) {
        if (access(orig, R_OK))
            return;
        entry_map bkup_entries;
        string remv;

        // 新建.backup目录
        auto b = new cpio_entry(".backup", S_IFDIR);
        bkup_entries[b->filename].reset(b);

        magisk_cpio o(orig);

        // 删除原有ramdisk可能存有的.backup
        // Remove possible backups in original ramdisk
        o.rm(".backup", true);
        rm(".backup", true);
        // 这里分别取原生的ramdisk以及刚刚patch的ramdisk
        auto lhs = o.entries.begin();
        auto rhs = entries.begin();

        while (lhs != o.entries.end() || rhs != entries.end()) {
            int res;
            bool backup = false;
            if (lhs != o.entries.end() && rhs != entries.end()) {
                res = lhs->first.compare(rhs->first);
            } else if (lhs == o.entries.end()) {
                res = 1;
            } else {
                res = -1;
            }

            if (res < 0) {
                // Something is missing in new ramdisk, backup!
                backup = true;
                fprintf(stderr, "Backup missing entry: ");
            } else if (res == 0) {
                // 这里表示发现两者不一样的地方，特别针对init文件
                if (lhs->second->filesize != rhs->second->filesize ||
                    memcmp(lhs->second->data, rhs->second->data, lhs->second->filesize) != 0) {
                    // Not the same!
                    backup = true;
                    fprintf(stderr, "Backup mismatch entry: ");
                }
            } else {
                // Something new in ramdisk
                remv += rhs->first;
                remv += (char) '\0';
                fprintf(stderr, "Record new entry: [%s] -> [.backup/.rmlist]\n", rhs->first.data());
            }
            if (backup) {
                // 将init等文件放在.backup目录下
                string back_name(".backup/");
                back_name += lhs->first;
                fprintf(stderr, "[%s] -> [%s]\n", lhs->first.data(), back_name.data());
                auto ex = static_cast<cpio_entry*>(lhs->second.release());
                ex->filename = back_name;
                bkup_entries[ex->filename].reset(ex);
            }
        ......
    }
    ```
结合一开始提到的那段shell命令，可以看出在ramdisk patch这个阶段所做的是就是
1. 使用定制的magiskinit替换原生init
2. 创建overlay.d/sbin目录并将定制的magisk复制到目录下
3. 解锁fstab
4. 备份ramdisk，主要为了备份init
5. 创建.backup并将config配置文件复制到目录下
###### 1.2.4 dtb
dtb的部分是为了要去除fsmgr_flags的校验
```c++
// native/jni/magiskboot/dtb.cpp
static bool dtb_patch(const char *file) {
    bool keep_verity = check_env("KEEPVERITY");

    size_t size;
    uint8_t *dtb;
    fprintf(stderr, "Loading dtbs from [%s]\n", file);
    mmap_rw(file, dtb, size);

    bool patched = false;
    uint8_t * const end = dtb + size;
    for (uint8_t *fdt = dtb; fdt < end;) {
        fdt = static_cast<uint8_t*>(memmem(fdt, end - fdt, DTB_MAGIC, sizeof(fdt32_t)));
        if (fdt == nullptr)
            break;
        if (int fstab = find_fstab(fdt); fstab >= 0) {
            int node;
            fdt_for_each_subnode(node, fdt, fstab) {
                if (!keep_verity) {
                    int len;
                    char *value = (char *) fdt_getprop(fdt, node, "fsmgr_flags", &len);
                    patched |= patch_verity(value, len) != len;
                }
            }
        }
        fdt += fdt_totalsize(fdt);
    }

    munmap(dtb, size);
    return patched;
}
```
###### 1.2.5 hexpatch
```c++
// native/jni/magiskboot/hexpatch.cpp
// 原理就是根据传入的hex来做替换
```
###### 1.2.6 repack
略
###### 1.2.7 patch示例
以miui12的boot.img作为示例
```shell
- Unpacking boot image
Parsing boot image: [boot.img]
# header版本
HEADER_VER      [2]
KERNEL_SZ       [11561996]
RAMDISK_SZ      [20066474]
SECOND_SZ       [0]
RECOV_DTBO_SZ   [0]
DTB_SZ          [123999]
OS_VERSION      [11.0.0]
OS_PATCH_LEVEL  [2022-02]
PAGESIZE        [2048]
NAME            []
CMDLINE         [bootopt=64S3,32N2,64N2]
CHECKSUM        [1c7765429e25833c9a109589cfe6ead8d89bf819000000000000000000000000]
KERNEL_DTB_SZ   [123999]
KERNEL_FMT      [gzip]
RAMDISK_FMT     [gzip]
VBMETA
- Checking ramdisk status
Loading cpio: [ramdisk.cpio]
- Stock boot image detected
- Patching ramdisk
// ramdisk patch部分
Loading cpio: [ramdisk.cpio]
Add entry [init] (0750)
Create directory [overlay.d] (0750)
Create directory [overlay.d/sbin] (0750)
Add entry [overlay.d/sbin/magisk32.xz] (0644)
Add entry [overlay.d/sbin/magisk64.xz] (0644)
Patch with flag KEEPVERITY=[false] KEEPFORCEENCRYPT=[false]
Found fstab file [first_stage_ramdisk/fstab.mt6768]
Remove pattern [,avb=vbmeta_system]
Remove pattern [,avb_keys=/avb/q-gsi.avbpubkey:/avb/r-gsi.avbpubkey:/avb/s-gsi.avbpubkey]
Remove pattern [,avb]
Remove pattern [,avb]
Remove pattern [,avb=vbmeta]
Remove pattern [,fileencryption=aes-256-xts:aes-256-cts:v2]
Found fstab file [miui.factoryreset.fstab]
Remove pattern [,fsverity]
Remove pattern [,fileencryption=aes-256-xts:aes-256-cts:v2+inlinecrypt_optimized]
Remove [verity_key]
Loading cpio: [ramdisk.cpio.orig]
// backup ramdisk.cpio.orig部分，对未同步的再做一次备份
Backup mismatch entry: [first_stage_ramdisk/fstab.mt6768] -> [.backup/first_stage_ramdisk/fstab.mt6768]
Backup mismatch entry: [init] -> [.backup/init]
Backup mismatch entry: [miui.factoryreset.fstab] -> [.backup/miui.factoryreset.fstab]
Record new entry: [overlay.d] -> [.backup/.rmlist]
Record new entry: [overlay.d/sbin] -> [.backup/.rmlist]
Record new entry: [overlay.d/sbin/magisk32.xz] -> [.backup/.rmlist]
Record new entry: [overlay.d/sbin/magisk64.xz] -> [.backup/.rmlist]
Backup missing entry: [verity_key] -> [.backup/verity_key]
Create directory [.backup] (0000)
Add entry [.backup/.magisk] (0000)
Dump cpio: [ramdisk.cpio]
Loading dtbs from [dtb]
Loading dtbs from [kernel_dtb]
// patch skip_参数
Patch @ 017950F8 [736B69705F696E697472616D667300] -> [77616E745F696E697472616D667300]
- Repacking boot image
Parsing boot image: [boot.img]
HEADER_VER      [2]
KERNEL_SZ       [11561996]
RAMDISK_SZ      [20066474]
SECOND_SZ       [0]
RECOV_DTBO_SZ   [0]
DTB_SZ          [123999]
OS_VERSION      [11.0.0]
OS_PATCH_LEVEL  [2022-02]
PAGESIZE        [2048]
NAME            []
CMDLINE         [bootopt=64S3,32N2,64N2]
CHECKSUM        [1c7765429e25833c9a109589cfe6ead8d89bf819000000000000000000000000]
KERNEL_DTB_SZ   [123999]
KERNEL_FMT      [gzip]
RAMDISK_FMT     [gzip]
VBMETA
Repack to boot image: [new-boot.img]
HEADER_VER      [2]
KERNEL_SZ       [11567292]
RAMDISK_SZ      [23699677]
SECOND_SZ       [0]
RECOV_DTBO_SZ   [0]
DTB_SZ          [123999]
OS_VERSION      [11.0.0]
OS_PATCH_LEVEL  [2022-02]
PAGESIZE        [2048]
NAME            []
CMDLINE         [bootopt=64S3,32N2,64N2]
CHECKSUM        [69cf38b1a7b67f4d26f25b8fe87cbaee29dcf565000000000000000000000000]
```
##### 1.3 patch boot流程图
![](https://p3.ssl.qhimg.com/t0129e32587c753c2f5.jpg)

patch boot阶段整体的流程就可以归结为上图，patch点主要在于ramdisk，更精确一点可以说是对init的修改，这里其实是Magisk的很关键一步，围绕boot.img的修改，没有动system.img，所以也就有了Magisk一直宣传的“systemless”的概念，当然，想要完整实现“systemless”还需要依赖于后面booting process过程中对于system的修改

另一方面，可以看出对boot的patch操作可以不仅仅局限于Magisk Manager，只需要准备好资源即可，参照github: magisk patch
#### 2 booting process
##### 2.1 android启动方式的演变

首先是一些术语的解释：
- rootdir：根目录，所有文件、文件夹或文件系统都存储在rootdir中或挂载在rootdir下。在Android上，rootdir可以是rootfs或system
- initramfs：android boot.img中的一部分，内核会将它处理成rootfs，更常见的说法是ramdisk
- recovery/boot partition：这两个分区都包含了ramdisk以及kernel，不同的是一个是正常启动到android环境，另一个则是引导到recovery模式
- SAR：system-as-root机制，也就是将system分区挂载成rootdir，而不是rootfs
- 2SI：两阶段初始化，在android10之后被启用

> 参考[Magisk官方文档](https://topjohnwu.github.io/Magisk/boot.html)

android的启动方式演变到现在可以归结为三类：

|  Method   | Initial rootdir  |Final rootdir  |
|  ----  | ----  |----  |
| A  | rootfs |rootfs |
| B  | system |system |
| C  | rootfs |system |

###### 2.1.1 Method A - 传统的ramdisk
最初版本android启动方式，使用的boot.img header还是最原始版本的，内核将initramfs作为rootdir，同时执行其内部的init文件来启动进而挂载system、userdata等其他分区

这个时期大概是android7.0之前，也就是没有出现A/B分区之前，还存在着recovery分区，recovery分区中也同样具备相同的kernel和独有的ramdisk-recovery
###### 2.1.2 Method B - 传统的SAR机制
android7.0之后，android推出了system-as-root机制，也就是将system分区作为rootdir（这样做的原因是希望将rootdir和android平台绑定，而不是让rootdir和厂商相关部分绑定，厂商部分可以通过修改vendor等定制分区），同时执行其内部的init文件，这样做也导致boot.img中的ramdisk变得无用，因此移除了recovery分区并将其ramdisk-recovery放在boot.img当中

另一方面，system由于是Android sparse image格式没办法直接挂载，通常会处理成ext4再挂载到system_root上，之前在patch binary的时候提到过skip_initramfs参数，这个参数是用来表示是否加载initramfs到rootfs，如果没有这个参数，则直接会加载ramdisk到rootfs，也就是进入了recovery模式
###### 2.1.2 Method C - 两阶段初始化（ramdisk to SAR）
android10之后要求各大厂商都必须要使用two-stage-init的模式，可以从源码中看看处理流程
```c++
// /system/core/init/main.cpp
int main(int argc, char** argv) {
    if (!strcmp(basename(argv[0]), "ueventd")) {
        return ueventd_main(argc, argv);
    }
    if (argc > 1) {
        if (!strcmp(argv[1], "subcontext")) {
            android::base::InitLogging(argv, &android::base::KernelLogger);
            const BuiltinFunctionMap& function_map = GetBuiltinFunctionMap();
 
            return SubcontextMain(argc, argv, &function_map);
        }
 
        if (!strcmp(argv[1], "selinux_setup")) {
            return SetupSelinux(argv);
        }
        if (!strcmp(argv[1], "second_stage")) {
            return SecondStageMain(argc, argv);
        }
    }
    // 在没有任务参数的情况下，会走FirstStageMain方法
    return FirstStageMain(argc, argv);
}
```
这个阶段内核会将rootfs作为rootdir，执行init进程走FirstStageMain方法，而它所做的工作就是挂载system分区并将它作为新的rootdir
```c++
int FirstStageMain(int argc, char** argv) {
    // 挂载分区、挂载/dev、/proc、sys/fs/selinux
    ......
    InitKernelLogging(argv);
    DoFirstStageMount();
 
    // 向下引导走
    const char* path = "/system/bin/init";
    const char* args[] = {path, "selinux_setup", nullptr};
    auto fd = open("/dev/kmsg", O_WRONLY | O_CLOEXEC);
    dup2(fd, STDOUT_FILENO);
    dup2(fd, STDERR_FILENO);
    close(fd);
    execv(path, const_cast<char**>(args));
}

int SetupSelinux(char** argv) {
    SetStdioToDevNull(argv);
    InitKernelLogging(argv);
 
    MountMissingSystemPartitions();
 
    // Set up SELinux, loading the SELinux policy.
    SelinuxSetupKernelLogging();
    SelinuxInitialize();
 
 
    if (selinux_android_restorecon("/system/bin/init", 0) == -1) {
        PLOG(FATAL) << "restorecon failed of /system/bin/init failed";
    }
 
    const char* path = "/system/bin/init";
    const char* args[] = {path, "second_stage", nullptr};
    execv(path, const_cast<char**>(args));
 
    return 1;
}
```
可以看出，当system分区挂载完成之后，FirstStageMain会通过execv带着selinux_setup执行/system/bin/init从而引导出第二阶段的init，接下来就有system中的init去执行剩余的步骤，包括解释执行init.rc等等

注意：第一阶段使用的是ramdisk-recovery中的init，它其实是system中的init的一个软链

以上就是有关于android启动方式的演变，正是由于这么多复杂的启动方式，也就意味着Magisk为了适配全量机型以及其systemless的特性，就要分别对这些场景进行归类处理
##### 2.2 magiskinit的处理
```c++
// native/jni/init/init.cpp
// 多次进入main方法
int main(int argc, char *argv[]) {
    ......
    BaseInit *init;
    cmdline cmd{};
    // 使用参数selinux_setup调用init，表明第一阶段已经完成
    if (argc > 1 && argv[1] == "selinux_setup"sv) {
        setup_klog();
        init = new SecondStageInit(argv);
    } else {
        // This will also mount /sys and /proc
        // 解析cmd命令，cmd命令可以从/proc/cmdline中看到
        load_kernel_info(&cmd);
        // 具体场景区分
        if (cmd.skip_initramfs)
            init = new SARInit(argv, &cmd);
        else if (cmd.force_normal_boot)
            init = new FirstStageInit(argv, &cmd);
        else if (access("/sbin/recovery", F_OK) == 0 || access("/system/bin/recovery", F_OK) == 0)
            init = new RecoveryInit(argv, &cmd);
        else if (check_two_stage())
            init = new FirstStageInit(argv, &cmd);
        else
            init = new RootFSInit(argv, &cmd);
    }
    // 确定好init类型后执行
    // Run the main routine
    init->start();
    exit(1);
}
```

|Type	|Boot Method	|Partition|	2SI|	Ramdisk in boot|
|  ----  | ----  |----  |----  |----  |----  |
|I	|A|	A-only|	No	|boot ramdisk|
|II	|B|	A/B	|Any|	recovery ramdisk|
|III|	B|	A-only|	Any	|N/A|
|IV|	C|	Any|	Yes|	Hybrid ramdisk|

上面的表格是Magisk会区分的四种场景
- Type I
    - 如果要启动的是正常系统，那么会归入最后一种情况进入到init = new RootFSInit(argv, &cmd)
    - 如果要启动的是recovery模式，由于recovery分区并未被magisk修改，直接从recovery分区启动即可
- Type II
    - 如果要启动的是正常系统，由于存在命令行参数skip_initramfs，会进入到init = new SARInit(argv, &cmd)
    - 如果要启动的是recovery模式，不会存在skip_initramfs参数，但由于boot.img中的ramdisk是ramdisk-recovery.img，所以会有/sbin/recovery或者/system/bin/recovery文件，进入到init = new RecoveryInit(argv, &cmd)
- Type III
    - 如果要启动的是正常系统,直接进入无magisk的原始系统
    - 如果要启动的是recovery模式，会导致magiskinit执行，magiskinit读取/.backup/.magisk配置文件得到RECOVERYMODE=true知道它自己是在recovery分区启动，它会调用check_key_combo()判断是否长按音量键上，如果长按则进入magisk系统，否则进入到原来的recovery系统。进入magisk系统仍然进入到init = new SARInit(argv, &cmd)，进入recovery模式会进入init = new RecoveryInit(argv, &cmd)
- Type IV
    - 如果要启动的是正常系统,由于有很多新的配置是随着Method C出现的,如force_normal_boot,/apex目录,所以会通过判断进入到init = new FirstStageInit(argv, &cmd)
    - 如果要启动的是recovery模式,由于存在/system/bin/init文件。仍然会进入到init = new FirstStageInit(argv, &cmd)

具体来看下各个阶段的做法，先从最原始的流程开始看起
###### 2.2.1 RootFSInit
RootFSInit方式是Magisk基于最原始的android启动流程来开发的，可以让我们更好的了解最开始在没有复杂的init流程和rootdir的场景下的执行流程
```c++
/************
 * Initramfs
 ************/
// native/jni/init/init.hpp
class RootFSInit : public MagiskInit {
private:
    void early_mount();
    void patch_rootfs();
public:
    RootFSInit(char *argv[], cmdline *cmd) : MagiskInit(argv, cmd) {
        LOGD("%s\n", __FUNCTION__);
    }
    void start() override {
        early_mount();
        patch_rootfs();
        exec_init();
    }
};
```
最原始的启动方式会分为三步：
- early_mount
- patch_rootfs
- exec_init

```c++
// native/jni/init/mount.cpp
// 真正挂载rootfs之前的准备工作
void RootFSInit::early_mount() {
    // 把init文件映射到内存，这个时候init文件在ramdisk patch的时候已经被替换成magiskinit了
    self = mmap_data::ro("/init");

    LOGD("Restoring /init\n");
    // 同时把备份的init替换当前的init，意味着将init换回原生init
    // 这一步是为了让exec_init阶段能够继续执行原生init
    rename("/.backup/init", "/init");
    // 根据devicetree来挂载
    mount_with_dt();
}

void MagiskInit::mount_with_dt() {
    vector<fstab_entry> fstab;
    // 读取fstab文件中的挂载信息
    read_dt_fstab(fstab);
    for (const auto &entry : fstab) {
        if (is_lnk(entry.mnt_point.data()))
            continue;
        // Derive partname from dev
        sprintf(blk_info.partname, "%s%s", basename(entry.dev.data()), cmd->slot);
        // 创建块文件
        setup_block(true);
        // 创建挂载目录
        xmkdir(entry.mnt_point.data(), 0755);
        // 执行只读挂载
        xmount(blk_info.block_dev, entry.mnt_point.data(), entry.type.data(), MS_RDONLY, nullptr);
        // 同时在mount_list中添加一个挂载点记录
        mount_list.push_back(entry.mnt_point);
    }
}

void BaseInit::read_dt_fstab(vector<fstab_entry> &fstab) {
    // dt_dir来自于内核cmdline中
    // dt_dir赋值是在根据key：androidboot.android_dt_dir来的，如果没有
    // 默认为/proc/device-tree/firmware/android
    if (access(cmd->dt_dir, F_OK) != 0)
        return;

    char cwd[128];
    getcwd(cwd, sizeof(cwd));
    // 切换到目录下
    chdir(cmd->dt_dir);
    run_finally cd([&]{ chdir(cwd); });
    // 没有fstab的话退出，在android10+的设备上已经没有/proc/device-tree的目录了
    if (access("fstab", F_OK) != 0)
        return;
    // 切换到fstab目录
    chdir("fstab");

    // Make sure dt fstab is enabled
    // 读取status文件看是否fstab都已经准备就绪
    if (access("status", F_OK) == 0) {
        auto status = rtrim(full_read("status"));
        if (status != "okay" && status != "ok")
            return;
    }

    auto dir = xopen_dir(".");
    // 根据fstab目录下的挂载信息来加载到fstab列表中
    for (dirent *dp; (dp = xreaddir(dir.get()));) {
        if (dp->d_type != DT_DIR)
            continue;
        chdir(dp->d_name);
        run_finally f([]{ chdir(".."); });

        if (access("status", F_OK) == 0) {
            auto status = rtrim(full_read("status"));
            if (status != "okay" && status != "ok")
                continue;
        }

        fstab_entry entry;

        read_info(dev);
        read_info(mnt_point) else {
            entry.mnt_point = "/";
            entry.mnt_point += dp->d_name;
        }
        read_info(type);
        read_info(mnt_flags);
        read_info(fsmgr_flags);

        fstab.emplace_back(std::move(entry));
    }
}
```
此时magiskinit已经加载到内存中了，init文件也替换成原生的，按照fstab都将目录挂载好了
```c++
// native/jni/init/rootdir.cpp
void RootFSInit::patch_rootfs() {
    // Create hardlink mirror of /sbin to /root
    // 创建/root目录
    mkdir("/root", 0777);
    // 复制/sbin的所有属性给/root
    clone_attr("/sbin", "/root");
    // 创建/sbin的硬链接给/root
    link_path("/sbin", "/root");

    // Handle custom sepolicy rules
    // 创建/dev/mnt和/dev/block目录
    xmkdir(TMP_MNTDIR, 0755);
    xmkdir("/dev/block", 0755);
    // 尝试哪个分区的块文件是可以创建的，比如userdatda、cache
    mount_rules_dir("/dev/block", TMP_MNTDIR);
    // Preserve custom rule path
    if (!custom_rules_dir.empty()) {
        string rules_dir = "./" + custom_rules_dir.substr(sizeof(TMP_MNTDIR));
        // rules_dir的路径软链到/.backup/.sepolicy.rules
        xsymlink(rules_dir.data(), TMP_RULESDIR);
    }

    // patch sepolicy，会根据是否是split policy来继续make_pair的操作
    if (patch_sepolicy("/sepolicy")) {
        auto init = mmap_data::rw("/init");
        init.patch({ make_pair(SPLIT_PLAT_CIL, "xxx") });
    }

    // Handle overlays
    // ramdisk patch中已经创建了/overlay.d
    if (access("/overlay.d", F_OK) == 0) {
        LOGD("Merge overlay.d\n");
        // 搜索/overlay.d中的rc文件进行加入rc_list便于后续对rc的patch
        load_overlay_rc("/overlay.d");
        // /overlay.d的文件、属性都移动到根目录并删除/overlay.d
        mv_path("/overlay.d", "/");
    }

    patch_init_rc("/init.rc", "/init.p.rc", "/sbin");
    // 将patch后的init.rc替换原生的init.rc
    rename("/init.p.rc", "/init.rc");

    // Dump magiskinit as magisk
    // early_mount阶段把magiskinit写入内存了，这里写入/sbin/magisk
    int fd = xopen("/sbin/magisk", O_WRONLY | O_CREAT, 0755);
    write(fd, self.buf, self.sz);
    close(fd);
}
```
这一步在rootfs挂载成rootdir之后，需要对rootdir中的文件进行patch，拆分开来看具体的动作
- patch sepolicy
    ```c++
    // native/jni/init/rootdir.cpp
    bool MagiskInit::patch_sepolicy(const char *file) {
        bool patch_init = false;
        sepolicy *sepol = nullptr;
        // 判断是否是split policy
        if (access(SPLIT_PLAT_CIL, R_OK) == 0) {
            LOGD("sepol: split policy\n");
            patch_init = true;
        } else if (access("/sepolicy", R_OK) == 0) {
            // 如果sepolicy存在，则加载
            LOGD("sepol: monolithic policy\n");
            sepol = sepolicy::from_file("/sepolicy");
        } else {
            LOGD("sepol: no selinux\n");
            return false;
        }

        if (access(SELINUX_VERSION, F_OK) != 0) {
            // Mount selinuxfs to communicate with kernel
            xmount("selinuxfs", SELINUX_MNT, "selinuxfs", 0, nullptr);
            mount_list.emplace_back(SELINUX_MNT);
        }

        if (patch_init)
            sepol = sepolicy::from_split();
        // magisk自身依赖的sepolicy的修改
        sepol->magisk_rules();

        // Custom rules
        // 获取自定义sepolicy的规则进行patch
        if (!custom_rules_dir.empty()) {
            if (auto dir = xopen_dir(custom_rules_dir.data())) {
                for (dirent *entry; (entry = xreaddir(dir.get()));) {
                    auto rule = custom_rules_dir + "/" + entry->d_name + "/sepolicy.rule";
                    if (xaccess(rule.data(), R_OK) == 0) {
                        LOGD("Loading custom sepolicy patch: [%s]\n", rule.data());
                        sepol->load_rule_file(rule.data());
                    }
                }
            }
        }

        LOGD("Dumping sepolicy to: [%s]\n", file);
        sepol->to_file(file);
        delete sepol;

        // Remove OnePlus stupid debug sepolicy and use our own
        if (access("/sepolicy_debug", F_OK) == 0) {
            unlink("/sepolicy_debug");
            link("/sepolicy", "/sepolicy_debug");
        }

        return patch_init;
    }

    // native/jni/magiskpolicy/rules.cpp
    // 权限的修改
    void sepolicy::magisk_rules() {
        // Temp suppress warnings
        auto bak = log_cb.w;
        log_cb.w = nop_log;

        // This indicates API 26+
        bool new_rules = exists("untrusted_app_25");

        // Prevent anything to change sepolicy except ourselves
        deny(ALL, "kernel", "security", "load_policy");

        // 定义type为domain
        type(SEPOL_PROC_DOMAIN, "domain");
        // 一系列的权限授予流程
        ......
        permissive(SEPOL_PROC_DOMAIN);  /* Just in case something is missing */
        typeattribute(SEPOL_PROC_DOMAIN, "mlstrustedsubject");
        typeattribute(SEPOL_PROC_DOMAIN, "netdomain");
        typeattribute(SEPOL_PROC_DOMAIN, "bluetoothdomain");
        type(SEPOL_FILE_TYPE, "file_type");
        typeattribute(SEPOL_FILE_TYPE, "mlstrustedobject");
    }
    ```
    patch sepolicy的部分总共做了三个事
    - 挂载selinuxfs，也就是/sys/fs/selinux，它来管理sepolicy
    - 修改sepolicy规则，添加角色
    - 添加自定义修改，这里的自定义修改是对应的module中对于sepolicy的修改

- patch init.rc
    在patch init.rc之前会对overlays进行处理
    ```c++
    // native/jni/init/rootdir.cpp
    static void load_overlay_rc(const char *overlay) {
        auto dir = open_dir(overlay);
        if (!dir) return;

        int dfd = dirfd(dir.get());
        // Do not allow overwrite init.rc
        // 删除overlay.d目录下的init.rc，这样做是为了防止在切换overlay.d到根目录的时候覆盖原生init.rc
        unlinkat(dfd, "init.rc", 0);
        // 搜索/overlay.d中的rc文件进行加入rc_list便于后续对rc的patch
        for (dirent *entry; (entry = xreaddir(dir.get()));) {
            if (str_ends(entry->d_name, ".rc")) {
                LOGD("Found rc script [%s]\n", entry->d_name);
                int rc = xopenat(dfd, entry->d_name, O_RDONLY | O_CLOEXEC);
                rc_list.push_back(fd_full_read(rc));
                close(rc);
                // 删除overlay.d下的其他rc文件
                unlinkat(dfd, entry->d_name, 0);
            }
        }
    }
    // native/jni/init/rootdir.cpp
    static void patch_init_rc(const char *src, const char *dest, const char *tmp_dir) {
        FILE *rc = xfopen(dest, "we");
        // 读取init.rc文件的内容并处理
        file_readline(src, [=](string_view line) -> bool {
            // Do not start vaultkeeper
            // vaultkeeper是一个开源的密码管理器
            if (str_contains(line, "start vaultkeeper")) {
                LOGD("Remove vaultkeeper\n");
                return true;
            }
            // Do not run flash_recovery
            // 无法通过三方工具刷写recovery分区
            if (str_starts(line, "service flash_recovery")) {
                LOGD("Remove flash_recovery\n");
                fprintf(rc, "service flash_recovery /system/bin/xxxxx\n");
                return true;
            }
            // 其他的都先写入/init.p.rc
            // Else just write the line
            fprintf(rc, "%s", line.data());
            return true;
        });

        fprintf(rc, "\n");

        // Inject custom rc scripts
        for (auto &script : rc_list) {
            // Replace template arguments of rc scripts with dynamic paths
            // 使用tmp_dir路径去替换rc文件中的${MAGISKTMP}
            replace_all(script, "${MAGISKTMP}", tmp_dir);
            fprintf(rc, "\n%s\n", script.data());
        }
        rc_list.clear();

        // Inject Magisk rc scripts
        char pfd_svc[16], ls_svc[16], bc_svc[16];
        gen_rand_str(pfd_svc, sizeof(pfd_svc));
        gen_rand_str(ls_svc, sizeof(ls_svc));
        gen_rand_str(bc_svc, sizeof(bc_svc));
        LOGD("Inject magisk services: [%s] [%s] [%s]\n", pfd_svc, ls_svc, bc_svc);
        // 写入自定义的magisk服务，根据native/jni/init/magiskrc.inc
        fprintf(rc, MAGISK_RC, tmp_dir, pfd_svc, ls_svc, bc_svc);
        // 写入/init.p.rc结束
        fclose(rc);
        clone_attr(src, dest);
    }

    // native/jni/init/magiskrc.inc
    constexpr char MAGISK_RC[] =
    "\n"

    "on post-fs-data\n"
    "    start logd\n"
    "    rm " UNBLOCKFILE "\n"
    "    start %2$s\n"
    "    wait " UNBLOCKFILE " " str(POST_FS_DATA_WAIT_TIME) "\n"
    "    rm " UNBLOCKFILE "\n"
    "\n"

    // 全路径的话就是/sbin/magisk
    "service %2$s %1$s/magisk --post-fs-data\n"
    "    user root\n"
    "    seclabel u:r:" SEPOL_PROC_DOMAIN ":s0\n"
    "    oneshot\n"
    "\n"

    "service %3$s %1$s/magisk --service\n"
    "    class late_start\n"
    "    user root\n"
    "    seclabel u:r:" SEPOL_PROC_DOMAIN ":s0\n"
    "    oneshot\n"
    "\n"

    "on property:sys.boot_completed=1\n"
    "    start %4$s\n"
    "\n"

    "service %4$s %1$s/magisk --boot-complete\n"
    "    user root\n"
    "    seclabel u:r:" SEPOL_PROC_DOMAIN ":s0\n"
    "    oneshot\n"
    "\n"
    ;
    ```
    patch init.rc的部分总共做了三个事
    - 禁用了原生init.rc中的某些服务
    - 加载rc_list，也就是overlay.d下的rc文件，和sepolicy同理，对应的是module中的修改
    - 在post-fs-data、service、boot-complete三个阶段启动/sbin/magisk服务，并以SEPOL_PROC_DOMAIN的SELinux context

```c++
// 执行原生的init完成其正常的工作
// native/jni/init/mount.cpp
void BaseInit::exec_init() {
    // Unmount in reverse order
    for (auto &p : reversed(mount_list)) {
        if (xumount(p.data()) == 0)
            LOGD("Unmount [%s]\n", p.data());
    }
    // 继续执行init，记住这里的init已经在early_mount时替换成原生的了
    execv("/init", argv);
    exit(1);
}
```
到了exec_init阶段时首先会对已挂载的目录进行卸载，这么做的原因是因为在后续已经不需要再对这些目录进行修改了，避免资源的无效占用。之后就可以根据原生的init来完成后续的操作了

###### 2.2.2 RecoveryInit
```c++
class RecoveryInit : public BaseInit {
public:
    RecoveryInit(char *argv[], cmdline *cmd) : BaseInit(argv, cmd) {}
    void start() override {
        LOGD("Ramdisk is recovery, abort\n");
        rename("/.backup/init", "/init");
        rm_rf("/.backup");
        exec_init();
    }
};
```
RecoveryInit目的是为了让设备正常引导其到recovery模式，所以直接将.backup中的init文件替换过来即可
###### 2.2.3 SARInit
略
###### 2.2.4 FirstStageInit/SecondStageInit
前文也提到过，Android10+的设备基本都是采用2SI的方式启动，这里就把两个阶段合并在一起来讲
```c++
/***************
 * 2 Stage Init
 ***************/

class FirstStageInit : public BaseInit {
private:
    void prepare();
public:
    FirstStageInit(char *argv[], cmdline *cmd) : BaseInit(argv, cmd) {
        LOGD("%s\n", __FUNCTION__);
    };
    void start() override {
        prepare();
        exec_init();
    }
};

class SecondStageInit : public SARBase {
private:
    void prepare();
public:
    SecondStageInit(char *argv[]) : SARBase(argv, nullptr) {
        LOGD("%s\n", __FUNCTION__);
    };
    void start() override {
        prepare();
        patch_rootdir();
        exec_init();
    }
};
```
可以看出来，两个阶段所做的事情类似，不过由于第二阶段才真正到了system目录下，所以才会执行patch_rootdir的动作
```c++
// native/jni/init/twostage.cpp
void FirstStageInit::prepare() {
    if (cmd->force_normal_boot) {
        // Android10+走这个分支
        // 创建/first_stage_ramdisk/system/bin
        xmkdirs(FSR "/system/bin", 0755);
        // 将magiskinit重命名为/first_stage_ramdisk/system/bin/init
        rename("/init" /* magiskinit */, FSR "/system/bin/init");
        // /system/bin/init软链到/first_stage_ramdisk/init
        symlink("/system/bin/init", FSR "/init");
        // 备份到原生init回到/init
        rename("/.backup/init", "/init");
        // /.backup变成/first_stage_ramdisk/.backup
        rename("/.backup", FSR "/.backup");
        // /overlay.d变成/first_stage_ramdisk/overlay.d
        rename("/overlay.d", FSR "/overlay.d");
        // 切换到/first_stage_ramdisk
        chdir(FSR);
    } else {
        xmkdir("/system", 0755);
        xmkdir("/system/bin", 0755);
        rename("/init" /* magiskinit */ , "/system/bin/init");
        rename("/.backup/init", "/init");
    }

    char fstab_file[128];
    fstab_file[0] = '\0';
    // Find existing fstab file
    // 寻找fstab文件
    for (const char *suffix : { cmd->fstab_suffix, cmd->hardware, cmd->hardware_plat }) {
        if (suffix[0] == '\0')
            continue;
        for (const char *prefix: { "odm/etc/fstab", "vendor/etc/fstab", "fstab" }) {
            // 拼接路径，比如红米note11，它的路径就是vendor/etc/fstab.mt6768
            sprintf(fstab_file, "%s.%s", prefix, suffix);
            if (access(fstab_file, F_OK) != 0) {
                fstab_file[0] = '\0';
            } else {
                LOGD("Found fstab file: %s\n", fstab_file);
                goto exit_loop;
            }
        }
    }
exit_loop:

    // Try to load dt fstab
    vector<fstab_entry> fstab;
    // 和RootFSInit阶段类似，当fstab目录存在时直接在该目录下收集挂载信息
    read_dt_fstab(fstab);
    if (!fstab.empty()) {
        // Dump dt fstab to fstab file in rootfs and force init to use it instead

        // All dt fstab entries should be first_stage_mount

        for (auto &entry : fstab) {
            if (!str_contains(entry.fsmgr_flags, "first_stage_mount")) {
                if (!entry.fsmgr_flags.empty())
                    entry.fsmgr_flags += ',';
                entry.fsmgr_flags += "first_stage_mount";
            }
        }

        if (fstab_file[0] == '\0') {
            const char *suffix =
                    cmd->fstab_suffix[0] ? cmd->fstab_suffix :
                    (cmd->hardware[0] ? cmd->hardware :
                    (cmd->hardware_plat[0] ? cmd->hardware_plat : nullptr));
            if (suffix == nullptr) {
                LOGE("Cannot determine fstab suffix!\n");
                return;
            }
            sprintf(fstab_file, "fstab.%s", suffix);
        }

        // Patch init to force IsDtFstabCompatible() return false
        auto init = mmap_data::rw("/init");
        init.patch({ make_pair("android,fstab", "xxx") });
    } else {
        // Parse and load the fstab file
        // Andorid10+会走这个分支
        file_readline(fstab_file, [&](string_view l) -> bool {
            if (l[0] == '#' || l.length() == 1)
                return true;
            char *line = (char *) l.data();

            int dev0, dev1, mnt_point0, mnt_point1, type0, type1,
                    mnt_flags0, mnt_flags1, fsmgr_flags0, fsmgr_flags1;

            sscanf(line, "%n%*s%n %n%*s%n %n%*s%n %n%*s%n %n%*s%n",
                   &dev0, &dev1, &mnt_point0, &mnt_point1, &type0, &type1,
                   &mnt_flags0, &mnt_flags1, &fsmgr_flags0, &fsmgr_flags1);

            fstab_entry entry;

            set_info(dev);
            set_info(mnt_point);
            set_info(type);
            set_info(mnt_flags);
            set_info(fsmgr_flags);

            fstab.emplace_back(std::move(entry));
            return true;
        });
    }

    {
        LOGD("Write fstab file: %s\n", fstab_file);
        auto fp = xopen_file(fstab_file, "we");
        for (auto &entry : fstab) {
            // Redirect system mnt_point so init won't switch root in first stage init
            if (entry.mnt_point == "/system")
                entry.mnt_point = "/system_root";

            // Force remove AVB for 2SI since it may bootloop some devices
            auto len = patch_verity(entry.fsmgr_flags.data(), entry.fsmgr_flags.length());
            entry.fsmgr_flags.resize(len);

            entry.to_file(fp.get());
        }
    }
    chmod(fstab_file, 0644);

    chdir("/");
}
```
而exec_init都和RootFSInit方式类似，执行原生的init，下面看看第二阶段的prepare
```c++
// native/jni/init/mount.cpp
void SecondStageInit::prepare() {
    backup_files();

    // 卸载/init和/proc/self/exe
    umount2("/init", MNT_DETACH);
    umount2("/proc/self/exe", MNT_DETACH);

    if (access("/system_root", F_OK) == 0)
        switch_root("/system_root");
}

void SARBase::backup_files() {
    // 如果/overlay.d存在的话，加载到overlays当中
    // /overlay.d必然存在，在patch ramdisk时已经添加
    if (access("/overlay.d", F_OK) == 0)
        backup_folder("/overlay.d", overlays);

    self = mmap_data::ro("/proc/self/exe");
    // /.backup/.magisk同理
    if (access("/.backup/.magisk", R_OK) == 0)
        config = mmap_data::ro("/.backup/.magisk");
}

static void switch_root(const string &path) {
    LOGD("Switch root to %s\n", path.data());
    int root = xopen("/", O_RDONLY);
    vector<string> mounts;
    // 读取/proc/mounts的信息进行分区挂载
    parse_mnt("/proc/mounts", [&](mntent *me) {
        // Skip root and self
        if (me->mnt_dir == "/"sv || me->mnt_dir == path)
            return true;
        // Do not include subtrees
        for (const auto &m : mounts) {
            if (strncmp(me->mnt_dir, m.data(), m.length()) == 0 && me->mnt_dir[m.length()] == '/')
                return true;
        }
        mounts.emplace_back(me->mnt_dir);
        return true;
    });
    for (auto &dir : mounts) {
        auto new_path = path + dir;
        mkdir(new_path.data(), 0755);
        xmount(dir.data(), new_path.data(), nullptr, MS_MOVE, nullptr);
    }
    chdir(path.data());
    // 把/system_root挂载为根目录
    xmount(path.data(), "/", nullptr, MS_MOVE, nullptr);
    chroot(".");

    LOGD("Cleaning rootfs\n");
    frm_rf(root);
}
```
第一阶段完成了system的挂载，第二阶段的prepare完成了根目录的切换，patch_rootdir可以划分为几个部分
- setup tmp
    ```c++
    // native/jni/init/rootdir.cpp
    // android11+移除了/sbin，因此magisk选择/dev随机目录作为存放二进制文件的目录
    if (access("/sbin", F_OK) == 0) {
        tmp_dir = "/sbin";
        sepol = "/sbin/.se";
    } else {
        char buf[8];
        gen_rand_str(buf, sizeof(buf));
        tmp_dir = "/dev/"s + buf;
        xmkdir(tmp_dir.data(), 0);
        sepol = "/dev/.se";
    }

    // native/jni/init/mount.cpp
    LOGD("Setup Magisk tmp at %s\n", path);
    // 将/devxxx挂载成tmpfs格式
    xmount("tmpfs", path, "tmpfs", 0, "mode=755");

    chdir(path);
    // 创建.magisk、.magisk/mirror等
    xmkdir(INTLROOT, 0755);
    xmkdir(MIRRDIR, 0);
    xmkdir(BLOCKDIR, 0);
    // 将./backup/.magisk写入/config
    int fd = xopen(INTLROOT "/config", O_WRONLY | O_CREAT, 0);
    xwrite(fd, config.buf, config.sz);
    close(fd);
    // 内存中的内容写入magiskinit
    fd = xopen("magiskinit", O_WRONLY | O_CREAT, 0755);
    xwrite(fd, self.buf, self.sz);
    close(fd);

    // The magisk binary will be handled later
    // 做一些软链
    // Create applet symlinks
    for (int i = 0; applet_names[i]; ++i)
        xsymlink("./magisk", applet_names[i]);
    xsymlink("./magiskinit", "magiskpolicy");
    xsymlink("./magiskinit", "supolicy");

    chdir("/");
    ```
- system_root mount
    ```c++
    // Mount system_root mirror
    ROOTMIR = /devxxx/.magisk/mirror/system_root
    xmkdir(ROOTMIR, 0755);
    // 通过bind mount的方法挂载到根目录
    xmount("/", ROOTMIR, nullptr, MS_BIND, nullptr);
    mount_list.emplace_back(tmp_dir + "/" ROOTMIR);
    ```
- patch init
    ```c++
    // Patch init
    int patch_count;
    {
        int src = xopen("/init", O_RDONLY | O_CLOEXEC);
        auto init = mmap_data::ro("/init");
        patch_count = init.patch({
            // 禁止加载split policy
            make_pair(SPLIT_PLAT_CIL, "xxx"), /* Force loading monolithic sepolicy */
            // 将/sepolicy替换成/dev/.se
            make_pair(MONOPOLICY, sepol)      /* Redirect /sepolicy to custom path */
         });
        // ROOTOVL = /devxxx/.magisk/rootdir
        xmkdir(ROOTOVL, 0);
        int dest = xopen(ROOTOVL "/init", O_CREAT | O_WRONLY | O_CLOEXEC, 0);
        // 将修改后的init替换原生init
        xwrite(dest, init.buf, init.sz);
        fclone_attr(src, dest);
        close(src);
        close(dest);
    }

    if (patch_count != 2) {
        // 当patch_count不为2时，也就说明修改并没有生效，因此还需要将libselinux.so中的/sepolicy路径修改
        // init is dynamically linked, need to patch libselinux
        const char *path = "/system/lib64/libselinux.so";
        if (access(path, F_OK) != 0) {
            path = "/system/lib/libselinux.so";
            if (access(path, F_OK) != 0)
                path = nullptr;
        }
        if (path) {
            char ovl[128];
            sprintf(ovl, ROOTOVL "%s", path);
            auto lib = mmap_data::ro(path);
            lib.patch({make_pair(MONOPOLICY, sepol)});
            xmkdirs(dirname(ovl), 0755);
            int dest = xopen(ovl, O_CREAT | O_WRONLY | O_CLOEXEC, 0);
            xwrite(dest, lib.buf, lib.sz);
            close(dest);
            clone_attr(path, ovl);
        }
    }

    // sepolicy
    patch_sepolicy(sepol);
    ```
- restore backup
    ```c++
    struct sockaddr_un sun;
    int sockfd = xsocket(AF_LOCAL, SOCK_STREAM | SOCK_CLOEXEC, 0);
    if (connect(sockfd, (struct sockaddr*) &sun, setup_sockaddr(&sun, INIT_SOCKET)) == 0) {
        LOGD("ACK init daemon to write backup files\n");
        // Let daemon know where tmp_dir is
        write_string(sockfd, tmp_dir);
        // Wait for daemon to finish restoring files
        read_int(sockfd);
    } else {
        LOGD("Restore backup files locally\n");
        restore_folder(ROOTOVL, overlays);
        overlays.clear();
    }
    close(sockfd);
    ```
- patch init.rc
    ```c++
    // Patch init.rc
    if (access("/init.rc", F_OK) == 0) {
        patch_init_rc("/init.rc", ROOTOVL "/init.rc", tmp_dir.data());
    } else {
        // Android 11's new init.rc
        xmkdirs(dirname(ROOTOVL NEW_INITRC), 0755);
        patch_init_rc(NEW_INITRC, ROOTOVL NEW_INITRC, tmp_dir.data());
    }
    ```
- extract magisk
    ```c++
    {
        auto magisk = mmap_data::ro("magisk32.xz");
        unlink("magisk32.xz");
        int fd = xopen("magisk32", O_WRONLY | O_CREAT, 0755);
        unxz(fd, magisk.buf, magisk.sz);
        close(fd);
        patch_socket_name("magisk32");
        if (access("magisk64.xz", F_OK) == 0) {
            magisk = mmap_data::ro("magisk64.xz");
            unlink("magisk64.xz");
            fd = xopen("magisk64", O_WRONLY | O_CREAT, 0755);
            unxz(fd, magisk.buf, magisk.sz);
            close(fd);
            patch_socket_name("magisk64");
            xsymlink("./magisk64", "magisk");
        } else {
            xsymlink("./magisk32", "magisk");
        }
    }
    ```
- mount rootdir
    ```c++
    ROOTOVL = /devxxx/.magisk/rootdir
    magic_mount(ROOTOVL);

    static void magic_mount(const string &sdir, const string &ddir = "") {
        auto dir = xopen_dir(sdir.data());
        // 利用bind mount机制将修改同步到根目录
        for (dirent *entry; (entry = xreaddir(dir.get()));) {
            string src = sdir + "/" + entry->d_name;
            string dest = ddir + "/" + entry->d_name;
            if (access(dest.data(), F_OK) == 0) {
                if (entry->d_type == DT_DIR) {
                    // Recursive
                    magic_mount(src, dest);
                } else {
                    LOGD("Mount [%s] -> [%s]\n", src.data(), dest.data());
                    xmount(src.data(), dest.data(), nullptr, MS_BIND, nullptr);
                    magic_mount_list += dest;
                    magic_mount_list += '\n';
                }
            }
        }
    }
    ```
到这里就完成了patch rootdir的部分，最后一步也是RootFSInit一样，交由原生init接管

像上面分析到的RootFSInit方式和2SI方式本质上并没有太大区别，唯一的差异点就在于受制于系统启动方式改变而做的妥协，像RootFSInit方式的时候，magiskinit执行时面对的环境是根目录可写挂载的，因此RootFSInit方式可以无限制的针对rootdir进行修改，但是对于2SI方式来说，在二阶段prepare节点时完成根目录切换，而这个时候magiskinit所处的环境是根目录只读挂载，无法直接操作，因此借助于bind mount这种机制，将/devxxx/.magisk/rootdir挂载到根目录上，在原有路径下修改完成后再调用bind mount同步回根目录上，这样本质上并没有修改只读的根目录

到目前为止，基本上可以了解Magisk对boot.img以及booting process所做的事情，这是Magisk实现Root的保障。当然，其中还有些点会比较让人疑惑（特别是init启动这个不分），需要再结合init的源码来巩固

### 五、参考
感谢这些文章带来的启发
1. https://android.stackexchange.com/questions/213167/how-does-magisk-work
2. https://bbs.kanxue.com/thread-275939.htm#msg_header_h2_3