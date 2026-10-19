# installerx-repacked

> [!IMPORTANT]
>
> ### 免责与分发来源声明
>
> 1. **非官方重打包**：本仓库由第三方维护，与 [wxxsfxyzm/InstallerX-Revived](https://github.com/wxxsfxyzm/InstallerX-Revived) 及其作者**没有任何隶属、授权或背书关系**，产物不是上游作者的正式发布版本。请勿就本仓库产物的问题到上游仓库提 issue 或要求作者负责。
> 2. **不托管、不改代码**：本仓库不设立、不运营任何用于存储或分发第三方 APK 的服务器；所有产物由 GitHub Actions 从上游 Release 直接下载并构建，仅做包名替换与重签名（`scripts/repack.py`），不含任何业务代码。
> 3. **系统级风险**：产物用于替换**系统安装器** `com.miui.packageinstaller`。替换系统应用可能导致无法安装应用、安装器崩溃或反复弹窗、开机循环、OTA 失败或数据丢失，**全部后果由使用者自行承担**。
> 4. **版权与合规**：上游以 **GPL-3.0** 授权，产物作为其派生作品同样以 GPL-3.0 分发，版权归原作者所有；对应源码为上游 tag 代码 + 本仓库的 `scripts/repack.py`。严禁商业用途或非法牟利。

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

1. **用途限制**：本项目仅供技术研究学习及交流使用，严禁用于任何商业用途或非法牟利行为。
2. **风险自负**：替换系统安装器属于系统级改动，可能导致无法安装或卸载应用、安装器异常、无法开机、数据丢失，甚至需要线刷才能恢复。使用前请自行备份重要数据并确认设备具备救砖能力；因使用本产物造成的任何直接或间接损失，本仓库及维护者不承担责任。
3. **商标与版权**：文中提及的 Xiaomi、HyperOS、MIUIPackageInstaller、InstallerX、Shizuku、Dhizuku 等商标及产品名称，其商标权与著作权均归其合法所有者所有。上游代码、界面与资源的版权归 [wxxsfxyzm](https://github.com/wxxsfxyzm) 所有，本仓库仅提供包名替换与重签名这一层改动。
4. **授权范围**：上游以 GPL-3.0 授权，本仓库产物作为派生作品同样以 GPL-3.0 分发；分发时须保留原作者的版权与许可声明，并提供对应源码（上游 tag + `scripts/repack.py`）。若你认为本仓库的存在侵犯了你的权益，请提交 issue，维护者会及时处理（含删除仓库与相关 Release）。
