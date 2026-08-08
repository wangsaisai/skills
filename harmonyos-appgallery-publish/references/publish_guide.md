# HarmonyOS AppGallery 上架完整参考指南

## 1. 工程版本配置

### `AppScope/app.json5` — 应用版本字段

```json5
{
  "app": {
    "bundleName": "com.example.yourapp",
    "versionCode": 1,         // 整数，每次上架必须递增，AppGallery 用此做版本去重
    "versionName": "1.0.0",   // 字符串，展示给用户
    "icon": "$media:app_icon",
    "label": "$string:app_name"
  }
}
```

**规则**：
- `versionCode` 纯数字，单调递增，上架前检查是否大于上一版本
- `versionName` 无格式限制，但建议用语义化版本号（1.0.0 / 1.0.1 / 1.1.0）
- 两者互相独立，`versionCode` 递增不要求 `versionName` 同步递增

---

## 2. SDK 版本配置

### `build-profile.json5` — SDK 版本字段

```json5
{
  "app": {
    "products": [
      {
        "name": "default",
        "compileSdkVersion": "HarmonyOS-5.0.5(17)",
        "compatibleSdkVersion": "HarmonyOS-5.0.0(12)",  // 最低可安装版本
        "runtimeOS": "HarmonyOS",
        "targetSdkVersion": "HarmonyOS-5.0.5(17)"       // 目标适配版本
      }
    ]
  }
}
```

### `compatibleSdkVersion` 详解

- 决定**哪些手机能安装**这个 App
- 设为 `5.0.0(12)` → HarmonyOS 5.0.0 (API 12) 及以上的设备都能安装
- 设为 `5.0.5(17)` → 仅 API 17 及以上设备能安装，老设备会被商店拦截

**最佳实践**：尽量设低，以覆盖更多设备；只有在代码里确实使用了高版本 API 且未做降级处理时，才被迫抬高。

### `targetSdkVersion` 详解

- 声明当前应用主要在哪个 API 版本下开发和测试
- 不影响安装门槛，只影响系统行为适配策略（某些系统功能会根据此值决定是否启用新行为）
- 通常设为当前开发 SDK 版本

### 推荐组合（证件夹项目）

```json5
"compatibleSdkVersion": "HarmonyOS-5.0.0(12)",
"targetSdkVersion": "HarmonyOS-5.0.5(17)"
```

---

## 3. Release 签名准备

### 三件套来源

| 文件 | 格式 | 来源 |
|---|---|---|
| 私钥库 | `.p12` | 本地在 DevEco Studio 生成（Build → Generate Key and CSR） |
| 证书 | `.cer` | 在 AppGallery Connect 申请后下载 |
| Profile | `.p7b` | 在 AppGallery Connect 创建 HarmonyOS Profile 后下载 |

### 生成 `.p12` 和 `.csr` 的步骤

1. DevEco Studio → Build → Generate Key and CSR
2. 填写 Key Store 路径（新建 `.p12` 文件）
3. 填写 Key Alias（后面配置里要用到，要记住）
4. 设置 Store Password 和 Key Password
5. 生成后得到：`.p12`（私钥）和 `.csr`（证书请求文件）

### 在 AGC 申请 `.cer` 的步骤

1. 登录 AppGallery Connect → 用户与访问 → 证书管理
2. 新增证书，上传 `.csr` 文件
3. 下载生成的 `.cer` 文件

### 在 AGC 生成 `.p7b` 的步骤

1. AppGallery Connect → 应用 → 你的应用 → HAP 包管理 → Profile 管理
2. 新建 Profile，类型选"发布"
3. 绑定你的 `.cer` 证书
4. 下载生成的 `.p7b` 文件

---

## 4. `build-profile.json5` 签名配置

### 完整 Release 签名配置示例

```json5
{
  "app": {
    "signingConfigs": [
      {
        "name": "default",
        "type": "HarmonyOS",
        "material": {
          "storeFile": "./your_key.p12",
          "storePassword": "your_store_password",
          "keyAlias": "your_key_alias",
          "keyPassword": "your_key_password",
          "signAlg": "SHA256withECDSA",
          "profile": "./your_profile.p7b",
          "certpath": "./your_cert.cer"
        }
      }
    ],
    "products": [
      {
        "name": "default",
        "signingConfig": "default",
        ...
      }
    ]
  }
}
```

**注意**：
- `storePassword` 和 `keyPassword` 可以是明文，但建议不提交进 git（加入 `.gitignore`）
- `keyAlias` 必须与 `.p12` 里的 alias 完全一致（大小写敏感）
- `certpath` 填 `.cer` 文件，不是 `.p7b`；两者不能混用

---

## 5. 构建 Release 包

### DevEco Studio GUI 方式

1. Build → Generate Signed Bundle/APK
2. 选择 `.p12`、填写密码和 Key Alias
3. 选择 Profile (`.p7b`)
4. Build Type 选 **Release**
5. 点击 Finish，等待构建完成
6. 输出文件在 `build/outputs/default/` 下，扩展名为 `.app` 或 `.hap`

### 命令行方式

```bash
# 进入工程根目录
cd /your/project/path

# 构建 Release 包
hvigorw assembleApp --mode project -p debuggable=false
```

---

## 6. AppGallery Connect 云测试

### 云测试设备"不适配"的原因

云测试平台的设备池是**公共资源**，设备可用性随时变化：

- 设备池中暂无该 API 版本的空闲设备
- 该 API 版本设备正被其他任务占用
- 测试场景类型限制（部分 API 版本只开放特定测试场景）

**这与应用的实际安装兼容性无关。**

### 云测试 vs 真实安装 的版本匹配逻辑

| 场景 | 匹配规则 |
|---|---|
| 云测试选设备 | 设备 API = 应用 `targetSdkVersion`（严格匹配或近似匹配） |
| 真实用户安装 | 设备 API ≥ 应用 `compatibleSdkVersion`（向下兼容） |

### 当云测试无可用设备时的处理策略

1. **等待设备池释放**：稍后重试，设备占用有时间窗口
2. **临时提高 `targetSdkVersion`**：让应用匹配当前池中 API 版本更高的设备；但不要动 `compatibleSdkVersion`
3. **用真机测试替代**：如果有真实设备，直接安装 Release 包测试

---

## 7. 常见错误排查

### 签名相关

| 错误信息 | 原因 | 解决方法 |
|---|---|---|
| `SignHap: key alias not found` | 配置里 `keyAlias` 不匹配 `.p12` 中的 alias | 在 DevEco Studio 的 Key Store 管理工具里核查 alias 名称 |
| `File content is not certificates` | `certpath` 字段指向了错误文件 | 检查 `.cer` 和 `.p7b` 是否搞混 |
| `Signing certificate expired` | `.cer` 过期 | 在 AGC 重新申请并下载新 `.cer` |
| `Profile bundleName does not match` | `.p7b` 绑定的 bundleName 与工程不一致 | 重新生成绑定正确 bundleName 的 Profile |

### 版本相关

| 错误信息 | 原因 | 解决方法 |
|---|---|---|
| AGC 拒绝上传：version already exists | `versionCode` 未递增 | 修改 `app.json5` 中 `versionCode`，增加整数值 |
| 安装失败：installation failed | 设备 API 低于 `compatibleSdkVersion` | 降低 `compatibleSdkVersion` 或使用更高版本设备 |

---

## 8. 上架前检查清单

- [ ] `versionCode` 已递增（大于上一版本）
- [ ] `versionName` 已更新（与发版说明一致）
- [ ] `compatibleSdkVersion` 设置合理（不要随意抬高）
- [ ] `.p12 / .cer / .p7b` 三件套齐全且路径正确
- [ ] `keyAlias` 与 `.p12` 中的一致
- [ ] 使用 Release 构建配置打包（非 Debug）
- [ ] 本地安装 Release 包验证基础功能
- [ ] AGC 应用信息（名称/图标/描述/截图）已更新
