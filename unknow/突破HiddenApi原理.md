## Android9+

### 原因

反射的时候增加了方法签名校验机制，如果该方法签名不在 割免列表 中，都会被拒绝访问。（HiddenApi都不在割免列表中）

### 解决方法

将方法签名加入到 割免列表 就能突破限制了。（class的签名都是以L开头的，将L加入到豁免列表，就能所有起飞了。）

**元反射：**

通过反射 getDeclaredMethod 方法，getDeclaredMethod 是 public 的，不存在问题。
反射调用 getDeclardMethod 是以系统身份去反射的系统类，因此元反射方法也被系统类加载的方法；所以我们的元反射方法调用的 getDeclardMethod 会被认为是系统调用的，可以反射任意的方法



### 源码分析

```c
//调用链：
java_lang_Class.cc#Class_getDeclaredMethodInternal 
	-> ShouldBlockAccessToMember
	-> hidden_api.h#GetMemberAction
	-> hidden_api.cc#GetMemberActionImpl

//分析：
Class_getDeclaredMethodInternal：//主要 ShouldBlockAccessToMember 函数返回false，就万事大吉了
	if (result == nullptr || ShouldBlockAccessToMember(result->GetArtMethod(), soa.Self())) {
    	return nullptr;
  	}
  	return soa.AddLocalReference<jobject>(result.Get());


ShouldBlockAccessToMember: //action别是kDeny就行了
	hiddenapi::Action action = hiddenapi::GetMemberAction(member, self, IsCallerTrusted, hiddenapi::kReflection);
  	return action == hiddenapi::kDeny;


GetMemberActionImpl:
	Runtime* runtime = Runtime::Current();
	//GetHiddenApiExemptions：获取豁免的Hidden Api签名。（art/runtime/runtime.h -> dalvik_system_VMRuntime.cc ，VMRuntime_setHiddenApiExemptions 该jni方法，其native函数的声明在VMRuntime.java中）
	//IsExempted：判断该方法签名是否在 豁免列表中。
	if (member_signature.IsExempted(runtime->GetHiddenApiExemptions())) {
	      action = kAllow;
	      MaybeWhitelistMember(runtime, member);
	      return kAllow;
    }
```



## android11+ 

### 原因

元反射失效了，因为增加了 调用者上下文判断 机制。机制给 调用者 跟 被调用的方法 增加了domian，调用者domain要等于或者小于 被调用的方法domian，才能允许访问。

### 解决方法

1、寻找domain小的调用栈，在那里进行 割免列表 的添加。

System.loadLibrary（java.lang.System 类在 /apex/com.android.runtime/javalib/core-oj.jar 里边），所以System.loadLibrary的调用栈的domain很低的。而在System.loadLibrary中最终会调到JNI_OnLoad函数，所以可以在JNI_OnLoad函数里边进行割免操作。

参考：https://bbs.pediy.com/thread-268936.htm



2、让系统无法识别调用栈，系统就会给一个最小的domain我们。

在native层创建一个新线程，然后获取该子线程的JniEnv，通过该JniEnv来进行割免操作。

参考：https://www.jianshu.com/p/6546ce67c8e0



3、利用BootClassLoader -- FreeReflection库（https://github.com/tiann/FreeReflection）

DexFile设置父ClassLoader为null（BootClassLoader，它用于加载一些Android系统框架的类，如果用BootClassLoader它来加载domain肯定很低），然后再进行割免操作。

(个人觉得，直接用还是可以的。对其进行改造确实麻烦，还得生成dex然后再转base64啥的。)



### 源码分析

```c
//调用链：
java_lang_Class.cc#Class_getDeclaredMethodInternal 
	-> ShouldDenyAccessToMember 
	-> hidden_api.h#ShouldDenyAccessToMember
	-> hidden_api.cc#ShouldDenyAccessToMemberImpl

//分析：
Class_getDeclaredMethodInternal：//主要 ShouldDenyAccessToMember 函数返回false，就万事大吉了
	if (result == nullptr || ShouldDenyAccessToMember(result->GetArtMethod(), soa.Self())) {
    	return nullptr;
  	}
  	return soa.AddLocalReference<jobject>(result.Get());



enum class Domain : char { //Domain范围
	kCorePlatform = 0, // 0
	kPlatform,  	   // 1 
	kApplication, 	   // 2
};


ShouldDenyAccessToMember：
	//caller上下文：调用者的上下文
	//fn_get_access_context() -> GetHiddenapiAccessContextFunction -> GetReflectionCaller
	const AccessContext caller_context = fn_get_access_context(); 
	//callee上下文：被调用的这个方法的上下文
	const AccessContext callee_context(member->GetDeclaringClass());

	//调用者是否允许调用该方法，返回true就证明允许调用。也就是说Domain越小越牛逼
	//CanAlwaysAccess函数：return callerDomain <= calleeDomain
	if (caller_context.CanAlwaysAccess(callee_context)) { 
	    return false; //返回false，Class_getDeclaredMethodInternal才能继续执行。
	}


GetReflectionCaller：
	//...
	ArtMethod *m = GetMethod();
	ObjPtr<mirror::Class> caller = m->GetDeclaringClass();
	//...

	//caller为空：domain直接设置为 Domain::kCorePlatform
	//caller不为空：获取DexFile（GetDexCache()），然后获取DexFile的domain（GetHiddenapiDomain()）
	return caller.IsNull() ? hiddenapi::AccessContext(true)
                         : hiddenapi::AccessContext(caller);

DexFile domain赋值逻辑：
	hidden_api.cc#InitializeDexFileDomain：
		Domain dex_domain = DetermineDomainFromLocation(dex_file.GetLocation(), class_loader);
		dex_file.SetHiddenapiDomain(dex_domain); //赋值domian

	// cat /proc/pid/maps |grep "/apex/.*.jar" 可查看进程运行时有哪些/apex/的jar
	hidden_api.cc#DetermineDomainFromLocation：
		if (ArtModuleRootDistinctFromAndroidRoot()) { //1.判断/system、/apex/com.android.art 这些目录是否存在
		    if (LocationIsOnArtModule(dex_location.c_str()) //2.是否在 /apex/com.android.art
		    	|| LocationIsOnConscryptModule(dex_location.c_str())) { //3.否是在 /apex/com.android.conscrypt
		      return Domain::kCorePlatform;
		    }
		    if (LocationIsOnApex(dex_location.c_str())) { //4.是否在 /apex/
		      return Domain::kPlatform;
		    }
	  	}
	    if (LocationIsOnSystemFramework(dex_location.c_str())) { //5.是否在 /system/framework
	      	return Domain::kPlatform;
	  	}
	    if (class_loader.IsNull()) {
	    	return Domain::kPlatform;
	  	}
	  	return Domain::kApplication;
```



## 实现代码

```c
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>
#include "android_log.h"

typedef union {
    JNIEnv* env;
    void* v_env;
} EnvToVoid;

/**
 * 通过VMRuntime类的方法 进行割免操作
 * 带@libcore.api.CorePlatformApi注解了，domain应该是被修饰成 corePlatform ？？？？
 */
bool api_exemptions_from_VMRuntime(JNIEnv* env) {
    //android11之后 VMRuntime 带@libcore.api.CorePlatformApi注解了，domain应该是被修饰成 corePlatform 了
    jclass vmRuntime_Class = env->FindClass("dalvik/system/VMRuntime");
    if (vmRuntime_Class == nullptr) {
        ALOGE("VMRuntime class not found.");
        env->ExceptionClear();
        return false;
    }

//    jfieldID THE_ONE_field = env->GetStaticFieldID(vmRuntime_Class,
//                                                           "THE_ONE",
//                                                           "Ldalvik/system/VMRuntime;");
//    if (THE_ONE_field == nullptr) {
//        ALOGE("THE_ONE field not found.");
//        return false;
//    }
//    jobject vmRuntime_Object = env->GetStaticObjectField(vmRuntime_Class, THE_ONE_field);
//    if (vmRuntime_Object == nullptr) {
//        ALOGE("VMRuntime object not found.");
//        return false;
//    }

    jmethodID getRuntime_Method = env->GetStaticMethodID(vmRuntime_Class,
                                                   "getRuntime",
                                                   "()Ldalvik/system/VMRuntime;");
    if (getRuntime_Method == nullptr) {
        ALOGE("getRuntime method not found.");
        return false;
    }
    jobject vmRuntime_Object = env->CallStaticObjectMethod(vmRuntime_Class, getRuntime_Method);
    if (vmRuntime_Object == nullptr) {
        ALOGE("VMRuntime object not found.");
        return false;
    }

    jmethodID setHiddenApiExemptions_Method = env->GetMethodID(vmRuntime_Class,
                                                        "setHiddenApiExemptions",
                                                        "([Ljava/lang/String;)V");
    if (setHiddenApiExemptions_Method == nullptr) {
        ALOGE("setHiddenApiExemptions method not found.");
        return false;
    }

    //设置豁免的Hidden Api签名。class的签名都是以L开头的，所以这里全部进行豁免
    jstring freeTargetClass = env->NewStringUTF("L");
    jobjectArray freeTargetArray = env->NewObjectArray(1,
                                                       env->FindClass("java/lang/String"),
                                                       NULL);
    env->SetObjectArrayElement(freeTargetArray, 0, freeTargetClass);

    env->CallVoidMethod(vmRuntime_Object, setHiddenApiExemptions_Method, freeTargetArray);

    env->DeleteLocalRef(freeTargetClass);
    env->DeleteLocalRef(freeTargetArray);
    return true;
}

/**
 * 通过ZygoteInit类的方法 进行割免操作
 */
bool api_exemptions_from_ZygoteInit(JNIEnv* env) {
    //这里使用ZygoteInit里的方法，内部也是调用 VMRuntime#setHiddenApiExemptions
    jclass zygoteInitClass = env->FindClass("com/android/internal/os/ZygoteInit");
    if (zygoteInitClass == nullptr) {
        ALOGE("ZygoteInit class not found.");
        env->ExceptionClear();
        return false;
    }

    //ZygoteInit#setApiBlacklistExemptions（android9-11）
    jmethodID apiExemptionsMethod = env->GetStaticMethodID(zygoteInitClass,
                                                           "setApiBlacklistExemptions",
                                                           "([Ljava/lang/String;)V");
    if (apiExemptionsMethod == nullptr) {
        ALOGE("setApiBlacklistExemptions method not found.");

        //ZygoteInit#setApiDenylistExemptions（android12）
        apiExemptionsMethod = env->GetStaticMethodID(zygoteInitClass,
                                                     "setApiDenylistExemptions",
                                                     "([Ljava/lang/String;)V");
    }

    if (apiExemptionsMethod == nullptr) {
        ALOGE("setApiDenylistExemptions method not found.");
        return false;
    }

    //设置豁免的Hidden Api签名。class的签名都是以L开头的，所以这里全部进行豁免
    jstring freeTargetClass = env->NewStringUTF("L");
    jobjectArray freeTargetArray = env->NewObjectArray(1,
                                                       env->FindClass("java/lang/String"),
                                                       NULL);
    env->SetObjectArrayElement(freeTargetArray, 0, freeTargetClass);

    env->CallStaticVoidMethod(zygoteInitClass, apiExemptionsMethod, freeTargetArray);

    env->DeleteLocalRef(freeTargetClass);
    env->DeleteLocalRef(freeTargetArray);
    return true;
}

bool perform_api_exemptions(JNIEnv* env){
    if (!api_exemptions_from_ZygoteInit(env)){
        return api_exemptions_from_VMRuntime(env);
    }
    return true;
}

/**
 * 在子线程中执行割免
 * @param arg - java_vm
 * @return 是否割免成功
 */
void *thread_unseal_fun(void *arg) {
    JavaVM *java_vm = static_cast<JavaVM *>(arg);
    if (java_vm == nullptr){
        ALOGE("thread_unseal_fun fail, java_vm is NULL.");
        return reinterpret_cast<void *>(false);
    }

    bool isSuccess;

    JNIEnv *env;
    java_vm->AttachCurrentThread(&env, NULL);

    isSuccess = perform_api_exemptions(env);

    java_vm->DetachCurrentThread();

    return reinterpret_cast<void *>(isSuccess);
}

/**
 * 通过切线程进行割免
 */
bool perform_unseal_from_Thread(JNIEnv *env){
    JavaVM *vm;
    env->GetJavaVM(&vm);

    if (vm == nullptr){
        ALOGE("GetJavaVM fail.");
        return false;
    }

    pthread_t tid;
    pthread_create(&tid, NULL, thread_unseal_fun, vm);

    bool isSuccess;
    pthread_join(tid, reinterpret_cast<void **>(&isSuccess));
    ALOGI("perform_unseal_from_Thread isSuccess=%d.", isSuccess);

    return isSuccess;
}

/**
 * 通过JNI_OnLoad进行割免
 */
bool perform_unseal_from_JNI_OnLoad(JavaVM *vm){
    EnvToVoid envToVoid;
    envToVoid.v_env = NULL;
    if (vm->GetEnv(&envToVoid.v_env, JNI_VERSION_1_6) != JNI_OK) {
        ALOGE("GetEnv fail.");
        return false;
    }

    bool isSuccess = perform_api_exemptions(envToVoid.env);
    ALOGI("perform_unseal_from_JNI_OnLoad isSuccess=%d.", isSuccess);

    return isSuccess;
}
```
