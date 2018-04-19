---
description: list the differences between android O and N
---

# Android O update

## Architecture

android N

A Bluetooth system service communicates with the Bluetooth stack through JNI and with applications through Binder IPC. The system service provides developers with access to various Bluetooth profiles. This diagram shows the general structure of the Bluetooth stack:

![](.gitbook/assets/undefined%20%282%29.png)



Android O

A Bluetooth application communicates with the Bluetooth process through Binder. The Bluetooth process uses JNI to communicate with the Bluetooth stack and provides developers with access to various Bluetooth profiles. This diagram shows the general structure of the Bluetooth stack:

![](.gitbook/assets/undefined%20%281%29.png)

**Vendor implementation**

Vendor devices interact with the Bluetooth stack using the Hardware Interface Design Language \(HIDL\).

#### HIDL {#hidl}

[HIDL](https://source.android.com/devices/architecture/hidl.html) defines the interface between the Bluetooth stack and the vendor implementation. To generate the Bluetooth HIDL files, pass the Bluetooth interface files into the HIDL generation tool. The interface files are located in`hardware/interfaces/bluetooth`.

#### Bluetooth stack development {#bluetooth-stack-development}

The Android 8.0 Bluetooth stack is a fully qualified Bluetooth stack. The qualification listing is on the Bluetooth SIG website under [QDID 97584](https://www.bluetooth.org/tpg/QLI_viewQDL.cfm?qid=35890).



Preference: [https://source.android.com/devices/bluetooth/](https://source.android.com/devices/bluetooth/)

