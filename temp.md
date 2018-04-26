---
description: description of Bluetooth enable procedure.
---

# Bluetooth enable

## Summary

register adapter

init stack

## enable

user or other apps should call enalbe function via bluetooth adapter.

### get default adapter

    typically use the default adapter.

    static method, use the only default adapter sAdapter. 

{% code-tabs %}
{% code-tabs-item title="frameworks/base/core/java/android/bluetooth/BluetoothAdapter.java" %}
```java
    /**
     * Get a handle to the default local Bluetooth adapter.
     * <p>Currently Android only supports one Bluetooth adapter, but the API
     * could be extended to support more. This will always return the default
     * adapter.
     * </p>
     * @return the default local adapter, or null if Bluetooth is not supported
     *         on this hardware platform
     */
    public static synchronized BluetoothAdapter getDefaultAdapter() {
        if (sAdapter == null) {
            IBinder b = ServiceManager.getService(BLUETOOTH_MANAGER_SERVICE);
            if (b != null) {
                IBluetoothManager managerService = IBluetoothManager.Stub.asInterface(b);
                sAdapter = new BluetoothAdapter(managerService);
            } else {
                Log.e(TAG, "Bluetooth binder is null");
            }
        }
        return sAdapter;
    }
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Bluetooth adapter register callback to bluetooth manager service.

{% code-tabs %}
{% code-tabs-item title="frameworks/base/core/java/android/bluetooth/BluetoothAdapter.java" %}
```java
    /**
     * Use {@link #getDefaultAdapter} to get the BluetoothAdapter instance.
     */
    BluetoothAdapter(IBluetoothManager managerService) {

        if (managerService == null) {
            throw new IllegalArgumentException("bluetooth manager service is null");
        }
        try {
            mServiceLock.writeLock().lock();
            mService = managerService.registerAdapter(mManagerCallback);
        } catch (RemoteException e) {
            Log.e(TAG, "", e);
        } finally {
            mServiceLock.writeLock().unlock();
        }
        mManagerService = managerService;
        mLeScanClients = new HashMap<LeScanCallback, ScanCallback>();
        mToken = new Binder();
    }
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### enable  in adapter

usually called like this:

    BluetoothAdapter mBluetoothAdapter = BluetoothAdapter.getDefaultAdapter\(\);

    mBluetoothAdapter.enable\(\);

make sure BLUETOOTH\_ADMIN permission owned before calling.

{% code-tabs %}
{% code-tabs-item title="frameworks/base/core/java/android/bluetooth/BluetoothAdapter.java" %}
```java
    /**
     * Turn on the local Bluetooth adapter&mdash;do not use without explicit
     * user action to turn on Bluetooth.
     * <p>This powers on the underlying Bluetooth hardware, and starts all
     * Bluetooth system services.
     * <p class="caution"><strong>Bluetooth should never be enabled without
     * direct user consent</strong>. If you want to turn on Bluetooth in order
     * to create a wireless connection, you should use the {@link
     * #ACTION_REQUEST_ENABLE} Intent, which will raise a dialog that requests
     * user permission to turn on Bluetooth. The {@link #enable()} method is
     * provided only for applications that include a user interface for changing
     * system settings, such as a "power manager" app.</p>
     * <p>This is an asynchronous call: it will return immediately, and
     * clients should listen for {@link #ACTION_STATE_CHANGED}
     * to be notified of subsequent adapter state changes. If this call returns
     * true, then the adapter state will immediately transition from {@link
     * #STATE_OFF} to {@link #STATE_TURNING_ON}, and some time
     * later transition to either {@link #STATE_OFF} or {@link
     * #STATE_ON}. If this call returns false then there was an
     * immediate problem that will prevent the adapter from being turned on -
     * such as Airplane mode, or the adapter is already turned on.
     *
     * @return true to indicate adapter startup has begun, or false on
     *         immediate error
     */
    @RequiresPermission(Manifest.permission.BLUETOOTH_ADMIN)
    public boolean enable() {
        android.util.SeempLog.record(56);
        if (isEnabled()) {
            if (DBG) Log.d(TAG, "enable(): BT already enabled!");
            return true;
        }
        try {
            return mManagerService.enable(ActivityThread.currentPackageName());
        } catch (RemoteException e) {Log.e(TAG, "", e);}
        return false;
    }
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### enable Bluetooth handled in manager service

    check permission and sendEnableMsg to BluetoothHandler

{% code-tabs %}
{% code-tabs-item title="frameworks/base/services/core/java/com/android/server/BluetoothManagerService.java" %}
```java
    public boolean enable(String packageName) throws RemoteException {
        final int callingUid = Binder.getCallingUid();
        final boolean callerSystem = UserHandle.getAppId(callingUid) == Process.SYSTEM_UID;

        if (isBluetoothDisallowed()) {
            if (DBG) {
                Slog.d(TAG,"enable(): not enabling - bluetooth disallowed");
            }
            return false;
        }

        if (!callerSystem) {
            if (!checkIfCallerIsForegroundUser()) {
                Slog.w(TAG, "enable(): not allowed for non-active and non system user");
                return false;
            }

            mContext.enforceCallingOrSelfPermission(BLUETOOTH_ADMIN_PERM,
                    "Need BLUETOOTH ADMIN permission");

            if (!isEnabled() && mPermissionReviewRequired
                    && startConsentUiIfNeeded(packageName, callingUid,
                            BluetoothAdapter.ACTION_REQUEST_ENABLE)) {
                return false;
            }
        }
        if (isStrictOpEnable()) {
            AppOpsManager mAppOpsManager = mContext.getSystemService(AppOpsManager.class);
            String packages = mContext.getPackageManager().getNameForUid(Binder.getCallingUid());
            if ((Binder.getCallingUid() >= Process.FIRST_APPLICATION_UID)
                    && (packages.indexOf("android.uid.systemui") != 0)
                    && (packages.indexOf("android.uid.system") != 0)) {
                int result = mAppOpsManager.noteOp(AppOpsManager.OP_BLUETOOTH_ADMIN,
                        Binder.getCallingUid(), packages);
                if (result == AppOpsManager.MODE_IGNORED) {
                    return false;
                }
            }
        }
        if (DBG) {
            Slog.d(TAG,"enable(" + packageName + "):  mBluetooth =" + mBluetooth +
                    " mBinding = " + mBinding + " mState = " +
                    BluetoothAdapter.nameForState(mState));
        }

        synchronized(mReceiver) {
            mQuietEnableExternal = false;
            mEnableExternal = true;
            // waive WRITE_SECURE_SETTINGS permission check
            sendEnableMsg(false, packageName);
        }
        if (DBG) Slog.d(TAG, "enable returning");
        return true;
    }
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### BluetoothHandler enable command

    mQuietEnable only use for system call.

{% code-tabs %}
{% code-tabs-item title="frameworks/base/services/core/java/com/android/server/BluetoothManagerService.java" %}
```java
                case MESSAGE_ENABLE:
                    if (DBG) {
                        Slog.d(TAG, "MESSAGE_ENABLE(" + msg.arg1 + "): mBluetooth = " + mBluetooth);
                    }
                    mHandler.removeMessages(MESSAGE_RESTART_BLUETOOTH_SERVICE);
                    mEnable = true;
                    ........
                    mQuietEnable = (msg.arg1 == 1);
                    if (mBluetooth == null) {
                        handleEnable(mQuietEnable);
                    } else {
                    ........
                    break;
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### handleEnalbe

    common scenario, mBluetooth is null. Bond to IBluetooth first.

{% code-tabs %}
{% code-tabs-item title="frameworks/base/services/core/java/com/android/server/BluetoothManagerService.java" %}
```java
    private void handleEnable(boolean quietMode) {
        mQuietEnable = quietMode;

        try {
            mBluetoothLock.writeLock().lock();
            if ((mBluetooth == null) && (!mBinding)) {
                //Start bind timeout and bind
                Message timeoutMsg=mHandler.obtainMessage(MESSAGE_TIMEOUT_BIND);
                mHandler.sendMessageDelayed(timeoutMsg,TIMEOUT_BIND_MS);
                Intent i = new Intent(IBluetooth.class.getName());
                if (!doBind(i, mConnection,Context.BIND_AUTO_CREATE | Context.BIND_IMPORTANT,
                        UserHandle.CURRENT)) {
                    mHandler.removeMessages(MESSAGE_TIMEOUT_BIND);
                } else {
                    mBinding = true;
                }
            } else if (mBluetooth != null) {
                //Enable bluetooth
                try {
                    if (!mQuietEnable) {
                        if(!mBluetooth.enable()) {
                            Slog.e(TAG,"IBluetooth.enable() returned false");
                        }
                    }
                    else {
                        if(!mBluetooth.enableNoAutoConnect()) {
                            Slog.e(TAG,"IBluetooth.enableNoAutoConnect() returned false");
                        }
                    }
        ........
    }
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### doBind

bind with service connection

{% code-tabs %}
{% code-tabs-item title="frameworks/base/services/core/java/com/android/server/BluetoothManagerService.java" %}
```java
    boolean doBind(Intent intent, ServiceConnection conn, int flags, UserHandle user) {
        ComponentName comp = intent.resolveSystemService(mContext.getPackageManager(), 0);
        intent.setComponent(comp);
        if (comp == null || !mContext.bindServiceAsUser(intent, conn, flags, user)) {
            Slog.e(TAG, "Fail to bind to: " + intent);
            return false;
        }
        return true;
    }
    
    private class BluetoothServiceConnection implements ServiceConnection {
        public void onServiceConnected(ComponentName componentName, IBinder service) {
            String name = componentName.getClassName();
            if (DBG) Slog.d(TAG, "BluetoothServiceConnection: " + name);
            Message msg = mHandler.obtainMessage(MESSAGE_BLUETOOTH_SERVICE_CONNECTED);
            if (name.equals("com.android.bluetooth.btservice.AdapterService")) {
                msg.arg1 = SERVICE_IBLUETOOTH;
            } else if (name.equals("com.android.bluetooth.gatt.GattService")) {
                msg.arg1 = SERVICE_IBLUETOOTHGATT;
            } else {
                Slog.e(TAG, "Unknown service connected: " + name);
                return;
            }
            msg.obj = service;
            mHandler.sendMessage(msg);
        }

        public void onServiceDisconnected(ComponentName componentName) {
            // Called if we unexpectedly disconnect.
            String name = componentName.getClassName();
            if (DBG) Slog.d(TAG, "BluetoothServiceConnection, disconnected: " + name);
            Message msg = mHandler.obtainMessage(MESSAGE_BLUETOOTH_SERVICE_DISCONNECTED);
            if (name.equals("com.android.bluetooth.btservice.AdapterService")) {
                msg.arg1 = SERVICE_IBLUETOOTH;
            } else if (name.equals("com.android.bluetooth.gatt.GattService")) {
                msg.arg1 = SERVICE_IBLUETOOTHGATT;
            } else {
                Slog.e(TAG, "Unknown service disconnected: " + name);
                return;
            }
            mHandler.sendMessage(msg);
        }
    }
```
{% endcode-tabs-item %}
{% endcode-tabs %}

and here is the Service Connection definition. after bind OK, onServiceConnected will be called.

{% code-tabs %}
{% code-tabs-item title="frameworks/base/core/java/android/content/ServiceConnection.java" %}
```java
/**
 * Interface for monitoring the state of an application service.  See
 * {@link android.app.Service} and
 * {@link Context#bindService Context.bindService()} for more information.
 * <p>Like many callbacks from the system, the methods on this class are called
 * from the main thread of your process.
 */
public interface ServiceConnection {
    /**
     * Called when a connection to the Service has been established, with
     * the {@link android.os.IBinder} of the communication channel to the
     * Service.
     *
     * @param name The concrete component name of the service that has
     * been connected.
     *
     * @param service The IBinder of the Service's communication channel,
     * which you can now make calls on.
     */
    void onServiceConnected(ComponentName name, IBinder service);

    /**
     * Called when a connection to the Service has been lost.  This typically
     * happens when the process hosting the service has crashed or been killed.
     * This does <em>not</em> remove the ServiceConnection itself -- this
     * binding to the service will remain active, and you will receive a call
     * to {@link #onServiceConnected} when the Service is next running.
     *
     * @param name The concrete component name of the service whose
     * connection has been lost.
     */
    void onServiceDisconnected(ComponentName name);

    /**
     * Called when the binding to this connection is dead.  This means the
     * interface will never receive another connection.  The application will
     * need to unbind and rebind the connection to activate it again.  This may
     * happen, for example, if the application hosting the service it is bound to
     * has been updated.
     *
     * @param name The concrete component name of the service whose
     * connection is dead.
     */
    default void onBindingDied(ComponentName name) {
    }
ja
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### BluetoothHandler service connected

onServiceConnected send service connected message to Bluetooth Handler. IBluetooth binder get the value here.

{% code-tabs %}
{% code-tabs-item title="frameworks/base/services/core/java/com/android/server/BluetoothManagerService.java" %}
```java
                case MESSAGE_BLUETOOTH_SERVICE_CONNECTED:
                {
                        ........
                        mBluetooth = IBluetooth.Stub.asInterface(Binder.allowBlocking(service));

                        if (!isNameAndAddressSet()) {
                            Message getMsg = mHandler.obtainMessage(MESSAGE_GET_NAME_AND_ADDRESS);
                            mHandler.sendMessage(getMsg);
                            if (mGetNameAddressOnly) return;
                        }

                        //Register callback object
                        try {
                            mBluetooth.registerCallback(mBluetoothCallback);
                        } catch (RemoteException re) {
                            Slog.e(TAG, "Unable to register BluetoothCallback",re);
                        }
                        //Inform BluetoothAdapter instances that service is up
                        sendBluetoothServiceUpCallback();

                        //Do enable request
                        try {
                            if (mQuietEnable == false) {
                                if (!mBluetooth.enable()) {
                                    Slog.e(TAG,"IBluetooth.enable() returned false");
                                }
                        ........
                }
```
{% endcode-tabs-item %}
{% endcode-tabs %}

IBluetooth binder register callback for bluetooth state change callback.

{% code-tabs %}
{% code-tabs-item title="frameworks/base/services/core/java/com/android/server/BluetoothManagerService.java" %}
```java
private final IBluetoothCallback mBluetoothCallback = new IBluetoothCallback.Stub() {
        @Override
        public void onBluetoothStateChange(int prevState, int newState) throws RemoteException  {
            Message msg = mHandler.obtainMessage(MESSAGE_BLUETOOTH_STATE_CHANGE,prevState,newState);
            mHandler.sendMessage(msg);
        }
    };
```
{% endcode-tabs-item %}
{% endcode-tabs %}

and let's go on with the mBluetooth.enable. And make it clear, the mBluetooth is the binder that bind BluetoothManagerService with Bluetooth AdapterService. 

So, mBluetooth.enable\(\) calls enable function in  AdapterService.java. 

{% code-tabs %}
{% code-tabs-item title="packages/apps/Bluetooth/src/com/android/bluetooth/btservice/AdapterService.java" %}
```java
     public boolean enable() {
         return enable(false);
     }

     public synchronized boolean enable(boolean quietMode) {
         enforceCallingOrSelfPermission(BLUETOOTH_ADMIN_PERM, "Need BLUETOOTH ADMIN permission");

         // Enforce the user restriction for disallowing Bluetooth if it was set.
         if (mUserManager.hasUserRestriction(UserManager.DISALLOW_BLUETOOTH, UserHandle.SYSTEM)) {
            debugLog("enable() called when Bluetooth was disallowed");
            return false;
         }

         debugLog("enable() - Enable called with quiet mode status =  " + mQuietmode);
         mQuietmode = quietMode;
         Message m = mAdapterStateMachine.obtainMessage(AdapterState.BLE_TURN_ON);
         mAdapterStateMachine.sendMessage(m);
         mBluetoothStartTime = System.currentTimeMillis();
         return true;
     }
```
{% endcode-tabs-item %}
{% endcode-tabs %}

By the way, mService in BluetoothAdapter.java and mBluetooth in BluetoothManagerService.java is the same object. And AdapterService is created as BluetoothManagerService do bind. Below is the bindService description from Android developler.

```coffeescript
bindService

boolean bindService (Intent service, 
                ServiceConnection conn, 
                int flags)
Connect to an application service, creating it if needed.

This defines a dependency between your application and the service.
The given conn will receive the service object when it is created and be told if it dies and restarts.
The service will be considered required by the system only for as long as the calling context exists.
For example, if this Context is an Activity that is stopped, the service will not be required to continue running until the Activity is resumed.
```

### AdapterState enable

And here process the adapter state machine. notifyAdapterStateChange  will send the BLE turning on state to BluetoothManagerService, and broadcast to the registers via BluetoothManagerService.

Then set a 2 seconds timer for gatt service start, as BleOnProcessStart is  to start gatt service mainly.

{% code-tabs %}
{% code-tabs-item title="packages/apps/Bluetooth/src/com/android/bluetooth/btservice/AdapterState.java" %}
```java
    private class OffState extends State {
        ........
        @Override
        public boolean processMessage(Message msg) {
            ........
            debugLog("Current state: OFF, message: " + msg.what);

            switch(msg.what) {
               case BLE_TURN_ON:
                   notifyAdapterStateChange(BluetoothAdapter.STATE_BLE_TURNING_ON);
                   mPendingCommandState.setBleTurningOn(true);
                   transitionTo(mPendingCommandState);
                   sendMessageDelayed(BLE_START_TIMEOUT, BLE_START_TIMEOUT_DELAY);
                   adapterService.BleOnProcessStart();
                   break;
               .........
    }
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Here is the implementation of BleOnPrecessStart, init config and jni callback, and start gatt service.

{% code-tabs %}
{% code-tabs-item title="packages/apps/Bluetooth/src/com/android/bluetooth/btservice/AdapterService.java" %}
```java
    void BleOnProcessStart() {
        debugLog("BleOnProcessStart()");

        if (getResources().getBoolean(
                R.bool.config_bluetooth_reload_supported_profiles_when_enabled)) {
            Config.init(getApplicationContext());
        }

        Class[] supportedProfileServices = Config.getSupportedProfiles();
        //Initialize data objects
        for (int i=0; i < supportedProfileServices.length;i++) {
            mProfileServicesState.put(supportedProfileServices[i].getName(),BluetoothAdapter.STATE_OFF);
        }

        debugLog("BleOnProcessStart() - Make Bond State Machine");

        mJniCallbacks.init(mBondStateMachine,mRemoteDevices);

        try {
            mBatteryStats.noteResetBleScan();
        } catch (RemoteException e) {
            // Ignore.
        }

        //Start Gatt service
        setGattProfileServiceState(supportedProfileServices,BluetoothAdapter.STATE_ON);
    }
```
{% endcode-tabs-item %}
{% endcode-tabs %}

start Gatt service in ProfileService,  start\(\) will route to GattService.

{% code-tabs %}
{% code-tabs-item title="packages/apps/Bluetooth/src/com/android/bluetooth/btservice/ProfileService.java" %}
```java
    private void doStart(Intent intent) {
        //Start service
        if (mAdapter == null) {
            Log.e(mName, "Error starting profile. BluetoothAdapter is null");
        } else {
            if (DBG) log("start()");
            mStartError = !start();
            if (!mStartError) {
                Log.d(mName, " profile started successfully");
                notifyProfileServiceStateChanged(BluetoothAdapter.STATE_ON);
            } else {
                Log.e(mName, "Error starting profile. BluetoothAdapter is null");
            }
        }
    }
```
{% endcode-tabs-item %}
{% endcode-tabs %}

As Gatt service started, notifyProfileServiceStateChanged will call back to adapter state machine. And hence, cancel the BLE start timer, enableNative to start stack, and start a 12 seconds' enable timeout.

{% code-tabs %}
{% code-tabs-item title="packages/apps/Bluetooth/src/com/android/bluetooth/btservice/AdapterState.java" %}
```java
                case BLE_STARTED:
                    //Remove start timeout
                    removeMessages(BLE_START_TIMEOUT);

                    //Enable
                    boolean isGuest = UserManager.get(mAdapterService).isGuestUser();
                    if (!adapterService.enableNative(isGuest)) {
                        errorLog("Error while turning Bluetooth on");
                        notifyAdapterStateChange(BluetoothAdapter.STATE_OFF);
                        transitionTo(mOffState);
                    } else {
                        sendMessageDelayed(ENABLE_TIMEOUT, ENABLE_TIMEOUT_DELAY);
                    }
                    break;
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### Stack Enable

enableNative routes to JNI bt service.

{% code-tabs %}
{% code-tabs-item title="packages/apps/Bluetooth/jni/com\_android\_bluetooth\_btservice\_AdapterService.cpp" %}
```cpp
static jboolean enableNative(JNIEnv* env, jobject obj, jboolean isGuest) {
    ALOGV("%s:",__FUNCTION__);

    jboolean result = JNI_FALSE;
    if (!sBluetoothInterface) return result;
    int ret = sBluetoothInterface->enable(isGuest == JNI_TRUE ? 1 : 0);
    result = (ret == BT_STATUS_SUCCESS || ret == BT_STATUS_DONE) ? JNI_TRUE : JNI_FALSE;
    return result;
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

sBluetoothInterface is init when loading JNI, which was called in AdapterService static variables. And sBluetoothInterface is the Bluetooth interface in stack.

Mark the code for getting bluetooth.default.so.

{% code-tabs %}
{% code-tabs-item title="hardware/libhardware/include/hardware/bluetooth.h" %}
```c
#define BT_HARDWARE_MODULE_ID "bluetooth"
#define BT_STACK_MODULE_ID "bluetooth"
#define BT_STACK_TEST_MODULE_ID "bluetooth_test"
```
{% endcode-tabs-item %}
{% endcode-tabs %}

{% code-tabs %}
{% code-tabs-item title="packages/apps/Bluetooth/jni/com\_android\_bluetooth\_btservice\_AdapterService.cpp" %}
```cpp
static void classInitNative(JNIEnv* env, jclass clazz) {
    int err;
    hw_module_t* module;
    ......
    char value[PROPERTY_VALUE_MAX];
    property_get("bluetooth.mock_stack", value, "");

    const char *id = (strcmp(value, "1")? BT_STACK_MODULE_ID : BT_STACK_TEST_MODULE_ID);

    err = hw_get_module(id, (hw_module_t const**)&module);

    if (err == 0) {
        hw_device_t* abstraction;
        err = module->methods->open(module, id, &abstraction);
        if (err == 0) {
            bluetooth_module_t* btStack = (bluetooth_module_t *)abstraction;
            sBluetoothInterface = btStack->get_bluetooth_interface();
        } else {
           ALOGE("Error while opening Bluetooth library");
        }
    } else {
        ALOGE("No Bluetooth Library found");
    }
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

And go on with the stack enable. 

{% code-tabs %}
{% code-tabs-item title="system/bt/btif/src/bluetooth.cc" %}
```cpp
static int enable(bool start_restricted) {
  LOG_INFO(LOG_TAG, "%s: start restricted = %d", __func__, start_restricted);

  restricted_mode = start_restricted;

  if (!interface_ready()) return BT_STATUS_NOT_READY;

  stack_manager_get_interface()->start_up_stack_async();
  return BT_STATUS_SUCCESS;
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

interface\_ready is to check if hal callback is ready, which is initiated when create AdapterService.

stack\_manager\_get\_interface init stack manager thread, which is used to manager start up and shut down asynchronous messages.

init\_stack: init osi, utils and config module.

{% code-tabs %}
{% code-tabs-item title="system/bt/btif/src/bluetooth.cc" %}
```cpp
static int init(bt_callbacks_t* callbacks) {
  LOG_INFO(LOG_TAG, "%s", __func__);

  if (interface_ready()) return BT_STATUS_DONE;

#ifdef BLUEDROID_DEBUG
  allocation_tracker_init();
#endif

  bt_hal_cbacks = callbacks;
  stack_manager_get_interface()->init_stack();
  btif_debug_init();
  return BT_STATUS_SUCCESS;
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

start\_up\_stack\_async, start up config module and enable bte main.

{% code-tabs %}
{% code-tabs-item title="system/bt/btif/src/stack\_manager.cc" %}
```cpp
static void start_up_stack_async(void) {
  thread_post(management_thread, event_start_up_stack, NULL);
}

// Synchronous function to start up the stack
static void event_start_up_stack(UNUSED_ATTR void* context) {
  ......
  // Include this for now to put btif config into a shutdown-able state
  module_start_up(get_module(BTIF_CONFIG_MODULE));
  bte_main_enable();

  if (future_await(local_hack_future) != FUTURE_SUCCESS) {
    LOG_ERROR(LOG_TAG, "%s failed to start up the stack", __func__);
    stack_is_running = true;  // So stack shutdown actually happens
    event_shut_down_stack(NULL);
    return;
  }

  stack_is_running = true;
  LOG_INFO(LOG_TAG, "%s finished", __func__);
  btif_thread_post(event_signal_stack_up, NULL);
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

bte\_main\_enable, start btsnoop and hci module, and start up btu.

{% code-tabs %}
{% code-tabs-item title="system/bt/main/bte\_main.cc" %}
```cpp
/******************************************************************************
 *
 * Function         bte_main_enable
 *
 * Description      BTE MAIN API - Creates all the BTE tasks. Should be called
 *                  part of the Bluetooth stack enable sequence
 *
 * Returns          None
 *
 *****************************************************************************/
void bte_main_enable() {
  APPL_TRACE_DEBUG("%s", __func__);

  module_start_up(get_module(BTSNOOP_MODULE));
  if (!module_start_up(get_module(HCI_MODULE))) {
    LOG_ERROR(LOG_TAG,
    "%s HCI_MODULE failed to start, Killing the bluetooth process", __func__);
    /* Killing the process to force a restart as part of fault tolerance */
    kill(getpid(), SIGKILL);
  }

  BTU_StartUp();
}

```
{% endcode-tabs-item %}
{% endcode-tabs %}

BTU\_StartUp

create thead bt\_workqueue\_thread, and post start up task to it.

{% code-tabs %}
{% code-tabs-item title="system/bt/stack/btu/btu\_init.cc" %}
```cpp
/*****************************************************************************
 *
 * Function         BTU_StartUp
 *
 * Description      Initializes the BTU control block.
 *
 *                  NOTE: Must be called before creating any tasks
 *                      (RPC, BTU, HCIT, APPL, etc.)
 *
 * Returns          void
 *
 *****************************************************************************/
void BTU_StartUp(void) {
  btu_trace_level = HCI_INITIAL_TRACE_LEVEL;

  bt_workqueue_thread = thread_new(BT_WORKQUEUE_NAME);
  if (bt_workqueue_thread == NULL) goto error_exit;

  thread_set_rt_priority(bt_workqueue_thread, BTU_TASK_RT_PRIORITY);

  // Continue startup on bt workqueue thread.
  thread_post(bt_workqueue_thread, btu_task_start_up, NULL);
  return;

error_exit:;
  LOG_ERROR(LOG_TAG, "%s Unable to allocate resources for bt_workqueue",
            __func__);
  BTU_ShutDown();
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

btu\_task\_start\_up

{% code-tabs %}
{% code-tabs-item title="system/bt/stack/btu/btu\_task.cc" %}
```cpp
void btu_task_start_up(UNUSED_ATTR void* context) {
  LOG(INFO) << "Bluetooth chip preload is complete";

  /* Initialize the mandatory core stack control blocks
     (BTU, BTM, L2CAP, and SDP)
   */
  btu_init_core();

  /* Initialize any optional stack components */
  BTE_InitStack();

  bta_sys_init();

  /* Initialise platform trace levels at this point as BTE_InitStack() and
   * bta_sys_init()
   * reset the control blocks and preset the trace level with
   * XXX_INITIAL_TRACE_LEVEL
   */
  module_init(get_module(BTE_LOGMSG_MODULE));
  // Inform the bt jni thread initialization is ok.
  btif_transfer_context(btif_init_ok, 0, NULL, 0, NULL);
  ........
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

btif\_init\_ok

{% code-tabs %}
{% code-tabs-item title="system/bt/btif/src/btif\_core.cc" %}
```cpp
void btif_init_ok(UNUSED_ATTR uint16_t event, UNUSED_ATTR char* p_param) {
  BTIF_TRACE_DEBUG("btif_task: received trigger stack init event");
  btif_dm_load_ble_local_keys();
  BTA_EnableBluetooth(bte_dm_evt);
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

bta\_dm\_enable

```cpp
/*******************************************************************************
 *
 * Function         bta_dm_enable
 *
 * Description      Initialises the BT device manager
 *
 *
 * Returns          void
 *
 ******************************************************************************/
void bta_dm_enable(tBTA_DM_MSG* p_data) {
  tBTA_DM_ENABLE enable_event;

  /* if already in use, return an error */
  if (bta_dm_cb.is_bta_dm_active == true) {
    APPL_TRACE_WARNING("%s Device already started by another application",
                       __func__);
    memset(&enable_event, 0, sizeof(tBTA_DM_ENABLE));
    enable_event.status = BTA_FAILURE;
    if (p_data->enable.p_sec_cback != NULL)
      p_data->enable.p_sec_cback(BTA_DM_ENABLE_EVT,
                                 (tBTA_DM_SEC*)&enable_event);
    return;
  }

  /* first, register our callback to SYS HW manager */
  bta_sys_hw_register(BTA_SYS_HW_BLUETOOTH, bta_dm_sys_hw_cback);

  /* make sure security callback is saved - if no callback, do not erase the
  previous one,
  it could be an error recovery mechanism */
  if (p_data->enable.p_sec_cback != NULL)
    bta_dm_cb.p_sec_cback = p_data->enable.p_sec_cback;
  /* notify BTA DM is now active */
  bta_dm_cb.is_bta_dm_active = true;

  /* send a message to BTA SYS */
  tBTA_SYS_HW_MSG* sys_enable_event =
      (tBTA_SYS_HW_MSG*)osi_malloc(sizeof(tBTA_SYS_HW_MSG));
  sys_enable_event->hdr.event = BTA_SYS_API_ENABLE_EVT;
  sys_enable_event->hw_module = BTA_SYS_HW_BLUETOOTH;

  bta_sys_sendmsg(sys_enable_event);
}
```

and go on with API enable event

bta\_sys\_hw\_api\_enable, and bta\_sys\_hw\_evt\_enabled\(BTM\_DeviceReset\).

{% code-tabs %}
{% code-tabs-item title="system/bt/stack/btm/btm\_devctl.cc" %}
```cpp
void BTM_DeviceReset(UNUSED_ATTR tBTM_CMPL_CB* p_cb) {
  /* Flush all ACL connections */
  btm_acl_device_down();

  /* Clear the callback, so application would not hang on reset */
  btm_db_reset();

  module_start_up_callbacked_wrapper(get_module(CONTROLLER_MODULE),
                                     bt_workqueue_thread, reset_complete);
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

start up controller molule, and callback to reset \_complete.  and reset complete will call back to bta\_sys\_hw\_btm\_cback with BTM\_DEV\_STATUS\_UP event.

{% code-tabs %}
{% code-tabs-item title="system/bt/bta/sys/bta\_sys\_main.cc" %}
```cpp
/******************************************************
*************************
 *
 * Function         bta_sys_hw_btm_cback
 *
 * Description     This function is registered by BTA SYS to BTM in order to get
 *                 status notifications
 *
 *
 * Returns
 *
 ******************************************************************************/
void bta_sys_hw_btm_cback(tBTM_DEV_STATUS status) {
  tBTA_SYS_HW_MSG* sys_event =
      (tBTA_SYS_HW_MSG*)osi_malloc(sizeof(tBTA_SYS_HW_MSG));

  APPL_TRACE_DEBUG("%s was called with parameter: %i", __func__, status);

  /* send a message to BTA SYS */
  if (status == BTM_DEV_STATUS_UP) {
    sys_event->hdr.event = BTA_SYS_EVT_STACK_ENABLED_EVT;
  } else if (status == BTM_DEV_STATUS_DOWN) {
    sys_event->hdr.event = BTA_SYS_ERROR_EVT;
  } else {
    /* BTM_DEV_STATUS_CMD_TOUT is ignored for now. */
    osi_free_and_reset((void**)&sys_event);
  }

  if (sys_event) bta_sys_sendmsg(sys_event);
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}



