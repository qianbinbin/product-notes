# L9300 配置 PlayAutoInstallConfig

1. 修改源码，并放到`packages/apps/`下

    要避免生成 odex 文件，在`build_apk.mk`中配置：
    ```
    LOCAL_DEX_PREOPT := false
    ```

2. 编译生成`PlayAutoInstallConfig.apk`和其它 apk

3. 替换原 apk

  1. 将`PlayAutoInstallConfig.apk`重命名为`PlayAutoInstallConfigA1.apk`，并替换`vendor/longcheer/shamrock/google/unbundled_apps/PlayAutoInstallConfigA1/`下的同名文件
  2. 参考 L8150 项目，修改编译文件`vendor/longcheer/shamrock/google/unbundled_apps/PlayAutoInstallConfigA1/Android.mk`：

    ```
    -LOCAL_MODULE_TARGET_ARCH := x86
    +LOCAL_MODULE_TARGET_ARCH := arm arm64
    +LOCAL_MULTILIB := both
    ```
  （如果不修改，则不会生成 apk）

4. 修改编译配置文件`device/qcom/msm8952/shamrock/shamrock.mk`，在`PRODUCT_PACKAGES`项中增加 PlayAutoInstallConfigA1 配置：

  ```
  -    thermal-engine.conf
  +    thermal-engine.conf \
  +    PlayAutoInstallConfigA1
  ```

5. 编译全部源码，刷机

6. APFE 服务端进行配置

7. 手机端验证
