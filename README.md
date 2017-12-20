# SmartDeviceLink android proxy升级指南
本文介绍了如何在android移动应用程序上集成SmartDeviceLink android proxy 4.4.0及以上版本，并提供示例供开发人员参考。
## 添加SmartDeviceLink
首先，请在[SmartDeviceLink网站](https://github.com/smartdevicelink/sdl_android/releases)上查看最新版本的release，下载源代码包到本地解压。然后在Android Studio里添加SDL模块,在菜单上选择File->New->Import Module...。
![import_module](/image/import_module.png)
选择先前下载的SmartDeviceLink android代码包顶层文件夹，并将模块名称改为“sdl_android”，完成导入。
![select_module](/image/select_module.png)
导入完成以后应该可以在项目工程里看到sdl_android模块
![project](/image/project.png)

为应用程序添加模块依赖
![dependencies](/image/dependencies.png)
## 集成SmartDeviceLink
### SDLService
创建新的SmartDeviceLink服务类
```java
public class SdlService extends Service implements IProxyListenerALM
```
添加类成员proxy，用于处理应用程序和车机之间的通信
```java
//The proxy handles communication between the application and SDL
private SdlProxyALM proxy = null;
```
重写onStartCommand方法，将“应用程序名称”和“应用程序ID”替换成应用程序分配到的名称和ID。
```java
@Override
public int onStartCommand(Intent intent, int flags, int startId) {
    boolean forceConnect = intent !=null && intent.getBooleanExtra(TransportConstants.FORCE_TRANSPORT_CONNECTED, false);

    if (proxy == null) {
        try {
            BaseTransportConfig transportConfig = null;

            if(Build.VERSION.SDK_INT > Build.VERSION_CODES.HONEYCOMB){
                if(intent!=null && intent.hasExtra(UsbManager.EXTRA_ACCESSORY)) {
                    transportConfig = new USBTransportConfig(getBaseContext(), (UsbAccessory) intent.getParcelableExtra(UsbManager.EXTRA_ACCESSORY), false, false); // create USB transport
                }
            } else{
                Log.e("SdlService", "Unable to start proxy. Android OS version is too low"); // optional
            }

            if(transportConfig != null){
                proxy = new SdlProxyALM(this, "应用程序名称", false, "应用程序ID", transportConfig);
            }
        } catch (SdlException e) {
            //There was an error creating the proxy
            if (proxy == null) {
                //Stop the SdlService
                stopSelf();
            }
        }
    }else if(forceConnect){
        proxy.forceOnConnected();
    }

    //use START_STICKY because we want the SDLService to be explicitly started and stopped as needed.
    return START_STICKY;
}
```
重写onDestroy和onBind方法
```java
@Override
public void onDestroy() {
    //Dispose of the proxy
    if (proxy != null) {
        try {
            proxy.dispose();
        } catch (SdlException e) {
            e.printStackTrace();
        } finally {
            proxy = null;
        }
    }

    super.onDestroy();
}

@Nullable
@Override
public IBinder onBind(Intent intent) {
    return null;
}
```
其他部分和以前一样，需要在SDLService类里重写SmartDeviceLink相关的回调方法。开发者可以利用Android Studio的警告消息自动完成这些部分。

最后在AndroidManifest.xml里面添加SdlService
```xml
<service
    android:name=".SdlService"
    android:enabled="true"/>
```
### SdlReceiver
创建新的SmartDeviceLink接收消息广播类
```java
public class SdlReceiver extends SdlBroadcastReceiver
```
重写SdlBroadcastReceiver的三个方法
```java
@Override
public void onSdlEnabled(Context context, Intent intent) {
    //Use the provided intent but set the class to your SdlService
    intent.setClass(context, SdlService.class);
    context.startService(intent);
}

@Override
public Class<? extends SdlRouterService> defineLocalSdlRouterClass() {
    return null; //Since we use AOA, we will not be using multiplexing
}

@Override
public void onReceive(Context context, Intent intent){
    super.onReceive(context, intent);
    //Your code
}
```
在应用程序的MainActivity里面启动监听
```java
SdlReceiver.queryForConnectedService(this);
```
在AndroidManifest.xml里面添加SdlReceiver
```xml
<receiver android:name=".SdlReceiver"
    android:exported="true"
    android:enabled="true" >
    <intent-filter>
        <action android:name="com.smartdevicelink.USB_ACCESSORY_ATTACHED"/> <!--For AOA -->
    </intent-filter>
</receiver>
```
### AOA
为了让应用程序支持AOA，首先在AndroidManifest.xml里面添加
```xml
<uses-feature android:name="android.hardware.usb.accessory"/>
```
然后在工程里添加accessory_filter.xml，内容如下
```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <usb-accessory manufacturer="SDL" model="Core" version="1.0"/>
</resources>
```
accessory_filter.xml添加完成以后，在AndroidManifest.xml里添加
```xml
<activity android:name="com.smartdevicelink.transport.USBAccessoryAttachmentActivity"
    android:launchMode="singleTop">
    <intent-filter>
        <action android:name="android.hardware.usb.action.USB_ACCESSORY_ATTACHED" />
    </intent-filter>

    <meta-data
        android:name="android.hardware.usb.action.USB_ACCESSORY_ATTACHED"
        android:resource="@xml/accessory_filter" />
</activity>
```
