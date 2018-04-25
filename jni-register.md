---
description: 'check the JNI register process, for met a register error recently.'
---

# JNI Register

## implementation in JNI

make configHciSnoopLogNative as a sample. 

{% code-tabs %}
{% code-tabs-item title="packages/apps/Bluetooth/jni/com\_android\_bluetooth\_btservice\_AdapterService.cpp" %}
```cpp
static jboolean configHciSnoopLogNative(JNIEnv* env, jobject obj, jboolean enable) {
    ALOGV("%s:",__FUNCTION__);

    jboolean result = JNI_FALSE;

    if (!sBluetoothInterface) return result;

    int ret = sBluetoothInterface->config_hci_snoop_log(enable);

    result = (ret == BT_STATUS_SUCCESS) ? JNI_TRUE : JNI_FALSE;

    return result;
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

## sMethods in JNI

{% code-tabs %}
{% code-tabs-item title="packages/apps/Bluetooth/jni/com\_android\_bluetooth\_btservice\_AdapterService.cpp" %}
```cpp
static JNINativeMethod sMethods[] = {
    /* name, signature, funcPtr */
    {"classInitNative", "()V", (void*)classInitNative},
    {"initNative", "()Z", (void*)initNative},
    {"cleanupNative", "()V", (void*)cleanupNative},
    {"enableNative", "(Z)Z", (void*)enableNative},
    {"disableNative", "()Z", (void*)disableNative},
    {"setAdapterPropertyNative", "(I[B)Z", (void*)setAdapterPropertyNative},
    {"getAdapterPropertiesNative", "()Z", (void*)getAdapterPropertiesNative},
    {"getAdapterPropertyNative", "(I)Z", (void*)getAdapterPropertyNative},
    {"getDevicePropertyNative", "([BI)Z", (void*)getDevicePropertyNative},
    {"setDevicePropertyNative", "([BI[B)Z", (void*)setDevicePropertyNative},
    {"startDiscoveryNative", "()Z", (void*)startDiscoveryNative},
    {"cancelDiscoveryNative", "()Z", (void*)cancelDiscoveryNative},
    {"createBondNative", "([BI)Z", (void*)createBondNative},
    {"createBondOutOfBandNative", "([BILandroid/bluetooth/OobData;)Z",
     (void*)createBondOutOfBandNative},
    {"removeBondNative", "([B)Z", (void*)removeBondNative},
    {"cancelBondNative", "([B)Z", (void*)cancelBondNative},
    {"getConnectionStateNative", "([B)I", (void*)getConnectionStateNative},
    {"pinReplyNative", "([BZI[B)Z", (void*)pinReplyNative},
    {"sspReplyNative", "([BIZI)Z", (void*)sspReplyNative},
    {"getRemoteServicesNative", "([B)Z", (void*)getRemoteServicesNative},
    {"connectSocketNative", "([BI[BIII)I", (void*)connectSocketNative},
    {"createSocketChannelNative", "(ILjava/lang/String;[BIII)I",
     (void*)createSocketChannelNative},
    {"configHciSnoopLogNative", "(Z)Z", (void*) configHciSnoopLogNative},
```
{% endcode-tabs-item %}
{% endcode-tabs %}

## register methods

{% code-tabs %}
{% code-tabs-item title="packages/apps/Bluetooth/jni/com\_android\_bluetooth\_btservice\_AdapterService.cpp" %}
```cpp
int register_com_android_bluetooth_btservice_AdapterService(JNIEnv* env) {
  return jniRegisterNativeMethods(
      env, "com/android/bluetooth/btservice/AdapterService", sMethods,
      NELEM(sMethods));
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

## call procedure

register\_com\_android\_bluetooth\_btservice\_AdapterService while load JNI

{% code-tabs %}
{% code-tabs-item title="packages/apps/Bluetooth/jni/com\_android\_bluetooth\_btservice\_AdapterService.cpp" %}
```cpp
jint JNI_OnLoad(JavaVM *jvm, void *reserved)
{
    JNIEnv *e;
    int status;

    ALOGV("Bluetooth Adapter Service : loading JNI\n");

    // Check JNI version
    if (jvm->GetEnv((void **)&e, JNI_VERSION_1_6)) {
        ALOGE("JNI version mismatch error");
        return JNI_ERR;
    }

    if ((status = android::register_com_android_bluetooth_btservice_AdapterService(e)) < 0) {
        ALOGE("jni adapter service registration failure, status: %d", status);
        return JNI_ERR;
    }

```
{% endcode-tabs-item %}
{% endcode-tabs %}

and goes up find JNI\_OnLoad in AdapterService, which will init during enable Bluetooth.

{% code-tabs %}
{% code-tabs-item title="packages/apps/Bluetooth/src/com/android/bluetooth/btservice/AdapterService.java" %}
```java
    static {
        System.loadLibrary("bluetooth_jni");
        classInitNative();
    }
```
{% endcode-tabs-item %}
{% endcode-tabs %}

## JNI Register Procedure

register\_com\_android\_bluetooth\_btservice\_AdapterService calling process.

### jniRegisterNativeMethods

{% code-tabs %}
{% code-tabs-item title="libnativehelper/JNIHelp.cpp" %}
```cpp
extern "C" int jniRegisterNativeMethods(C_JNIEnv* env, const char* className,
    const JNINativeMethod* gMethods, int numMethods)
{
    JNIEnv* e = reinterpret_cast<JNIEnv*>(env);

    ALOGV("Registering %s's %d native methods...", className, numMethods);

    scoped_local_ref<jclass> c(env, findClass(env, className));
    if (c.get() == NULL) {
        char* tmp;
        const char* msg;
        if (asprintf(&tmp,
                     "Native registration unable to find class '%s'; aborting...",
                     className) == -1) {
            // Allocation failed, print default warning.
            msg = "Native registration unable to find class; aborting...";
        } else {
            msg = tmp;
        }
        e->FatalError(msg);
    }

    if ((*env)->RegisterNatives(e, c.get(), gMethods, numMethods) < 0) {
        char* tmp;
        const char* msg;
        if (asprintf(&tmp, "RegisterNatives failed for '%s'; aborting...", className) == -1) {
            // Allocation failed, print default warning.
            msg = "RegisterNatives failed; aborting...";
        } else {
            msg = tmp;
        }
        e->FatalError(msg);
    }

    return 0;
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### RegisterNatives

java\_class is AdapterService, methods for sMethods. Both methods and native methods in AdapterService will be checked.

{% code-tabs %}
{% code-tabs-item title="art/runtime/jni\_internal.cc" %}
```cpp
  static jint RegisterNatives(JNIEnv* env,
                              jclass java_class,
                              const JNINativeMethod* methods,
                              jint method_count) {
    if (UNLIKELY(method_count < 0)) {
      JavaVmExtFromEnv(env)->JniAbortF("RegisterNatives", "negative method count: %d",
                                       method_count);
      return JNI_ERR;  // Not reached except in unit tests.
    }
    CHECK_NON_NULL_ARGUMENT_FN_NAME("RegisterNatives", java_class, JNI_ERR);
    ScopedObjectAccess soa(env);
    StackHandleScope<1> hs(soa.Self());
    Handle<mirror::Class> c = hs.NewHandle(soa.Decode<mirror::Class>(java_class));
    if (UNLIKELY(method_count == 0)) {
      LOG(WARNING) << "JNI RegisterNativeMethods: attempt to register 0 native methods for "
          << c->PrettyDescriptor();
      return JNI_OK;
    }
    CHECK_NON_NULL_ARGUMENT_FN_NAME("RegisterNatives", methods, JNI_ERR);
    for (jint i = 0; i < method_count; ++i) {
      const char* name = methods[i].name;
      const char* sig = methods[i].signature;
      const void* fnPtr = methods[i].fnPtr;
      if (UNLIKELY(name == nullptr)) {
        ReportInvalidJNINativeMethod(soa, c.Get(), "method name", i);
        return JNI_ERR;
      } else if (UNLIKELY(sig == nullptr)) {
        ReportInvalidJNINativeMethod(soa, c.Get(), "method signature", i);
        return JNI_ERR;
      } else if (UNLIKELY(fnPtr == nullptr)) {
        ReportInvalidJNINativeMethod(soa, c.Get(), "native function", i);
        return JNI_ERR;
      }
      bool is_fast = false;
      // Notes about fast JNI calls:
      //
      // On a normal JNI call, the calling thread usually transitions
      // from the kRunnable state to the kNative state. But if the
      // called native function needs to access any Java object, it
      // will have to transition back to the kRunnable state.
      //
      // There is a cost to this double transition. For a JNI call
      // that should be quick, this cost may dominate the call cost.
      //
      // On a fast JNI call, the calling thread avoids this double
      // transition by not transitioning from kRunnable to kNative and
      // stays in the kRunnable state.
      //
      // There are risks to using a fast JNI call because it can delay
      // a response to a thread suspension request which is typically
      // used for a GC root scanning, etc. If a fast JNI call takes a
      // long time, it could cause longer thread suspension latency
      // and GC pauses.
      //
      // Thus, fast JNI should be used with care. It should be used
      // for a JNI call that takes a short amount of time (eg. no
      // long-running loop) and does not block (eg. no locks, I/O,
      // etc.)
      //
      // A '!' prefix in the signature in the JNINativeMethod
      // indicates that it's a fast JNI call and the runtime omits the
      // thread state transition from kRunnable to kNative at the
      // entry.
      if (*sig == '!') {
        is_fast = true;
        ++sig;
      }

      // Note: the right order is to try to find the method locally
      // first, either as a direct or a virtual method. Then move to
      // the parent.
      ArtMethod* m = nullptr;
      bool warn_on_going_to_parent = down_cast<JNIEnvExt*>(env)->vm->IsCheckJniEnabled();
      for (ObjPtr<mirror::Class> current_class = c.Get();
           current_class != nullptr;
           current_class = current_class->GetSuperClass()) {
        // Search first only comparing methods which are native.
        m = FindMethod<true>(current_class.Ptr(), name, sig);
        if (m != nullptr) {
          break;
        }

        // Search again comparing to all methods, to find non-native methods that match.
        m = FindMethod<false>(current_class.Ptr(), name, sig);
        if (m != nullptr) {
          break;
        }

        if (warn_on_going_to_parent) {
          LOG(WARNING) << "CheckJNI: method to register \"" << name << "\" not in the given class. "
                       << "This is slow, consider changing your RegisterNatives calls.";
          warn_on_going_to_parent = false;
        }
      }

      if (m == nullptr) {
        c->DumpClass(LOG_STREAM(ERROR), mirror::Class::kDumpClassFullDetail);
        LOG(ERROR)
            << "Failed to register native method "
            << c->PrettyDescriptor() << "." << name << sig << " in "
            << c->GetDexCache()->GetLocation()->ToModifiedUtf8();
        ThrowNoSuchMethodError(soa, c.Get(), name, sig, "static or non-static");
        return JNI_ERR;
      } else if (!m->IsNative()) {
        LOG(ERROR)
            << "Failed to register non-native method "
            << c->PrettyDescriptor() << "." << name << sig
            << " as native";
        ThrowNoSuchMethodError(soa, c.Get(), name, sig, "native");
        return JNI_ERR;
      }

      VLOG(jni) << "[Registering JNI native method " << m->PrettyMethod() << "]";

      if (UNLIKELY(is_fast)) {
        // There are a few reasons to switch:
        // 1) We don't support !bang JNI anymore, it will turn to a hard error later.
        // 2) @FastNative is actually faster. At least 1.5x faster than !bang JNI.
        //    and switching is super easy, remove ! in C code, add annotation in .java code.
        // 3) Good chance of hitting DCHECK failures in ScopedFastNativeObjectAccess
        //    since that checks for presence of @FastNative and not for ! in the descriptor.
        LOG(WARNING) << "!bang JNI is deprecated. Switch to @FastNative for " << m->PrettyMethod();
        is_fast = false;
        // TODO: make this a hard register error in the future.
      }

      const void* final_function_ptr = m->RegisterNative(fnPtr, is_fast);
      UNUSED(final_function_ptr);
    }
    return JNI_OK;
  }
```
{% endcode-tabs-item %}
{% endcode-tabs %}

## Error Register

the error message once met is as below. The root reason is not declaration configHciSnoopLogNative method in AdapterService.java.

> 04-24 10:12:34.961 16805 16805 E zygote64: Failed to register native method com.android.bluetooth.btservice.AdapterService.configHciSnoopLogNative\(Z\)Z in /system/app/Bluetooth/Bluetooth.apk  
> 04-24 10:12:34.962 16805 16805 F zygote64: jni\_internal.cc:593\] JNI FatalError called: RegisterNatives failed for 'com/android/bluetooth/btservice/AdapterService'; aborting...



