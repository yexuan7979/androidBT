# Bluetooth init

Description of the Bluetooth init procedure in android O.

## start bluetooth service

When boot android system, the system server will be created and start the necessary server.  And Bluetooth service is started here.

```java
frameworks/base/services/java/com/android/server/SystemServer.java
    /**
     * Starts a miscellaneous grab bag of stuff that has yet to be refactored
     * and organized.
     */
    private void startOtherServices() {
         .........
            // Skip Bluetooth if we have an emulator kernel
            // TODO: Use a more reliable check to see if this product should
            // support Bluetooth - see bug 988521
            if (isEmulator) {
                Slog.i(TAG, "No Bluetooth Service (emulator)");
            } else if (mFactoryTestMode == FactoryTest.FACTORY_TEST_LOW_LEVEL) {
                Slog.i(TAG, "No Bluetooth Service (factory test)");
            } else if (!context.getPackageManager().hasSystemFeature
                       (PackageManager.FEATURE_BLUETOOTH)) {
                Slog.i(TAG, "No Bluetooth Service (Bluetooth Hardware Not Present)");
            } else if (disableBluetooth) {
                Slog.i(TAG, "Bluetooth Service disabled by config");
            } else {
                traceBeginAndSlog("StartBluetoothService");
                mSystemServiceManager.startService(BluetoothService.class);
                traceEnd();
            }
        .........
        mSystemServiceManager.startBootPhase(SystemService.PHASE_SYSTEM_SERVICES_READY);
        .........
        mActivityManagerService.systemReady(new Runnable() {
            @Override
            public void run() {
                Slog.i(TAG, "Making services ready");
                mSystemServiceManager.startBootPhase(
                        SystemService.PHASE_ACTIVITY_MANAGER_READY);
                Trace.traceBegin(Trace.TRACE_TAG_SYSTEM_SERVER, "PhaseActivityManagerReady");
        ........
    }
```

### Bluetooth service

and in Bluetooth service, create  Bluetooth manager service and react on boot phase to add bluetooth service into system and check if enable or get information.

```text
frameworks/base/services/core/java/com/android/server/BluetoothService.java
class BluetoothService extends SystemService {
    private BluetoothManagerService mBluetoothManagerService;

    public BluetoothService(Context context) {
        super(context);
        mBluetoothManagerService = new BluetoothManagerService(context);
    }

    @Override
    public void onStart() {
    }

    @Override
    public void onBootPhase(int phase) {
        if (phase == SystemService.PHASE_SYSTEM_SERVICES_READY) {
            publishBindervaluablesService(BluetoothAdapter.BLUETOOTH_MANAGER_SERVICE,
                    mBluetoothManagerService);
        } else if (phase == SystemService.PHASE_ACTIVITY_MANAGER_READY) {
            mBluetoothManagerService.handleOnBootPhase();
        }
    }
    ....
}

```

### Publish binder service

publishBinderService: accessible to other services and apps

```text
    /**
     * Publish the service so it is accessible to other services and apps.
     */
    protected final void publishBinderService(String name, IBinder service) {
        publishBinderService(name, service, false);
    }

    /**
     * Publish the service so it is accessible to other services and apps.
     */
    protected final void publishBinderService(String name, IBinder service,
            boolean allowIsolated) {
        ServiceManager.addService(name, service, allowIsolated);
    }
```

### Bluetooth manager service

BluetoothManagerService, from the class name, obviously use to manager the bluetooth service.        

        Bluetooth Handler: handle messages as below.

```text
    private static final int MESSAGE_ENABLE = 1;
    private static final int MESSAGE_DISABLE = 2;
    private static final int MESSAGE_REGISTER_ADAPTER = 20;
    private static final int MESSAGE_UNREGISTER_ADAPTER = 21;
    private static final int MESSAGE_REGISTER_STATE_CHANGE_CALLBACK = 30;
    private static final int MESSAGE_UNREGISTER_STATE_CHANGE_CALLBACK = 31;
    private static final int MESSAGE_BLUETOOTH_SERVICE_CONNECTED = 40;
    private static final int MESSAGE_BLUETOOTH_SERVICE_DISCONNECTED = 41;
    private static final int MESSAGE_RESTART_BLUETOOTH_SERVICE = 42;
    private static final int MESSAGE_BLUETOOTH_STATE_CHANGE = 60;
    private static final int MESSAGE_TIMEOUT_BIND = 100;
    private static final int MESSAGE_TIMEOUT_UNBIND = 101;
    private static final int MESSAGE_GET_NAME_AND_ADDRESS = 200;
    private static final int MESSAGE_USER_SWITCHED = 300;
    private static final int MESSAGE_USER_UNLOCKED = 301;
    private static final int MESSAGE_ADD_PROXY_DELAYED = 400;
    private static final int MESSAGE_BIND_PROFILE_SERVICE = 401;
    private static final int MESSAGE_RESTORE_USER_SETTING = 500;
```

        register receivers and check if set the auto enable flag as below shows.

```text
frameworks/base/services/core/java/com/android/server/BluetoothManagerService.java
    BluetoothManagerService(Context context) {
        mHandler = new BluetoothHandler(IoThread.get().getLooper());
        ......
        // Observe BLE scan only mode settings change.
        registerForBleScanModeChange();
        mCallbacks = new RemoteCallbackList<IBluetoothManagerCallback>();
        mStateChangeCallbacks = new RemoteCallbackList<IBluetoothStateChangeCallback>();

        IntentFilter filter = new IntentFilter();
        filter.addAction(BluetoothAdapter.ACTION_LOCAL_NAME_CHANGED);
        filter.addAction(BluetoothAdapter.ACTION_BLUETOOTH_ADDRESS_CHANGED);
        filter.addAction(Intent.ACTION_SETTING_RESTORED);
        filter.setPriority(IntentFilter.SYSTEM_HIGH_PRIORITY);
        mContext.registerReceiver(mReceiver, filter);

        loadStoredNameAndAddress();
        if (isBluetoothPersistedStateOn()) {
            if (DBG) Slog.d(TAG, "Startup: Bluetooth persisted state is ON.");
            mEnableExternal = true;
        }

        String airplaneModeRadios = Settings.Global.getString(mContentResolver,
            Settings.Global.AIRPLANE_MODE_RADIOS);
        if (airplaneModeRadios == null ||
            airplaneModeRadios.contains(Settings.Global.RADIO_BLUETOOTH)) {
            mContentResolver.registerContentObserver(
                Settings.Global.getUriFor(Settings.Global.AIRPLANE_MODE_ON),
                true, mAirplaneModeObserver);
        }
        ........
    }

```

and go on handleOnBootPhase in BluetoothManagerService, the annotation is clear.

```text
    /**
     * Send enable message and set adapter name and address. Called when the boot phase becomes
     * PHASE_SYSTEM_SERVICES_READY.
     */
    public void handleOnBootPhase() {
        if (DBG) Slog.d(TAG, "Bluetooth boot completed");
        UserManagerInternal userManagerInternal =
                LocalServices.getService(UserManagerInternal.class);
        userManagerInternal.addUserRestrictionsListener(mUserRestrictionsListener);
        final boolean isBluetoothDisallowed = isBluetoothDisallowed();
        if (isBluetoothDisallowed) {
            return;
        }
        if (mEnableExternal && isBluetoothPersistedStateOnBluetooth()) {
            if (DBG) Slog.d(TAG, "Auto-enabling Bluetooth.");
            sendEnableMsg(mQuietEnableExternal, REASON_SYSTEM_BOOT);
        } else if (!isNameAndAddressSet()) {
            if (DBG) Slog.d(TAG, "Getting adapter name and address");
            Message getMsg = mHandler.obtainMessage(MESSAGE_GET_NAME_AND_ADDRESS);
            mHandler.sendMessage(getMsg);
        }
    }
```



Here the init procedure done.

## Summary

start bluetooth service  when system server is up.

create bluetooth manager service to handle further messages, listen for associated bluetooth event and init local variables.

check the bluetooth name and address, and auto enable or not.

