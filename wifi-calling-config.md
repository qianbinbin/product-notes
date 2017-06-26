# Wi-Fi Calling（WLAN 通话）相关配置

## Wi-Fi Calling 开关默认值配置

WFC (Wi-Fi Calling) 开关默认值，配置在`packages/apps/CarrierConfig/`下，`assets/carrier_config_*.xml`，其配置会在`res/xml/vendor.xml`中被覆盖，是一个属性名为`carrier_default_wfc_ims_enabled_bool`的 bool 值：

```java
/**
 * Default WFC_IMS_enabled: true VoWiFi by default is on
 *                          false VoWiFi by default is off
 * @hide
 */
public static final String KEY_CARRIER_DEFAULT_WFC_IMS_ENABLED_BOOL =
        "carrier_default_wfc_ims_enabled_bool";
```

未配置时默认为`false`。

其中`*`是国家码，参考：https://en.wikipedia.org/wiki/Mobile_country_code ，例如中国联通为 46001。

9300 项目会在`device/qcom/msm8952/shamrock/overlay/packages/apps/CarrierConfig/`中覆盖。

## Wi-Fi Calling 设置项是否显示配置

`WirelessSettings.java`中，Wi-Fi Calling 的显示与否与两个方法有关，如果两个方法都返回`true`，则显示这个开关，否则不显示：

```java
@Override
public void onResume() {
    super.onResume();

    // ...

    // update WFC setting
    final Context context = getActivity();
    if (ImsManager.isWfcEnabledByPlatform(context) &&
            ImsManager.isWfcProvisionedOnDevice(context)) {
        getPreferenceScreen().addPreference(mButtonWfc);

        mButtonWfc.setSummary(WifiCallingSettings.getWfcModeSummary(
                context, ImsManager.getWfcMode(context, mTm.isNetworkRoaming())));
    } else {
        removePreference(KEY_WFC_SETTINGS);
    }
}
```

### ImsManager.isWfcEnabledByPlatform()

```java
/**
 * Returns a platform configuration for WFC which may override the user
 * setting. Note: WFC presumes that VoLTE is enabled (these are
 * configuration settings which must be done correctly).
 */
public static boolean isWfcEnabledByPlatform(Context context) {
    if (SystemProperties.getInt(PROPERTY_DBG_WFC_AVAIL_OVERRIDE,
            PROPERTY_DBG_WFC_AVAIL_OVERRIDE_DEFAULT) == 1) {
        return true;
    }

    return
           context.getResources().getBoolean(
                   com.android.internal.R.bool.config_device_wfc_ims_available) &&
           getBooleanCarrierConfig(context,
                   CarrierConfigManager.KEY_CARRIER_WFC_IMS_AVAILABLE_BOOL) &&
           isGbaValid(context);
}
```

方法返回值主要与以下属性和方法有关。

#### persist.dbg.wfc_avail_ovr 属性

这个方法首先获取`persist.dbg.wfc_avail_ovr`这个属性，从命名看它可能是 debug 用的：

```java
/*
 * Debug flag to override configuration flag
 */

public static final String PROPERTY_DBG_WFC_AVAIL_OVERRIDE = "persist.dbg.wfc_avail_ovr";
```

如果为`1`，则方法直接返回`true`，如果不为`1`则继续。

在 9300 中，它定义在`device/qcom/msm8952/shamrock/system.prop`中，值为`1`，提交记录是 2017 年 3 月，由 Google 员工合入。

#### config_device_wfc_ims_available 属性

`config_device_wfc_ims_available`属性，从名称看可能用来表示设备本身 WFC 是否可用，配置位于`frameworks/base/core/res/res/values/config.xml`中，并在`device/qcom/msm8952/shamrock/overlay/frameworks/base/core/res/res/values/config.xml`中被覆盖。

在 9300 中，它的默认值为`false`，如果方法运行到这里则直接返回`false`。

#### carrier_wfc_ims_available_bool 属性

如果上面属性为`true`，继续获取属性`carrier_wfc_ims_available_bool`，其未配置时默认值为`false`，从名称看可能用来表示运营商 WFC 是否可用：

```java
/**
 * Flag specifying whether WFC over IMS should be available for carrier: independent of
 * carrier provisioning. If false: hard disabled. If true: then depends on carrier
 * provisioning, availability etc.
 */
public static final String KEY_CARRIER_WFC_IMS_AVAILABLE_BOOL = "carrier_wfc_ims_available_bool";
```

它的配置与 Wi-Fi Calling 开关默认值类似，无法在运行时修改。

#### isGbaValid()

GBA 是指通用引导架构，此方法是在运营商要求仅在使用 GBA SIM 卡时，IMS（IP 多媒体子系统）才可用，此时要检查 GBA 比特位，未配置时默认返回`true`。此方法尚未深入了解。

在 9300 插入联通卡时，返回值为`true`。

### ImsManager.isWfcProvisionedOnDevice()

```java
/**
 * Indicates whether VoWifi is provisioned on device
 */
public static boolean isWfcProvisionedOnDevice(Context context) {
    if (getBooleanCarrierConfig(context,
            CarrierConfigManager.KEY_CARRIER_VOLTE_PROVISIONING_REQUIRED_BOOL)) {
        ImsManager mgr = ImsManager.getInstance(context,
                SubscriptionManager.getDefaultVoicePhoneId());
        if (mgr != null) {
            return mgr.isWfcProvisioned();
        }
    }

    return true;
}
```

方法返回值与以下两个属性有关。

#### carrier_volte_provisioning_required_bool 属性

```java
/** Flag specifying whether provisioning is required for VOLTE. */
public static final String KEY_CARRIER_VOLTE_PROVISIONING_REQUIRED_BOOL
        = "carrier_volte_provisioning_required_bool";
```

未配置时默认为`false`，配置方法与 Wi-Fi Calling 开关默认值类似。因此在 9300 上此方法直接返回`true`。

#### net.lte.ims.wfc.provisioned 属性

`ImsManager#isWfcProvisioned()`方法实际获取的是如下属性：

```java
// SystemProperties used as cache

private static final String WFC_PROVISIONED_PROP = "net.lte.ims.wfc.provisioned";
```

这个属性是可以在运行时修改的，尚未深入了解。

9300 未配置这个属性，默认返回值为`true`，插入联通卡后系统也没有更新属性。
