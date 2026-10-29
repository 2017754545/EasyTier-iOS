# 构建与签名说明

本仓库是 EasyTier iOS 客户端的自建分支，用于产出**未签名 IPA**，由使用者用自己的 Apple 开发者证书签名后安装。

- 上游代码：<https://github.com/EasyTier/EasyTier-iOS>（GPLv3）
- 本分支改动：
  - 包标识：`com.bg5ipl.easytier` / `.tunnel` / `.widgets`
  - App Group：`group.com.bg5ipl.easytier`
  - 移除 iCloud 容器相关 entitlements（无 iCloud 权限也能正常保存配置，自动回落本地 Documents）
  - Widget 扩展去掉不必要的 NetworkExtension entitlement，降低签名门槛
  - 构建流水线改为「iOS 27 SDK + 完全不签名」输出

## 构建

`.github/workflows/build-ipa.yml`，runner 使用 `xcode-27`（macOS 27 + Xcode 27.x + iOS 27 SDK）。

流程：

1. 选中 Xcode 27.x，打印 `iPhoneOS SDK` 版本
2. 安装 Rust `aarch64-apple-ios` target 与 `protoc`
3. `xcodebuild archive`，带 `CODE_SIGNING_ALLOWED=NO`
4. 打包 `Payload/EasyTier.app` → `EasyTier-unsigned.ipa`
5. 上传 artifact，并更新 `ipa-unsigned` release 资产

也可以手动触发（Actions → Build Unsigned IPA → Run workflow），可指定 `xcode_version`（如 `27.0` / `27.1`）。

## 自行签名

产物内部**完全未签名**，需要签三个 bundle，且顺序为由内到外：

1. `Payload/EasyTier.app/PlugIns/EasyTierWidgetExtension.appex` → 描述文件 `com.bg5ipl.easytier.widgets`
2. `Payload/EasyTier.app/PlugIns/EasyTierNetworkExtension.appex` → 描述文件 `com.bg5ipl.easytier.tunnel`
3. `Payload/EasyTier.app` → 描述文件 `com.bg5ipl.easytier`

三个 App ID 都需要开启 App Groups（`group.com.bg5ipl.easytier`），主 App 与 Network Extension 还需要开启 Network Extensions → Packet Tunnel。

用命令行签的话：

```bash
codesign -f -s "Apple Development: <你的名字> (XXXXXXXXXX)" \
  --entitlements tunnel.entitlements --generate-entitlement-der \
  Payload/EasyTier.app/PlugIns/EasyTierNetworkExtension.appex
codesign -f -s "Apple Development: <你的名字> (XXXXXXXXXX)" \
  --entitlements app.entitlements --generate-entitlement-der \
  Payload/EasyTier.app
```

再用 `zip -qry out.ipa Payload` 重新打包。
