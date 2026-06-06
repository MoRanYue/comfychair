# ComfyUI Android 客户端 - 图库本地持久化与增量同步方案

## 1. 背景分析

### 当前问题
- [`GalleryRepository`](app/src/main/java/sh/hnet/comfychair/repository/GalleryRepository.kt) 通过 [`ComfyUIClient.fetchAllHistory()`](app/src/main/java/sh/hnet/comfychair/ComfyUIClient.kt:881) 从服务器 `/history` 端点获取所有历史记录
- ComfyUI 服务器重启后，`/history` 端点返回空对象 `{}`，所有图库项目丢失
- 现有 [`GalleryMetadataCache`](app/src/main/java/sh/hnet/comfychair/storage/GalleryMetadataCache.kt) 仅保存元数据（无图像），且存储在应用内部 `filesDir`，而非用户指定的外部存储路径
- 现有 [`MediaCache`](app/src/main/java/sh/hnet/comfychair/cache/MediaCache.kt) 中的磁盘缓存位于 `cacheDir`，可能被系统清理，且不是持久化存储

### 目标
1. **持久化存储**：将历史记录元数据和图像保存到 `/sdcard/Android/data/sh.hnet.comfychair/gallery/` 的 `metadata/` 和 `images/` 子目录
2. **增量同步**：每次刷新时，仅从服务器获取新的或变更的项目，避免重复下载
3. **服务器重启容错**：服务器重启后，图库仍可展示本地已缓存的内容

## 2. 架构设计

```mermaid
flowchart TD
    subgraph "新增组件"
        LS[LocalGalleryStorage\n本地图库存储]
    end

    subgraph "修改组件"
        GR[GalleryRepository\n图库仓库]
        MC[MediaCache\n媒体缓存]
        CC[ComfyUIClient\nAPI客户端]
        GI[GalleryItem\n数据模型]
    end

    subgraph "存储目录结构"
        META[gallery/metadata/index.json\n项目索引]
        META2[gallery/metadata/{promptId}.json\n历史记录JSON]
        IMG[gallery/images/{filename}\n图像/视频文件]
    end

    GR -->|读取/写入| LS
    LS --> META
    LS --> META2
    LS --> IMG
    GR -->|增量获取| CC
    MC -->|优先检查| IMG
```

### 存储目录结构

```
/sdcard/Android/data/sh.hnet.comfychair/gallery/
├── metadata/
│   ├── index.json              # 所有图库项目的索引（快速加载列表）
│   ├── {promptId_1}.json       # 每个 prompt 的完整历史记录 JSON
│   ├── {promptId_2}.json
│   └── ...
└── images/
    ├── {filename_1}.png        # 图像/视频文件（直接以原始文件名存储）
    ├── {filename_2}.webp
    └── ...
```

## 3. 详细设计

### 3.1 新增文件：`LocalGalleryStorage`

**路径**: [`app/src/main/java/sh/hnet/comfychair/storage/LocalGalleryStorage.kt`](app/src/main/java/sh/hnet/comfychair/storage/LocalGalleryStorage.kt)

**职责**:
- 管理 `gallery/metadata/` 和 `gallery/images/` 目录的读写
- 提供索引文件的序列化/反序列化
- 提供单条元数据的增删改查
- 提供图像/视频文件的增删查

**核心 API**:

| 方法 | 描述 |
|------|------|
| `getGalleryDir(context)` | 返回 `getExternalFilesDir("gallery")` |
| `getMetadataDir(context)` | 返回 `gallery/metadata/` |
| `getImagesDir(context)` | 返回 `gallery/images/` |
| `loadIndex(context)` | 读取 `index.json`，返回 `List<GalleryItem>` |
| `saveIndex(context, items)` | 写入 `index.json` |
| `loadHistory(context, promptId)` | 读取 `{promptId}.json`，返回历史记录 JSON |
| `saveHistory(context, promptId, json)` | 写入 `{promptId}.json` |
| `deleteHistory(context, promptId)` | 删除 `{promptId}.json` |
| `imageFileExists(context, filename)` | 检查本地是否存在图像文件 |
| `saveImage(context, filename, bytes)` | 保存图像到 `images/` |
| `loadImageBytes(context, filename)` | 从 `images/` 读取图像字节 |
| `deleteImage(context, filename)` | 从 `images/` 删除图像 |
| `getAllLocalPromptIds(context)` | 获取所有本地已缓存的 prompt ID 集合 |
| `clearAll(context)` | 清空整个 gallery 目录 |

**索引文件格式** (`index.json`):
```json
{
  "version": 1,
  "lastSyncTimestamp": 1717000000000,
  "items": [
    {
      "promptId": "abc-123",
      "filename": "ComfyUI_00001_.png",
      "subfolder": "",
      "type": "output",
      "isVideo": false,
      "timestamp": 1716900000000
    }
  ]
}
```

### 3.2 修改 `GalleryItem` 数据类

**路径**: [`app/src/main/java/sh/hnet/comfychair/viewmodel/GalleryViewModel.kt`](app/src/main/java/sh/hnet/comfychair/viewmodel/GalleryViewModel.kt:33)

**变更**:
- 新增 `timestamp: Long = 0` 字段，用于排序和增量判断
- 新增 `localCacheExists: Boolean = false` 字段，指示本地是否已缓存图像

```kotlin
data class GalleryItem(
    val promptId: String,
    val filename: String,
    val subfolder: String,
    val type: String,
    val isVideo: Boolean,
    val index: Int = 0,
    val timestamp: Long = 0,       // 新增：Unix 毫秒时间戳
    val localCacheExists: Boolean = false  // 新增：本地图像是否已缓存
)
```

### 3.3 修改 `ComfyUIClient`

**路径**: [`app/src/main/java/sh/hnet/comfychair/ComfyUIClient.kt`](app/src/main/java/sh/hnet/comfychair/ComfyUIClient.kt)

**变更**:
- 新增 `fetchSpecificHistory(promptIds: Set<String>, callback)` 方法，批量获取指定 prompt ID 的历史记录
- 复用现有的 `/history/{prompt_id}` 端点逻辑

```kotlin
fun fetchSpecificHistory(
    promptIds: Set<String>,
    callback: (historyMap: Map<String, JSONObject>?) -> Unit
) {
    // 并发或串行调用 /history/{prompt_id} 获取多个 prompt 的历史
    // 返回 Map<promptId, historyJson>
}
```

### 3.4 重写 `GalleryRepository` 核心逻辑

**路径**: [`app/src/main/java/sh/hnet/comfychair/repository/GalleryRepository.kt`](app/src/main/java/sh/hnet/comfychair/repository/GalleryRepository.kt)

#### 增量同步流程

```mermaid
flowchart TD
    A[开始刷新] --> B[加载本地索引\nindex.json]
    B --> C[获取本地 prompt ID 集合]
    C --> D{在线模式?}
    D -->|是| E[调用 /history\n获取所有服务器 prompt ID]
    D -->|否| F[直接使用本地数据]
    
    E --> G[找出新增项\n服务器有但本地无]
    G --> H[对于每个新增项:]
    H --> H1[调用 /history/{id}\n获取完整历史JSON]
    H1 --> H2[保存元数据到\nmetadata/{id}.json]
    H2 --> H3[下载图像/视频\n保存到 images/]
    H3 --> H4[更新索引文件]
    H4 --> I
    
    E --> J[找出已删除项\n本地有但服务器无]
    J --> K{服务器重启?}
    K -->|是| L[保留本地数据\n不移除]
    K -->|否| M[从本地存储删除]
    
    L --> I
    M --> I
    
    I[合并本地+新增数据] --> N[更新 GalleryItems StateFlow]
    N --> O[更新索引文件\ntimestamp]
```

**关键变更**:

1. **`loadGalleryInternal()`** 重写:
   - 始终从 `LocalGalleryStorage.loadIndex()` 加载本地数据
   - 在线时，调用 `/history` 获取服务器数据
   - 遍历服务器数据，仅对本地不存在的项目执行完整下载
   - 对服务器已删除但本地存在的项目：**保留**（因为服务器可能已重启）
   - 合并结果并显示

2. **`loadFromOfflineCache()`** 重写:
   - 改为从 `LocalGalleryStorage` 加载
   - 不再依赖旧的 `GalleryMetadataCache`

3. **`deleteItem()` 和 `deleteItems()`** 更新:
   - 除了删除服务器端，还要删除本地 `metadata/{id}.json` 和 `images/{filename}`

4. **`clearCache()` 和 `reset()`** 更新:
   - 可选择是否清理本地 gallery 存储

### 3.5 修改 `MediaCache`

**路径**: [`app/src/main/java/sh/hnet/comfychair/cache/MediaCache.kt`](app/src/main/java/sh/hnet/comfychair/cache/MediaCache.kt)

**变更**:
- 在 `fetchBitmap()`, `fetchImage()`, `fetchVideoBytes()` 等方法中，在线程取服务器之前，首先检查 `LocalGalleryStorage` 中是否存在对应的本地文件
- 如果本地文件存在，直接读取并返回（配合已有的缓存策略）
- 下载新图像时，同时保存到 `LocalGalleryStorage`

```kotlin
// 在 fetchImage 等方法开头添加：
// 1. 检查 LocalGalleryStorage 中是否存在本地文件
// 2. 如果存在，读取并缓存到内存/磁盘缓存
// 3. 如果不存在，从服务器获取
```

### 3.6 其他文件修改

**`AndroidManifest.xml`**:
- `minSdk = 33`，`getExternalFilesDir()` 无需额外权限
- 无需添加新权限

**`GalleryMetadataCache.kt`**:
- 可保留用于向后兼容，但核心逻辑迁移到 `LocalGalleryStorage`
- 或在迁移完成后废弃此文件

## 4. 实施步骤

### 步骤 1：创建 `LocalGalleryStorage` 类
- 实现 `gallery/metadata/` 和 `gallery/images/` 目录管理
- 实现索引文件读写
- 实现单条元数据文件读写
- 实现图像文件存储和读取

### 步骤 2：更新 `GalleryItem` 数据类
- 添加 `timestamp` 和 `localCacheExists` 字段
- 更新相关序列化逻辑

### 步骤 3：在 `ComfyUIClient` 中添加增量获取方法
- 添加 `fetchSpecificHistory()` 方法
- 支持批量获取指定 prompt ID 的历史

### 步骤 4：重写 `GalleryRepository.syncFromServer()` 核心逻辑
- 实现增量同步算法
- 处理新增项的元数据和图像下载
- 处理已删除项的清理（保留服务器重启场景）

### 步骤 5：修改 `MediaCache` 优先检查本地存储
- 在获取图像/视频前先查 `LocalGalleryStorage`
- 下载新图像时自动保存到本地

### 步骤 6：更新删除/清理逻辑
- 删除操作同步清理本地存储文件
- 清理缓存时区分临时缓存和持久存储

### 步骤 7：测试与验证
- 验证增量同步：服务器新增项目后，客户端只下载新增项
- 验证持久化：服务器重启后，图库仍显示已缓存项目
- 验证图像离线可用：断网后仍可查看已缓存的图像
- 验证删除：删除后本地和服务器端均移除

## 5. 注意事项

1. **文件命名冲突**：`images/` 目录中不同 prompt 可能生成同名文件（如 `ComfyUI_00001_.png`），但 ComfyUI 保证同一 prompt 内文件名唯一。不同 prompt 的同名文件会被后续写入覆盖。解决方案：在 `index.json` 中建立 promptId 到文件名的映射，保存时如果文件已存在但属于不同 prompt，需要重命名。但这种情况概率很低，暂不作特殊处理。

2. **存储空间**：图像文件可能占用大量空间。用户可通过设置清除缓存。

3. **并发安全**：`LocalGalleryStorage` 应使用文件锁或同步块保证并发安全。

4. **性能**：索引文件应控制在合理大小（数千条记录），如果过大可考虑分页。

5. **迁移**：现有 `GalleryMetadataCache` 中已缓存的数据不会自动迁移到新存储，用户重新连接服务器后会通过增量同步重新下载。
