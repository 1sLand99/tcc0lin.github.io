# Zygisk-V27.0源码阅读


隔了很久再读Magisk源码中关于Zygisk的部分，上次翻源码还是v25.0，这次已经更新到了v27.0。粗略扫了眼，变化的地方还是挺多的，想搜索一下关键字也基本上搜索不到，懒得重新过一遍源码，既然是关于zygisk，那就以`(zygisk_enabled)`作为关键搜索词切入

```c&#43;&#43;
void load_modules() {
    ......

    if (zygisk_enabled) {
        string native_bridge_orig = get_prop(NBPROP);
        if (native_bridge_orig.empty()) {
            native_bridge_orig = &#34;0&#34;;
        }
        native_bridge = native_bridge_orig != &#34;0&#34; ? ZYGISKLDR &#43; native_bridge_orig : ZYGISKLDR;
        set_prop(NBPROP, native_bridge.data());
        // Weather Huawei&#39;s Maple compiler is enabled.
        // If so, system server will be created by a special Zygote which ignores the native bridge
        // and make system server out of our control. Avoid it by disabling.
        if (get_prop(&#34;ro.maple.enable&#34;) == &#34;1&#34;) {
            set_prop(&#34;ro.maple.enable&#34;, &#34;0&#34;);
        }
        inject_zygisk_libs(system);
    }
    ......
}
```
定位到load_modules函数，发现这里竟然使用了native_bridge_orig，对应的变量名也做了个效果，NBPROP？
```c&#43;&#43;
#define NBPROP          &#34;ro.dalvik.vm.native.bridge&#34;
```
这是也要仿照riru了吗？

### 一、Zygisk注入方式改变
继续load_modules，ZYGISKLDR对应的是`libzygisk.so`，也就是`src/core/zygisk`目录，从之前的[riru原理理解](https://tcc0lin.github.io/riru%E5%8E%9F%E7%90%86%E7%90%86%E8%A7%A3/)这篇文章中，已经知道对于native_bridge的使用大概流程如下
- 调用LoadNativeBridge函数
- dlopen native_bridge对应的so动态库
- dlsym kNativeBridgeInterfaceSymbol获取callbacks，kNativeBridgeInterfaceSymbol的值是NativeBridgeItf
- 调用isCompatibleWith处理

对应的看下Zygisk相对应的实现
```c&#43;&#43;
// src/core/zygisk/entry.cpp

extern &#34;C&#34; [[maybe_unused]] NativeBridgeCallbacks NativeBridgeItf{
    .version = 2,
    .padding = {},
    .isCompatibleWith = &amp;is_compatible_with,
};

static bool is_compatible_with(uint32_t) {
    zygisk_logging();
    hook_functions();
    ZLOGD(&#34;load success\n&#34;);
    return false;
}
```
这里注意两点：
1. hook_functions根据之前版本的的Zygisk实现来看，应该就是JNI hook的地方
2. 既然是JNI hook，那么表明Zygisk并没有像riru那样存在中转的riruloader.so，相当于直接调用了riru.so

```c&#43;&#43;
// src/core/zygisk/hook.cpp

void hook_functions() {
    default_new(g_hook);
    g_hook-&gt;hook_plt();
}

void HookContext::hook_plt() {
    ino_t android_runtime_inode = 0;
    dev_t android_runtime_dev = 0;
    ino_t native_bridge_inode = 0;
    dev_t native_bridge_dev = 0;

    for (auto &amp;map : lsplt::MapInfo::Scan()) {
        if (map.path.ends_with(&#34;/libandroid_runtime.so&#34;)) {
            android_runtime_inode = map.inode;
            android_runtime_dev = map.dev;
        } else if (map.path.ends_with(&#34;/libnativebridge.so&#34;)) {
            native_bridge_inode = map.inode;
            native_bridge_dev = map.dev;
        }
    }

    PLT_HOOK_REGISTER(native_bridge_dev, native_bridge_inode, dlclose);
    PLT_HOOK_REGISTER(android_runtime_dev, android_runtime_inode, fork);
    PLT_HOOK_REGISTER(android_runtime_dev, android_runtime_inode, unshare);
    PLT_HOOK_REGISTER(android_runtime_dev, android_runtime_inode, androidSetCreateThreadFunc);
    PLT_HOOK_REGISTER(android_runtime_dev, android_runtime_inode, selinux_android_setcontext);
    PLT_HOOK_REGISTER_SYM(android_runtime_dev, android_runtime_inode, &#34;__android_log_close&#34;, android_log_close);

    if (!lsplt::CommitHook())
        ZLOGE(&#34;plt_hook failed\n&#34;);

    // Remove unhooked methods
    plt_backup.erase(
            std::remove_if(plt_backup.begin(), plt_backup.end(),
            [](auto &amp;t) { return *std::get&lt;3&gt;(t) == nullptr;}),
            g_hook-&gt;plt_backup.end());
}
```
替换了XHOOK框架使用了自己实现的PLT_HOOK，对应的这个项目[LSPlt](https://github.com/LSPosed/LSPlt)
1. fork机制和之前是相同的，提前fork
2. unshare对于新的namespace划分时unmount掉其中的Magisk特征
3. androidSetCreateThreadFunc
    ```c&#43;&#43;
    DCL_HOOK_FUNC(static void, androidSetCreateThreadFunc, void *func) {
        ZLOGD(&#34;androidSetCreateThreadFunc\n&#34;);
        g_hook-&gt;hook_jni_env();
        old_androidSetCreateThreadFunc(func);
    }

    void HookContext::hook_jni_env() {
        using method_sig = jint(*)(JavaVM **, jsize, jsize *);
        auto get_created_vms = reinterpret_cast&lt;method_sig&gt;(
                dlsym(RTLD_DEFAULT, &#34;JNI_GetCreatedJavaVMs&#34;));
        if (!get_created_vms) {
            for (auto &amp;map: lsplt::MapInfo::Scan()) {
                if (!map.path.ends_with(&#34;/libnativehelper.so&#34;)) continue;
                void *h = dlopen(map.path.data(), RTLD_LAZY);
                if (!h) {
                    ZLOGW(&#34;Cannot dlopen libnativehelper.so: %s\n&#34;, dlerror());
                    break;
                }
                get_created_vms = reinterpret_cast&lt;method_sig&gt;(dlsym(h, &#34;JNI_GetCreatedJavaVMs&#34;));
                dlclose(h);
                break;
            }
            if (!get_created_vms) {
                ZLOGW(&#34;JNI_GetCreatedJavaVMs not found\n&#34;);
                return;
            }
        }

        JavaVM *vm = nullptr;
        jsize num = 0;
        jint res = get_created_vms(&amp;vm, 1, &amp;num);
        if (res != JNI_OK || vm == nullptr) {
            ZLOGW(&#34;JavaVM not found\n&#34;);
            return;
        }
        JNIEnv *env = nullptr;
        res = vm-&gt;GetEnv(reinterpret_cast&lt;void **&gt;(&amp;env), JNI_VERSION_1_6);
        if (res != JNI_OK || env == nullptr) {
            ZLOGW(&#34;JNIEnv not found\n&#34;);
            return;
        }

        // Replace the function table in JNIEnv to hook RegisterNatives
        memcpy(&amp;new_env, env-&gt;functions, sizeof(*env-&gt;functions));
        new_env.RegisterNatives = &amp;env_RegisterNatives;
        old_env = env-&gt;functions;
        env-&gt;functions = &amp;new_env;
    }

    static jint env_RegisterNatives(
            JNIEnv *env, jclass clazz, const JNINativeMethod *methods, jint numMethods) {
        auto className = get_class_name(env, clazz);
        if (className == &#34;com/android/internal/os/Zygote&#34;) {
            // Restore JNIEnv as we no longer need to replace anything
            env-&gt;functions = g_hook-&gt;old_env;

            vector&lt;JNINativeMethod&gt; newMethods(methods, methods &#43; numMethods);
            vector&lt;JNINativeMethod&gt; &amp;backup = g_hook-&gt;jni_backup[className];
            HOOK_JNI(nativeForkAndSpecialize);
            HOOK_JNI(nativeSpecializeAppProcess);
            HOOK_JNI(nativeForkSystemServer);
            return g_hook-&gt;old_env-&gt;RegisterNatives(env, clazz, newMethods.data(), numMethods);
        } else {
            return g_hook-&gt;old_env-&gt;RegisterNatives(env, clazz, methods, numMethods);
        }
    }
    ```
    这里就不多说了，还是熟系的配方


### 二、LD_PRELOAD其他的用处
Magisk团队找了更直接的方法来修改sepolicy，像他们在details.md所说的
```md
## Pre-Init

`magiskinit` will replace `init` as the first program to run.

- Early mount required partitions. On legacy system-as-root devices, we switch root to system; on 2SI devices, we patch the original `init` to redirect the 2nd stage init file to magiskinit and execute it to mount partitions for us.
- Inject magisk services into `init.rc`
- On devices using monolithic policy, load sepolicy from `/sepolicy`; otherwise we hijack nodes in selinuxfs with FIFO, set `LD_PRELOAD` to hook `security_load_policy` and assist hijacking on 2SI devices, and start a daemon to wait until init tries to load sepolicy.
- Patch sepolicy rules. If we are using &#34;hijack&#34; method, load patched sepolicy into kernel, unblock init and exit daemon
- Execute the original `init` to continue the boot process

```
使用LD_PRELOAD hook security_load_policy
```c&#43;&#43;
// src/init/selinux.cpp

if (access(&#34;/system/bin/init&#34;, F_OK) == 0) {
    // On 2SI devices, the 2nd stage init file is always a dynamic executable.
    // This meant that instead of going through convoluted methods trying to alter
    // and block init&#39;s control flow, we can just LD_PRELOAD and replace the
    // security_load_policy function with our own implementation.
    dump_preload();
    setenv(&#34;LD_PRELOAD&#34;, &#34;/dev/preload.so&#34;, 1);
}
```
preload.so对应的是preload.c
```c&#43;&#43;
// src/init/preload.c

static void preload_init() {
    // Make sure our next exec won&#39;t get bugged
    unsetenv(&#34;LD_PRELOAD&#34;);
    unlink(&#34;/dev/preload.so&#34;);
}

int security_load_policy(void *data, size_t len) {
    int (*load_policy)(void *, size_t) = dlsym(RTLD_NEXT, &#34;security_load_policy&#34;);
    // Skip checking errors, because if we cannot find the symbol, there
    // isn&#39;t much we can do other than crashing anyways.
    int result = load_policy(data, len);

    // Wait for ack
    int fd = open(&#34;/sys/fs/selinux/enforce&#34;, O_RDONLY);
    char c;
    read(fd, &amp;c, 1);
    close(fd);

    return result;
}
```
在使用monolithic策略的设备上，Magisk直接从/sepolicy文件中加载sepolicy规则。这个文件通常位于系统的根目录下，用于存储selinux策略。这种方式比较简单直接，不需要进行额外的hook操作

但是在其他的设备上，Magisk使用FIFO（命名管道）劫持selinuxfs中的节点，以实现selinux hook。具体来说，Magisk会创建一个FIFO文件，并挂载到selinuxfs中的&#34;load&#34;和&#34;enforce&#34;节点上，用于接收selinux策略和enforce值。这样一来，即使系统中没有/sepolicy文件，Magisk也可以通过劫持selinuxfs中的节点，来实现selinux hook

在2SI设备上，由于第二阶段的init文件是一个动态可执行文件，而不是静态的/init可执行文件，因此Magisk还需要协助劫持selinuxfs。具体来说，Magisk会在init进程启动之前，通过LD_PRELOAD的方式，将自己的preload.so库注入到init进程中，并替换security_load_policy函数为自己的实现，以实现selinux hook。后面Magisk启动守护程序，等待init进程尝试加载selinux策略文件。当init进程启动时，Magisk的钩子函数会拦截security_load_policy的调用，并将selinux策略文件和enforce值写入FIFO文件中，以实现自定义的selinux策略

---

> 作者: tcc0lin  
> URL: http://localhost:1313/posts/zygisk-v27.0%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/  

