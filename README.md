# installerx-repacked

> **⚠️ 免责声明 — 请先读**
>
> 本仓库是**非官方**重打包产物，与上游作者无任何关联；产物用于替换**系统安装器**
> `com.miui.packageinstaller`。安装或替换系统应用可能导致**无法安装应用、开机循环（bootloop）、
> OTA 失败、数据丢失、必须线刷恢复**。
> 一切后果由使用者自行承担，本仓库不提供任何担保与技术支持。
> 请仅在**你本人拥有、且具备救砖能力**的设备上使用。完整条款见 [免责声明](#免责声明)。

把 [wxxsfxyzm/InstallerX-Revived](https://github.com/wxxsfxyzm/InstallerX-Revived) 的 preview
重新打包成 **`com.miui.packageinstaller`**（HyperOS 系统安装器）身份。
每天检查上游 release，有新版本就自动重打包、签名、发 Release。

仓库只负责产出可装的 APK，**不碰你的设备**：没有 adb 自动化、没有定时安装、不需要 token。

## 产物

<https://github.com/FairyJJ/installerx-repacked/releases>

```
installerx-<upstream-tag>-repacked.apk
SHA256SUMS.txt
```

取用：

```sh
./scripts/download-latest.sh              # 下到 ./dist
adb install -r -t ./dist/installerx-*-repacked.apk
```

## 替换映射

| | |
|---|---|
| 上游 | `wxxsfxyzm/InstallerX-Revived`（含 prerelease，preview 就是 prerelease） |
| 原始 applicationId | `com.rosan.installer.x.revived` |
| 目标包名 | `com.miui.packageinstaller` |
| versionCode | 钉死 **99999999** |
| 类目录 | **不搬**（`--move-classes` 关闭） |
| Compose 资源 | 不需要 |

## 只改外层字符串，不动类路径

InstallerX 的 `applicationId` 是 `com.rosan.installer.x.revived`，但**代码类在 `com.rosan.installer.*`**，
两者不是一个字符串。而且它还依赖：

- `com.rosan.dhizuku.server.provider` —— Dhizuku 跨应用 provider
- `moe.shizuku.*` —— Shizuku

这些是**跨应用契约**，改了直接断掉 Shizuku/Dhizuku 通信。所以只替换 manifest 与字符串常量里的
`com.rosan.installer.x.revived`，类目录原样保留。

## priv-app 与 privileged 权限

目标路径 `/product/priv-app/MIUIPackageInstaller/`，设备上 `ro.control_privapp_permissions=enforce`。
上游声明的 27 个权限里只有 4 个是 `privileged`：

```
INSTALL_PACKAGES  DELETE_PACKAGES  MANAGE_USERS  UPDATE_APP_OPS_STATS
```

这 4 个在 `privapp-permissions-product.xml` 里都已存在 → 不会因白名单缺项导致开机卡死。
data 版覆盖安装后仍继承 priv-app 身份，`INSTALL_PACKAGES: granted=true`。

## 为什么 versionCode 钉死 99999999

`PackageManagerService` 开机扫描系统包时，若 `/data/app` 里该包 versionCode **小于**系统分区上的版本，
判定为「系统应用已升级、data 版是旧残留」，直接删掉 data 版 → 退回原版。
系统原版是 **55500**，所以拿 99999999 压住。

代价：应用内「检查更新」永远认为已是最新，升级只能从本仓库 Release 手动拉包覆盖安装。

## 工作流

`.github/workflows/build.yml`

- 触发：每天 03:00 UTC（11:00 CST）+ 手动 `workflow_dispatch`（勾 `force` 可强制重建）
- 解析上游最新 release（**包含 prerelease**），asset 挑名字含 `online` 的那个
- 若 `installerx-<tag>` 这个 Release 已存在 → 跳过
- 否则下载 APK → `scripts/repack.py` → 发布 Release + 上传 artifact

## 坑（脚本里已处理）

1. **apktool 的 build 缓存**：改完 `apktool.yml` 的 `versionInfo.versionCode` 直接 `b`，
   apktool 打印 `AndroidManifest.xml and resources have not changed` 且**不写入**新 versionCode。
   必须先 `rm -rf src/build`，脚本会强制清理并在构建后校验。
2. **别用全局替换**：只替换外层应用标识，类路径与 Shizuku/Dhizuku 依赖原样保留。
3. **签名固定**：uber-apk-signer 内置 debug key，每次构建签名一致，覆盖安装不冲突。
4. **保留桌面图标**：`LauncherAlias` 不删，装完能看到入口。

## 手动构建（本地）

```sh
curl -fsSL -o apktool.jar https://github.com/iBotPeaches/Apktool/releases/download/v3.0.3/apktool_3.0.3.jar
curl -fsSL -o uber-apk-signer.jar https://github.com/patrickfav/uber-apk-signer/releases/download/v1.3.0/uber-apk-signer-1.3.0.jar

APKTOOL_JAR=./apktool.jar SIGNER_JAR=./uber-apk-signer.jar \
python3 scripts/repack.py \
  --apk InstallerX.apk \
  --old com.rosan.installer.x.revived \
  --new com.miui.packageinstaller \
  --version-code 99999999 \
  --out-dir dist
```

## 回退

```sh
adb shell "pm uninstall-system-updates com.miui.packageinstaller"
```

## 上游与致谢

| | |
|---|---|
| 上游项目 | [wxxsfxyzm/InstallerX-Revived](https://github.com/wxxsfxyzm/InstallerX-Revived) |
| 上游作者 | [@wxxsfxyzm](https://github.com/wxxsfxyzm) |
| 项目主页 | <https://wxxsfxyzm.github.io/InstallerX-Revived-Website> |
| 上游许可证 | **GPL-3.0** |
| 本仓库性质 | 非官方，仅做包名替换 + 重签名 |

InstallerX-Revived 的全部功能（安装器 UI、Shizuku/Dhizuku 支持、安装逻辑）均为上游作者的成果。
本仓库**不修改任何业务代码**，只把 `applicationId` 从 `com.rosan.installer.x.revived` 替换为
`com.miui.packageinstaller`、抬 versionCode 顶替系统安装器。原项目请见上方链接，建议优先支持上游。

## 许可证

上游以 **GPL-3.0** 授权，因此本仓库的 APK 产物同样是 GPL-3.0 的派生作品，分发时：

- 保留原作者的版权声明与许可证声明（见 `LICENSE`）
- 提供对应源码 —— 源码 = 上游 tag 对应的代码（链接见每个 Release 的 notes）+ 本仓库的
  `scripts/repack.py`（唯一改动就是包名替换与 versionCode 覆盖，无业务补丁）
- 本仓库自身的构建脚本同样以 GPL-3.0 提供

## 免责声明

**下载、安装或使用本仓库任何产物，即表示你已阅读、理解并同意以下全部条款。如不同意，请立即停止使用。**

### 1. 非官方、无关联

本仓库由第三方维护，与 [wxxsfxyzm/InstallerX-Revived](https://github.com/wxxsfxyzm/InstallerX-Revived)
及其作者**没有任何隶属、授权、赞助或背书关系**。产物不是上游作者的正式发布版本。
**请勿**就本仓库产物的问题到上游仓库提 issue、向作者索要支持或要求其承担责任。

### 2. 系统级操作风险

本产物用于替换系统应用 `com.miui.packageinstaller`（HyperOS 系统安装器），属于**系统级改动**。
已知或可能的风险包括但不限于：

- 无法安装/卸载应用，安装器崩溃或反复弹窗
- 开机循环（bootloop）、系统服务起不来、无法进入桌面
- 系统 OTA 更新失败、更新后产物被覆盖或冲突
- 应用数据、设备数据丢失
- 需要恢复出厂设置、线刷/9008 救砖，甚至设备变砖

**风险与后果全部由使用者承担。** 请在使用前自行确认：设备可解锁、有可用的 fastboot / 9008 通道、
已完整备份重要数据（含 `/data`），并清楚回退方法（`pm uninstall-system-updates`）。

### 3. 无担保

产物按**「现状」（AS IS）**提供，不作任何明示或暗示的担保，包括但不限于适销性、
特定用途适用性、不侵权、无缺陷、以及不间断或无错误运行。是否可用、是否适配你的机型与系统版本，
需你自行验证。

### 4. 不提供技术支持

本仓库不承诺响应任何 issue、不提供安装指导、不承担因使用产物导致的任何直接或间接损失
（含数据丢失、设备损坏、业务中断、利润损失）。Issue 可以提，但不保证回复或修复。

### 5. 合法合规使用

仅限在**你本人拥有所有权，或已获得设备所有者明确授权**的设备上使用。
使用者应自行遵守所在地法律法规、设备厂商条款与保修约定。
**禁止**将本产物用于绕过付费、规避授权校验、欺诈、侵犯他人权利或其它违法用途。

### 6. 授权范围

上游以 **GPL-3.0** 授权，本产物作为其派生作品同样以 GPL-3.0 分发，**版权归上游作者**。
本仓库不主张对上游代码的任何权利，仅提供包名替换与重签名这一层改动（见 `scripts/repack.py`）。
若你认为本仓库的存在侵犯了你的权利，请提交 issue，维护者会及时处理（含删除仓库）。

### 7. 条款变更

本免责声明可能随时更新，以本文件当前内容为准，不另行通知。继续使用即视为接受更新后的条款。


