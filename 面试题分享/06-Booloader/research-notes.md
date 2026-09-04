# Zephyr + MCUboot Bootloader 调研记录

调研日期：2026-08-09

## 结论

选择 Zephyr + MCUboot，采用 swap 模式下的 test upgrade 作为视频案例。

选择原因：

- Zephyr 和 MCUboot 都在持续维护，且采用 Apache-2.0 许可证。
- Zephyr 官方 DFU 文档明确集成 MCUboot。
- MCUboot 官方设计文档给出了镜像格式、Flash map、双槽、swap、image trailer、确认和回滚流程。
- 关键结论能在 `bootutil_public.h`、`image_validate.c` 等源码中交叉验证。
- 整套设计能压缩成“保留旧版本、验证新版本、断点交换、试运行确认、失败回滚”一条叙事链。

## 候选项目比较

### Zephyr + MCUboot

优点：安全校验、断电恢复和功能回滚三条链完整，官方设计资料详细，适合解释设计原因。

限制：MCUboot 支持多种升级模式，本期必须明确只讲 swap + test/revert，不能把某一种模式说成唯一实现。

### ESP-IDF 二级 Bootloader + A/B OTA

优点：项目成熟，`otadata` 双扇区、镜像状态和应用回滚机制都很清楚。

限制：ROM Bootloader、二级 Bootloader、分区表、OTA data、Secure Boot 和 eFuse anti-rollback 同时出现，400 字很难交代清楚层级。

### PX4 Bootloader

优点：真实飞控项目，串口/USB 固件刷写和应用跳转流程直观。

限制：核心故事更偏刷写协议，双槽试运行与自动回滚不是它最突出的设计主线。

## 官方证据与对应结论

### 1. Zephyr 的 MCUboot 分区

MCUboot 的 Zephyr 文档要求定义：

- `boot_partition`：MCUboot 本体。
- `slot0_partition`：主镜像槽。
- `slot1_partition`：副镜像槽。

对应视频结论：新镜像先进入副槽，当前可运行镜像不在下载阶段被覆盖。

来源：

- [Building and using MCUboot with Zephyr](https://github.com/mcu-tools/mcuboot/blob/main/docs/readme-zephyr.md)
- [Zephyr Device Firmware Upgrade](https://docs.zephyrproject.org/latest/services/device_mgmt/dfu.html)

### 2. 镜像验证

MCUboot 镜像 TLV 可包含 SHA hash、RSA/ECDSA/Ed25519 签名和 security counter。签名文档说明私钥负责发布端签名，Bootloader 中只需要验证公钥。

`image_validate.c` 中的 `bootutil_img_validate()` 会计算镜像 hash、比对 hash TLV，并调用签名验证函数；启用硬件防回滚时还会处理 `IMAGE_TLV_SEC_CNT`。

对应视频结论：hash 解决损坏检测，签名解决发布者认证，security counter 可阻止降级。

来源：

- [MCUboot Design - Image format](https://docs.mcuboot.com/design.html#image-format)
- [MCUboot Signed Images](https://github.com/mcu-tools/mcuboot/blob/main/docs/signed_images.md)
- [image_validate.c](https://github.com/mcu-tools/mcuboot/blob/main/boot/bootutil/src/image_validate.c)
- [image.h](https://github.com/mcu-tools/mcuboot/blob/main/boot/bootutil/include/bootutil/image.h)

### 3. Test、Permanent 与 Revert

官方设计定义：

- `BOOT_SWAP_TYPE_TEST`：交换并试运行；未确认时下次启动回滚。
- `BOOT_SWAP_TYPE_PERM`：永久切换到新镜像。
- `BOOT_SWAP_TYPE_REVERT`：此前 test 没有被确认，把旧镜像换回。

公共 API 中：

- `boot_set_pending(permanent = 0)` 请求一次试运行。
- `boot_set_confirmed()` 确认当前主槽镜像。

对应视频结论：数字签名通过后仍要进行运行时自检，因为安全真实性与业务可用性是两件事。

来源：

- [MCUboot Design - Boot swap types](https://docs.mcuboot.com/design.html#boot-swap-types)
- [bootutil_public.h](https://github.com/mcu-tools/mcuboot/blob/main/boot/bootutil/include/bootutil/bootutil_public.h)

### 4. Image trailer 与断电恢复

官方设计说明 image trailer 位于镜像槽末尾，包含 swap status、swap info、`copy_done`、`image_ok` 和 magic。交换过程中会持续写入各扇区的进度，全部交换完成后再设置 `copy_done`。设备复位后可以判断已经完成到哪一步并继续。

对应视频结论：交换进度不能只放在 RAM；每一步写入 Flash，才能抵抗升级过程中的掉电和复位。

来源：

- [MCUboot Design - Image trailer](https://docs.mcuboot.com/design.html#image-trailer)
- [MCUboot Design - High-level operation](https://docs.mcuboot.com/design.html#high-level-operation)

## 需要避免的错误表述

- 不说“MCUboot 负责从网络下载固件”。下载通常由应用、MCUmgr 或设备管理层完成，MCUboot 负责启动决策和镜像处理。
- 不说“MCUboot 永远使用 A/B 原地启动”。本期讲的是 swap 模式；它还支持 overwrite、direct-XIP 和 RAM-load。
- 不说“签名通过就代表固件一定正常”。签名只证明完整性和来源，运行功能仍需应用自检和确认。
- 不说“交换过程绝对不会失败”。更准确的说法是 trailer 状态让中断后的交换可以恢复，无法恢复的错误仍可能进入 FAIL/PANIC。
- 不把 `image_ok` 说成 Bootloader 自动写入。试运行成功后应由新应用主动确认。

## 项目入口

- [MCUboot GitHub](https://github.com/mcu-tools/mcuboot)
- [Zephyr GitHub](https://github.com/zephyrproject-rtos/zephyr)
- [MCUboot 官方文档](https://docs.mcuboot.com/)
