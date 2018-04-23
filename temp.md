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

and let's go on with the mBluetooth.enable.

```text

```



```text

```



```text

```

