---
name: harmonyos-file-sandbox
description: This skill should be used when working with HarmonyOS file system operations, media URIs, sandbox paths, or file copy/access errors. Covers the distinction between media URIs and sandbox paths, correct usage of @ohos.file.fs APIs (openSync, copyFileSync, mkdirSync), Picker URI temporary authorization, common error codes (13900002, 13900011), and directory self-healing patterns. Trigger phrases include "文件操作", "沙箱", "media URI", "fs.openSync", "No such file", "13900002", "相册", "picker", "文件路径", "复制文件", "图片导入".
---

# HarmonyOS 本地文件沙箱 Skill

## 适用场景

This skill is relevant when:
- Copying images from media library (photo picker) into app sandbox
- Accessing or creating files/directories in the app sandbox
- Diagnosing `13900002 No such file or directory` or similar fs errors
- Implementing thumbnail generation, export, or temp file management
- Working with `@ohos.file.fs`, `fileUri`, or `photoAccessHelper`

Read `references/file_sandbox_guide.md` for the detailed API reference and code patterns.

## Key Concepts (Quick Reference)

### Media URI vs Sandbox Path

| Type | Example | Can `fs.openSync` use directly? |
|---|---|---|
| Media URI | `file://media/Photo/1/IMG_xxx.jpg` | NO — requires authorized access |
| Sandbox path | `/data/storage/el2/base/haps/entry/files/raw/xxx.jpg` | YES |
| `fileUri.getUriFromPath(sandboxPath)` | `file:///data/storage/el2/base/...` | YES (same underlying path) |

**Rule**: Never pass a raw media URI to `fs.openSync`. Always use authorized copy flow first.

### Picker URI Authorization Flow

```typescript
// 1. Launch picker — returns authorized URIs (temporary)
const result = await photoAccessHelper.PhotoViewPicker().select(options);
const uris: string[] = result.photoUris; // authorized media URIs

// 2. Open source file using authorized URI (NOT fs.openSync directly)
const srcFile = await fs.open(uri, fs.OpenMode.READ_ONLY);

// 3. Copy to sandbox destination using fs.copyFile (fd → path)
const destPath = context.filesDir + '/raw/' + filename;
await fs.copyFile(srcFile.fd, destPath);
await fs.close(srcFile.fd);
```

### Directory Self-Healing Pattern

Do NOT use `accessSync` alone to check if a directory exists before creating it — it cannot distinguish file vs directory:

```typescript
// CORRECT defensive pattern
function ensureDirectory(dirPath: string): void {
  try {
    const stat = fs.statSync(dirPath);
    if (!stat.isDirectory()) {
      throw new Error(`Path exists but is not a directory: ${dirPath}`);
    }
    // directory already exists, nothing to do
  } catch (e) {
    if ((e as BusinessError).code === 13900002) {
      // does not exist — create it
      fs.mkdirSync(dirPath, true); // true = create parent dirs
    } else {
      throw e; // unexpected error
    }
  }
}
```

### Sandbox Directory Layout (Recommended)

```
context.filesDir/
├── raw/       # original images copied from picker
├── thumb/     # generated thumbnails
├── export/    # user-exported files (PDF, zip, etc.)
└── temp/      # temporary processing files (cleared on app start)
```

Always call `ensureDirectory()` for each subdirectory at app startup or before first use.

### Common Error Codes

| Code | Meaning | Fix |
|---|---|---|
| `13900002` | No such file or directory | Directory not created yet; use `ensureDirectory()` |
| `13900011` | File already exists | Use overwrite flag or check before creating |
| `13900019` | Operation not permitted | Trying to write to read-only media URI path |
| `13900020` | Invalid argument | Wrong file descriptor or null path |

### `fs.openSync` vs `fs.open` (Async)

- In API 12+, prefer `fs.open()` (async) for I/O in Service/Worker contexts
- In UI thread short operations, `fs.openSync` is acceptable but may throw `13900002` if dirs don't exist
- Always close file descriptors: `fs.closeSync(fd)` or `fs.close(fd)`

### Getting the App Sandbox Root

```typescript
import { common } from '@kit.AbilityKit';

const context = getContext(this) as common.UIAbilityContext;
const filesDir = context.filesDir;       // /data/storage/el2/base/haps/entry/files
const cacheDir = context.cacheDir;       // /data/storage/el2/base/haps/entry/cache
const tempDir  = context.tempDir;        // /data/storage/el2/base/haps/entry/temp
```
