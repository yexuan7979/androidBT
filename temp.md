---
description: temp usage
---

# temp

insert code

```text
/*******************************************************************************
**
** Function         bta_av_ci_data
**
** Description      forward the BTA_AV_CI_SRC_DATA_READY_EVT to stream state machine
**
**
** Returns          void
**
*******************************************************************************/
static void bta_av_ci_data(tBTA_AV_DATA *p_data)
{
    tBTA_AV_SCB *p_scb;
    int     i;
    UINT8   chnl = (UINT8)p_data->hdr.layer_specific;
    for( i=0; i < BTA_AV_NUM_STRS; i++ )
    {
        p_scb = bta_av_cb.p_scb[i];
        //Check if the Stream is in Started state before sending data
        //in Dual Handoff mode, get SCB where START is done.
        if(p_scb && (p_scb->chnl == chnl) && (p_scb->started))
        {
            bta_av_ssm_execute(p_scb, BTA_AV_SRC_DATA_READY_EVT, p_data);
        }
    }
}
```



```text
     boolean startDiscovery() {
        debugLog("startDiscovery");
        enforceCallingOrSelfPermission(BLUETOOTH_ADMIN_PERM,
                                       "Need BLUETOOTH ADMIN permission");
        //do not allow new connections with active multicast
        A2dpService a2dpService = A2dpService.getA2dpService();
        if (a2dpService != null &&
            a2dpService.isMulticastFeatureEnabled() &&
            a2dpService.isMulticastOngoing(null)) {
            Log.i(TAG,"A2dp Multicast is Ongoing, ignore discovery");
            return false;
        }

        if (mAdapterProperties.isDiscovering()) {
            Log.i(TAG,"discovery already active, ignore startDiscovery");
            return false;
        }
        return startDiscoveryNative();
    }
```



```text
        @Override
        public void clientConnect(int clientIf, String address, boolean isDirect, int transport,
                boolean opportunistic, int phy) {
            GattService service = getService();
            if (service == null) return;

            //do not allow new connections with active multicast
            A2dpService a2dpService = A2dpService.getA2dpService();
            if (a2dpService != null &&
                    a2dpService.isMulticastOngoing(null)) {
                Log.i(TAG,"A2dp Multicast is Ongoing, ignore Connection Request");
                return;
            }
            
            service.clientConnect(clientIf, address, isDirect, transport, opportunistic, phy);
        }
```



```text
        public void serverConnect(int serverIf, String address, boolean isDirect, int transport) {
            GattService service = getService();
            if (service == null) return;

            //do not allow new connections with active multicast
            A2dpService a2dpService = A2dpService.getA2dpService();
            if (a2dpService != null &&
                    a2dpService.isMulticastOngoing(null)) {
                Log.i(TAG,"A2dp Multicast is Ongoing, ignore Connection Request");
                return;
            }

            service.serverConnect(serverIf, address, isDirect, transport);
        }
```



```text
    public boolean connect(BluetoothDevice device) {
        enforceCallingOrSelfPermission(BLUETOOTH_PERM, "Need BLUETOOTH permission");
        A2dpService a2dpService = A2dpService.getA2dpService();
        //do not allow new connections with active multicast
        if (a2dpService != null &&
                a2dpService.isMulticastOngoing(device)) {
            Log.i(TAG,"A2dp Multicast is Ongoing, ignore Connection Request");
            return false;
        }

        if (getConnectionState(device) != BluetoothProfile.STATE_DISCONNECTED) {
            Log.e(TAG, "Pan Device not disconnected: " + device);
            return false;
        }
        /* Cancel discovery while initiating PANU connection, if It's in progress */
        if (mAdapter != null && mAdapter.isDiscovering()) {
            Log.d(TAG,"Inquiry is going on, Cancelling inquiry while initiating PANU connection");
            mAdapter.cancelDiscovery();
        }
        Message msg = mHandler.obtainMessage(MESSAGE_CONNECT,device);
        mHandler.sendMessage(msg);
        return true;
    }
```



```text
    public boolean connect(BluetoothDevice device) {
        enforceCallingOrSelfPermission(BLUETOOTH_ADMIN_PERM, "Need BLUETOOTH ADMIN permission");
        Log.d(TAG, "connect: device=" + device);
        if (getPriority(device) == BluetoothProfile.PRIORITY_OFF) {
            Log.w(TAG, "connect: PRIORITY_OFF, device=" + device);
            return false;
        }

        A2dpService a2dpService = A2dpService.getA2dpService();
        //do not allow new connections with active multicast
        if (a2dpService != null &&
                a2dpService.isMulticastOngoing(device)) {
            Log.i(TAG,"A2dp Multicast is Ongoing, ignore Connection Request");
            return false;
        }

        int connectionState = mStateMachine.getConnectionState(device);
        if (connectionState == BluetoothProfile.STATE_CONNECTED
                || connectionState == BluetoothProfile.STATE_CONNECTING) {
            Log.w(TAG,
                    "connect: already connected/connecting, connectionState=" + connectionState
                            + ", device=" + device);
            return false;
        }
        mStateMachine.sendMessage(HeadsetStateMachine.CONNECT, device);

        return true;
    }
```



```text
    boolean connect(BluetoothDevice device) {
        enforceCallingOrSelfPermission(BLUETOOTH_PERM, "Need BLUETOOTH permission");

        A2dpService a2dpService = A2dpService.getA2dpService();
        //do not allow new connections with active multicast
        if (a2dpService != null &&
                a2dpService.isMulticastOngoing(device)) {
            Log.i(TAG,"A2dp Multicast is Ongoing, ignore Connection Request");
            return false;
        }

        if (getConnectionState(device) != BluetoothInputDevice.STATE_DISCONNECTED) {
            Log.e(TAG, "Hid Device not disconnected: " + device);
            return false;
        }
        if (getPriority(device) == BluetoothInputDevice.PRIORITY_OFF) {
            Log.e(TAG, "Hid Device PRIORITY_OFF: " + device);
            return false;
        }

        Message msg = mHandler.obtainMessage(MESSAGE_CONNECT, device);
        mHandler.sendMessage(msg);
        return true;
    }
```



```text
        public boolean connect(BluetoothDevice device) {
            A2dpService service = getService();
            if (service == null) return false;
            //do not allow new connections with active multicast
            if (service.isMulticastOngoing(device)) {
                Log.i(TAG,"A2dp Multicast is Ongoing, ignore Connection Request");
                return false;
            }
            return service.connect(device);
```



```text
        @Override
        public void run() {
            timestamp = System.currentTimeMillis();

            //do not allow new connections with active multicast
            A2dpService a2dpService = A2dpService.getA2dpService();
            if (a2dpService != null &&
                    a2dpService.isMulticastOngoing(device)) {
                Log.i(TAG,"A2dp Multicast is Ongoing, ignore OPP send");
                return ;
            }
            .....
        }
```



```text
        public boolean setScanMode(int mode, int duration) {
            if (!Utils.checkCaller()) {
                Log.w(TAG, "setScanMode() - Not allowed for non-active user");
                return false;
            }

            //do not allow setmode when multicast is active
            A2dpService a2dpService = A2dpService.getA2dpService();
            if (a2dpService != null &&
                a2dpService.isMulticastFeatureEnabled() &&
                a2dpService.isMulticastOngoing(null)) {
                Log.i(TAG,"A2dp Multicast is Ongoing, ignore setmode " + mode);
                mScanmode = mode;
                return false;
            }

            AdapterService service = getService();
            if (service == null) return false;
            // when scan mode is not changed during multicast, reset it last to
            // scan mode, as we will set mode to none for multicast
            mScanmode = service.getScanMode();
            Log.i(TAG,"setScanMode: prev mode: " + mScanmode + " new mode: " + mode);
            return service.setScanMode(mode, duration);
        }
```



```text
     boolean createBond(BluetoothDevice device, int transport, OobData oobData) {
        //Flyme:suheng fix##516898,if adapter state not on,createBond may cause stack crash
        if (getState() != BluetoothAdapter.STATE_ON) {
            debugLog("AdapterState Not ON for CreateBond,just return false");
            return false;
        }
        //@}
        enforceCallingOrSelfPermission(BLUETOOTH_ADMIN_PERM,
            "Need BLUETOOTH ADMIN permission");
        DeviceProperties deviceProp = mRemoteDevices.getDeviceProperties(device);
        if (deviceProp != null && deviceProp.getBondState() != BluetoothDevice.BOND_NONE) {
            return false;
        }
        // Multicast: Do not allow bonding while multcast
        A2dpService a2dpService = A2dpService.getA2dpService();
        if (a2dpService != null &&
            a2dpService.isMulticastFeatureEnabled() &&
            a2dpService.isMulticastOngoing(null)) {
            Log.i(TAG,"A2dp Multicast is ongoing, ignore bonding");
            return false;
        }
        .....
    }
```



```text

```

