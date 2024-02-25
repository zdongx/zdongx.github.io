Android Art虚拟机创建过程源码分析

*Art虚拟机是Android应用程序运行的基础，重要性是不言而喻的。写这系列的博客也是为了加深对虚拟机的理解。*

考虑到Android的应用程序都是由Zygote进程fork而来，从zygote进程启动的地方入手。

```c++
//frameworks/base/cmds/app_process/app_main.cpp
int main(int argc, char* const argv[])
{
		//... 
    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
  
  	//...
    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
    }
}
```

main方法中线创建了一个AppRuntime对象 runtime，然后调用了runtime的start方法。其中AppRuntime继承自AndroidRuntime，start方法也是在AndroidRuntime中定义。

```c++
//frameworks/base/core/jni/AndroidRuntime.cpp
/*
 * Start the Android runtime.  This involves starting the virtual machine
 * and calling the "static void main(String[] args)" method in the class
 * named by "className".
 *
 * Passes the main function two arguments, the class name and the specified
 * options string.
 */
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
  	//... ...
    /* start the virtual machine */
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;
    if (startVm(&mJavaVM, &env, zygote, primary_zygote) != 0) { //启动虚拟机
        return;
    }
    onVmCreated(env);

     /*
     * Register android functions.
     */
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }  

    free(slashClassName);

    ALOGD("Shutting down VM\n");
    if (mJavaVM->DetachCurrentThread() != JNI_OK)
        ALOGW("Warning: unable to detach main thread\n");
    if (mJavaVM->DestroyJavaVM() != 0)
        ALOGW("Warning: VM did not shut down cleanly\n");
}
```

从这段代码的注释中可以看到，这个方法的作用是启动Android 的runtime。结束后又会调用DetachCurrentThread来释放当前线程。

```c++
//frameworks/base/core/jni/AndroidRuntime.cpp
int AndroidRuntime::startVm(JavaVM** pJavaVM, JNIEnv** pEnv, bool zygote, bool primary_zygote)
{
  
  	// parseXXX 解析虚拟机参数
  
      /*
     * Initialize the VM.
     *
     * The JavaVM* is essentially per-process, and the JNIEnv* is per-thread.
     * If this call succeeds, the VM is ready, and we can start issuing
     * JNI calls.
     */
    if (JNI_CreateJavaVM(pJavaVM, pEnv, &initArgs) < 0) {
        ALOGE("JNI_CreateJavaVM failed\n");
        return -1;
    }
    return 0;
}
```

startVM方法前半部分调用parseXXX方法来解析虚拟机参数，最后通过JNI_CreateJavaVM来初始化虚拟机，从注释中也可以看到，JavaVm是每个进程的，而JNIEnv 是每个线程的。该方法调用成功后，就可以开始JNI调用了。

```c++
//art/runtime/jni/java_vm_ext.cc
// JNI Invocation interface.

extern "C" jint JNI_CreateJavaVM(JavaVM** p_vm, JNIEnv** p_env, void* vm_args) {
	//... ...
  
  bool ignore_unrecognized = args->ignoreUnrecognized;
  if (!Runtime::Create(options, ignore_unrecognized)) { //step1: 创建Runtime
    return JNI_ERR;
  }
  
   // everything set up before we start using JNI.
  android::InitializeNativeLoader(); 

  Runtime* runtime = Runtime::Current();
  bool started = runtime->Start(); //step2：启动runtime
  if (!started) {
    delete Thread::Current()->GetJniEnv();
    delete runtime->GetJavaVM();
    LOG(WARNING) << "CreateJavaVM failed";
    return JNI_ERR;
  }

  *p_env = Thread::Current()->GetJniEnv();
  *p_vm = runtime->GetJavaVM();
  return JNI_OK;
}
```

