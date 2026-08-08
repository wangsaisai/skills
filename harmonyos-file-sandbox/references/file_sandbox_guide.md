# HarmonyOS 文件沙箱完整参考指南

## 1. 沙箱目录体系

### 应用可用的沙箱根路径

```typescript
import { common } from '@kit.AbilityKit';

const context = getContext(this) as common.UIAbilityContext;

context.filesDir   // /data/storage/el2/base/haps/entry/files  — 持久存储，推荐用于原始图片
context.cacheDir   // /data/storage/el2/base/haps/entry/cache  — 可被系统清理的缓存
context.tempDir    // /data/storage/el2/base/haps/entry/temp   — 临时文件，重启后可能消失
```

**规则**：
- 图片、证件数据等需要持久保存的文件放 `filesDir`
- 缩略图、预览图等可重新生成的文件放 `cacheDir`
- 导出处理中的临时文件放 `tempDir`，操作完后主动清理

### 推荐子目录结构

```
filesDir/
├── raw/       # 从相册导入的原始图片（JPEG/PNG）
├── thumb/     # 生成的缩略图（小尺寸 JPEG）
├── export/    # 用户导出操作生成的 PDF / ZIP
└── temp/      # 图片处理中间临时文件（每次启动清理）
```

---

## 2. URI 类型区分

HarmonyOS 里有两种完全不同的 URI：

### 媒体 URI（Media URI）

```
file://media/Photo/1/IMG_20240101_120000.jpg
```

- 由相册 Picker 返回
- 代表**媒体库管理的文件**，有权限管理
- **不能直接传给 `fs.openSync`**（没有沙箱访问权限）
- 只能通过 `photoAccessHelper` 的授权机制读取，或通过 `fs.open(uri)` 在授权上下文里打开

### 沙箱路径（Sandbox Path）

```
/data/storage/el2/base/haps/entry/files/raw/image_001.jpg
```

- 应用自己的文件系统路径
- 可以直接用 `fs.openSync / fs.copyFile / fs.readText` 等操作
- 通过 `context.filesDir + '/raw/filename.jpg'` 拼接

### 沙箱 file:// URI

```typescript
import fileUri from '@ohos.file.fileuri';

const sandboxPath = context.filesDir + '/raw/image_001.jpg';
const uri = fileUri.getUriFromPath(sandboxPath);
// => file:///data/storage/el2/base/haps/entry/files/raw/image_001.jpg
```

这种 URI 指向沙箱路径，可以传给 Image 组件渲染，也可以用来共享给其他应用。

---

## 3. 从相册 Picker 导入图片

### 完整导入流程

```typescript
import photoAccessHelper from '@ohos.file.photoAccessHelper';
import fs from '@ohos.file.fs';
import { common } from '@kit.AbilityKit';

async function importFromPicker(context: common.UIAbilityContext): Promise<string[]> {
  // 1. 打开相册 Picker（返回授权 URI 列表）
  const picker = new photoAccessHelper.PhotoViewPicker();
  const result = await picker.select({
    MIMEType: photoAccessHelper.PhotoViewMIMETypes.IMAGE_TYPE,
    maxSelectNumber: 9
  });
  
  const importedPaths: string[] = [];
  
  for (const uri of result.photoUris) {
    // 2. 使用授权 URI 打开源文件（必须用 fs.open，不能用 fs.openSync）
    const srcFile = await fs.open(uri, fs.OpenMode.READ_ONLY);
    
    // 3. 确定目标沙箱路径
    const filename = `img_${Date.now()}_${Math.random().toString(36).slice(2)}.jpg`;
    const destPath = context.filesDir + '/raw/' + filename;
    
    // 4. 复制到沙箱
    await fs.copyFile(srcFile.fd, destPath);
    await fs.close(srcFile.fd);
    
    importedPaths.push(destPath);
  }
  
  return importedPaths;
}
```

### 关键注意点

- `picker.select()` 返回的 URI 是**临时授权**的，仅在当前 UIAbility 生命周期内有效
- 必须在 UIAbility 的生命周期内完成复制，不要存储 media URI 留到后续再用
- `fs.open(uri)` 异步打开是必须的；`fs.openSync(mediaUri)` 会报错 `13900019`

---

## 4. 目录自愈工具函数

```typescript
import fs from '@ohos.file.fs';
import { BusinessError } from '@ohos.base';

/**
 * 确保目录存在。若不存在则创建（含父目录）。
 * 若路径存在但不是目录，则抛出错误。
 */
export function ensureDirectory(dirPath: string): void {
  try {
    const stat = fs.statSync(dirPath);
    if (!stat.isDirectory()) {
      throw new Error(`Path exists but is not a directory: ${dirPath}`);
    }
    // 已存在且是目录，无需处理
  } catch (e) {
    const err = e as BusinessError;
    if (err.code === 13900002) {
      // 不存在，创建（true = 递归创建父目录）
      fs.mkdirSync(dirPath, true);
    } else {
      throw e;
    }
  }
}

/**
 * 在应用启动时调用，确保所有必要子目录都存在。
 */
export function initSandboxDirectories(filesDir: string): void {
  ensureDirectory(filesDir + '/raw');
  ensureDirectory(filesDir + '/thumb');
  ensureDirectory(filesDir + '/export');
  ensureDirectory(filesDir + '/temp');
}
```

**为什么不能只用 `accessSync`**：

```typescript
// 错误写法 — accessSync 无法区分文件和目录
if (!fs.accessSync(dirPath)) {
  fs.mkdirSync(dirPath); // 若 dirPath 是文件，这里会报错
}

// 正确写法 — 使用 statSync().isDirectory() 判断
```

---

## 5. 文件删除与清理

### 删除单个文件

```typescript
import fs from '@ohos.file.fs';

function deleteFile(filePath: string): void {
  try {
    fs.unlinkSync(filePath);
  } catch (e) {
    const err = e as BusinessError;
    if (err.code !== 13900002) { // 已经不存在也不报错
      throw e;
    }
  }
}
```

### 清理 temp 目录（启动时调用）

```typescript
function clearTempDirectory(tempDir: string): void {
  try {
    const entries = fs.readdirSync(tempDir);
    for (const entry of entries) {
      try {
        fs.unlinkSync(tempDir + '/' + entry);
      } catch (_) {
        // 单个文件删除失败不影响其他
      }
    }
  } catch (e) {
    // temp 目录本身不存在，忽略
  }
}
```

---

## 6. 文件路径与 URI 转换

```typescript
import fileUri from '@ohos.file.fileuri';

// 沙箱路径 → file:// URI（用于 Image 组件、分享等）
const uri = fileUri.getUriFromPath('/data/storage/el2/base/haps/entry/files/raw/img.jpg');
// => file:///data/storage/el2/base/haps/entry/files/raw/img.jpg

// 在 ArkUI Image 组件中使用沙箱路径（可直接用路径，也可用 URI）
Image(context.filesDir + '/raw/img.jpg')
Image(fileUri.getUriFromPath(context.filesDir + '/raw/img.jpg'))
```

---

## 7. 常见错误码速查

| 错误码 | 含义 | 常见原因 | 处理方式 |
|---|---|---|---|
| `13900002` | No such file or directory | 目录未创建；路径拼写错误 | 检查目录是否存在，调用 `ensureDirectory` |
| `13900011` | File already exists | `mkdirSync` 目标已存在 | 先用 `statSync` 检查 |
| `13900019` | Operation not permitted | 用 `fs.openSync` 直接打开 media URI | 改用 `fs.open(uri)` 异步方式 |
| `13900020` | Invalid argument | fd 为 -1；路径为空或 null | 检查 open 是否成功，做 null 防护 |
| `13900025` | No space left on device | 设备存储空间不足 | 提示用户清理存储 |

---

## 8. 权限声明

需要在 `module.json5` 的 `requestPermissions` 中声明的权限：

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.READ_IMAGEVIDEO",
    "reason": "$string:permission_read_image",
    "usedScene": {
      "abilities": ["EntryAbility"],
      "when": "inuse"
    }
  },
  {
    "name": "ohos.permission.WRITE_IMAGEVIDEO",
    "reason": "$string:permission_write_image",
    "usedScene": {
      "abilities": ["EntryAbility"],
      "when": "inuse"
    }
  }
]
```

**注意**：使用 `PhotoViewPicker` 时**不需要**声明相册权限（Picker 本身处理了权限），只有在直接调用 `photoAccessHelper.getAssets()` 访问媒体库时才需要声明。

---

## 9. 完整导入→存储→显示链路

```typescript
// 1. 导入（相册 Picker）
const paths = await importFromPicker(context);

// 2. 存储路径到数据库（存 sandbox path，不存 media URI）
await db.insert('images', { path: paths[0], createdAt: Date.now() });

// 3. 显示（ArkUI Image 组件直接用 sandbox path）
Image(this.imagePath)
  .width('100%')
  .objectFit(ImageFit.Cover)
```

**永远不要把 media URI 存入数据库**：媒体 URI 的授权是临时的，下次启动后就失效了，会导致图片加载报错。
