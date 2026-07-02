# 本地曲库接入落地方案

> Phase 1 目标：**选个文件夹 → 自动扫描 → 看到所有歌 → 点任意一首播放**
>
> 纯本地，零联网，独立于登录态。

---

## 目录

1. [整体架构](#1-整体架构)
2. [数据模型](#2-数据模型)
3. [API 设计](#3-api-设计)
4. [UI 设计](#4-ui-设计)
5. [播放集成](#5-播放集成)
6. [歌词与封面处理](#6-歌词与封面处理)
7. [文件变更清单](#7-文件变更清单)
8. [排期建议](#8-排期建议)
9. [Phase 2 预留](#9-phase-2-预留)

---

## 1. 整体架构

```
┌──────────────────────────────────────────────────────────────────────┐
│  渲染进程 (index.html)                                                │
│                                                                      │
│  ┌────────────────────┐   ┌───────────────────────────────────────┐  │
│  │ 本地曲库页面        │   │ 播放器核心                            │  │
│  │ - 目录选择按钮       │   │ - 接收 stream URL 播放               │  │
│  │ - 扫描触发           │   │ - 加载 /lyric 到歌词解析器           │  │
│  │ - 列表渲染+搜索过滤   │   │ - 加载 /cover 到封面展示            │  │
│  │ - 点击播放           │   │                                      │  │
│  └────────┬────────────┘   └──────────┬────────────────────────────┘  │
│           │                           │                              │
└───────────┼───────────────────────────┼──────────────────────────────┘
            │ HTTP (fetch)              │ HTTP (audio.src)
            ▼                           ▼
┌──────────────────────────────────────────────────────────────────────┐
│  本地 HTTP 服务 (server.js)                                          │
│                                                                      │
│  POST /api/local-music/dir        设置扫描目录                        │
│  POST /api/local-music/scan       增量扫描 + ID3 解析                │
│  GET  /api/local-music/list       返回全量索引                       │
│  GET  /api/local-music/:id/stream Range 流式音频                     │
│  GET  /api/local-music/:id/cover  缓存封面图片                       │
│  GET  /api/local-music/:id/lyric  LRC 歌词文本                       │
│  PATCH /api/local-music/:id       更新字段（liked 等）               │
│                                                                      │
│  ┌─────────────────────────────────────┐                             │
│  │ 索引文件 local-music.index.json     │                             │
│  │ 封面缓存目录 local-covers/          │                             │
│  └─────────────────────────────────────┘                             │
└──────────────────────────────────────────────────────────────────────┘
            │
            ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Electron 主进程 (main.js + preload.js)                              │
│                                                                      │
│  IPC: select-music-directory → dialog.showOpenDialog({ properties:   │
│        ['openDirectory'] })                                          │
│  初始化: LOCAL_MUSIC_INDEX → userData/local-music.index.json         │
│          LOCAL_COVERS_DIR → userData/local-covers/                   │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 2. 数据模型

### 2.1 索引文件: `local-music.index.json`

存储在 `{userData}/local-music.index.json`，由 `LOCAL_MUSIC_INDEX` 环境变量指定。

```json
{
  "version": 1,
  "musicDir": "D:/Music",
  "files": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "filePath": "D:/Music/周杰伦/七里香.flac",
      "fileStat": {
        "size": 10485760,
        "mtime": 1690000000
      },
      "meta": {
        "title": "七里香",
        "artist": "周杰伦",
        "album": "七里香",
        "duration": 299.5
      },
      "hasCover": true,
      "coverPath": "550e8400-e29b-41d4-a716-446655440000.jpg",
      "hasLyric": true,
      "lyricFilePath": "D:/Music/周杰伦/七里香.lrc",
      "liked": false,
      "playCount": 0,
      "addedAt": 1700000000
    }
  ]
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | UUID v4，主键 |
| `filePath` | string | 文件绝对路径 |
| `fileStat.size` | number | 文件大小（增量扫描判据） |
| `fileStat.mtime` | number | 修改时间戳秒（增量扫描判据） |
| `meta.title` | string | ID3 标题，解析失败则用文件名 |
| `meta.artist` | string | ID3 艺术家 |
| `meta.album` | string | ID3 专辑 |
| `meta.duration` | number | 时长秒 |
| `hasCover` | boolean | 是否有内嵌封面 |
| `coverPath` | string | 封面缓存相对路径名 |
| `hasLyric` | boolean | 同级 lrc 是否存在 |
| `lyricFilePath` | string | lrc 文件绝对路径 |
| `liked` | boolean | 收藏状态（Phase 1 写入，Phase 2 UI 操作） |
| `playCount` | number | 播放计数（Phase 1 写入，Phase 2 统计用） |
| `addedAt` | number | 首次入库时间戳 |

### 2.2 封面缓存

- 目录: `{userData}/local-covers/`
- 命名: `{id}.jpg` 或 `{id}.png`（与音频原格式一致）
- 不做缩放（Phase 1 避免引入 sharp 等重型图片处理库）
- 删除歌曲时同步删除对应封面文件

---

## 3. API 设计

### 3.1 `POST /api/local-music/dir`

设置音乐目录，持久化到索引文件的 `musicDir` 字段。

**Request:**
```json
{ "dir": "D:/Music" }
```

**Response:**
```json
{ "ok": true, "dir": "D:/Music" }
```

**失败:**
```json
{ "ok": false, "error": "INVALID_DIR", "message": "目录不存在或不可访问" }
```

**验证逻辑:**
- `dir` 非空
- `fs.existsSync(dir)` 存在且为目录
- 存储 `path.resolve(dir)` 到 `localMusicIndex.musicDir`

---

### 3.2 `POST /api/local-music/scan`

增量扫描 `musicDir` 下的所有音频文件，解析 ID3，更新索引。

**扫描算法:**

```
1. 读取旧索引，建立 Map<filePath, entry>
2. 递归遍历 musicDir
   对于每个音频文件 (mp3/flac/wav/ogg/m4a/aac/wma/ape):
     a. 检查新旧: 如果旧条目存在且 mtime + size 未变 → 复用旧条目
     b. 新文件或已变更 → music-metadata.parseFile() 解析 ID3
     c. 提取 title/artist/album/duration
     d. 提取内嵌封面 → 写入 local-covers/{id}.jpg
     e. 检测同级 .lrc → 写入 hasLyric + lyricFilePath
     f. push 到 newFiles
3. deletedPaths = 旧索引中有但文件系统中已删除的 → 从 newFiles 删除
4. 清理已删除文件的封面
5. 写入磁盘
```

**Response:**
```json
{ "ok": true, "total": 128, "dir": "D:/Music" }
```

**同步扫描说明:**
- Phase 1 采用同步扫描（`fs.readdirSync` + `mm.parseFile` await），不引入 WebSocket/SSE 进度推送
- 500 首歌约 5-15 秒（取决于文件 IO 和 ID3 解析耗时）
- 前端显示 loading spinner + "扫描中…" 文字
- Phase 2 考虑大目录异步扫描 + 进度

---

### 3.3 `GET /api/local-music/list`

返回当前内存中的全量索引。

**Response:**
```json
{
  "ok": true,
  "dir": "D:/Music",
  "files": [ ... ]
}
```

**说明:**
- 全量返回，前端做搜索过滤和排序（本地应用数据量有限）
- 不做服务端分页

---

### 3.4 `GET /api/local-music/:id/stream`

流式返回音频文件，支持 HTTP Range 头部。

**请求头:**
```
Range: bytes=0-1023   (可选)
```

**响应 (无 Range):**
```
HTTP/1.1 200 OK
Content-Type: audio/flac
Content-Length: 10485760
Accept-Ranges: bytes

<binary data>
```

**响应 (带 Range):**
```
HTTP/1.1 206 Partial Content
Content-Type: audio/flac
Content-Range: bytes 0-1023/10485760
Content-Length: 1024
Accept-Ranges: bytes

<binary data>
```

**MIME 映射:**
| 扩展名 | Content-Type |
|--------|-------------|
| .mp3 | audio/mpeg |
| .flac | audio/flac |
| .wav | audio/wav |
| .ogg | audio/ogg |
| .m4a | audio/mp4 |
| .aac | audio/aac |
| .wma | audio/x-ms-wma |
| .ape | audio/ape |

**安全校验:**
- 验证 `id` 存在于索引中
- 验证 `filePath` 在 `musicDir` 之下（防路径穿越）
- 验证文件仍存在于磁盘

---

### 3.5 `GET /api/local-music/:id/cover`

返回已缓存的封面图片。

**Response (有封面):**
```
HTTP/1.1 200 OK
Content-Type: image/jpeg
Cache-Control: public, max-age=86400

<binary>
```

**Response (无封面):**
```
HTTP/1.1 404 Not Found
No cover
```

**性能控制:**
- Phase 1 不做图片缩放，直接输出原始二进制
- HTTP 缓存头部 `Cache-Control: public, max-age=86400`
- 前端 `<img>` 标签自动利用浏览器 HTTP 缓存

---

### 3.6 `GET /api/local-music/:id/lyric`

返回 LRC 歌词文本。

**Response:**
```
HTTP/1.1 200 OK
Content-Type: text/plain; charset=utf-8

[00:00.00] 七里香 - 周杰伦
[00:04.00] 窗外的麻雀 在电线杆上多嘴
[00:08.50] 你说这一句 很有夏天的感觉
```

**编码处理:**
- 先用 `buf.toString('utf8')` 读取
- 如果结果包含替换字符 `�`，退回到 `buf.toString('latin1')` 输出
- Phase 1 不做完整的 GBK 检测（不引入 iconv-lite 依赖）

**安全校验:**
- 验证 `lyricFilePath` 在 `musicDir` 之下（防路径穿越）
- 验证文件仍存在于磁盘

---

### 3.7 `PATCH /api/local-music/:id`

更新索引中指定歌曲的字段（Phase 2 用，Phase 1 写入但不暴露 UI）。

**Request:**
```json
{ "liked": true }
```

**Response:**
```json
{ "ok": true, "id": "550e8400-..." }
```

**支持的更新字段:**
- `liked` (boolean)

---

## 4. UI 设计

### 4.1 页面结构

本地曲库页面是一个**全屏覆盖层**，在主页上方打开，模糊背景。

```
┌────────────────────────────────────────────────────────────────┐
│ [本地曲库]  D:/Music                    [选择目录] [扫描] [关闭] │
├────────────────────────────────────────────────────────────────┤
│ [🔍 搜索本地歌曲…_____________________]          128 首        │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  [封面] 七里香                    周杰伦    七里香      4:59  │
│  [封面] 以父之名                  周杰伦    七里香      5:42  │
│  [封面] 晴天                      周杰伦    叶惠美      4:29  │
│  [封面] 浮夸                      陈奕迅    U-87       4:47   │
│  [封面] 富士山下                  陈奕迅    What's...  4:23  │
│  ...                                                          │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 4.2 交互流程

```
用户点击"本地曲库"
  ↓
打开覆盖层页面
  ↓
首次进入: 索引为空 → 显示引导文字 "点击'选择目录'添加你的音乐文件夹"
已有目录: 加载索引 → 显示歌曲列表
  ↓
用户点击"选择目录"
  ↓
Electron: dialog.showOpenDialog(openDirectory)
  ↓
POST /api/local-music/dir → 保存目录
  ↓
POST /api/local-music/scan → 扫描文件
  ↓
GET /api/local-music/list → 加载索引
  ↓
renderLocalMusicList() → 渲染列表
  ↓
用户搜索 → filterLocalMusic() → 前端过滤重渲染
  ↓
用户点击某首歌
  ↓
关闭覆盖层 → 播放 → 加载歌词 → 加载封面
```

### 4.3 搜索过滤

- 前端过滤，不请求服务端
- 匹配字段：`meta.title`、`meta.artist`、`meta.album`
- 大小写不敏感
- 输入即过滤（`oninput` 事件）
- 200ms debounce（可选）

### 4.4 列表项交互

| 交互 | 行为 |
|------|------|
| 点击歌曲 | 关闭页面 → 开始播放 |
| 当前播放歌曲 | 左侧高亮条 + 背景色 |
| 有歌词的歌曲 | 艺术家旁显示 "词" 徽标 |
| 无封面的歌曲 | 显示 "♪" 占位符 |
| 搜索框 | 实时过滤列表 |
| 关闭按钮 / 点击其他区域 | 关闭覆盖层，回到之前页面 |

---

## 5. 播放集成

### 5.1 播放路径

现有 `handleFiles()` 中本地文件的播放路径可以作为蓝本，但需要改为从索引获取歌曲信息而不是从文件 input。

**关键步骤:**

```
playLocalMusicItem(id):
  1. 从 localMusicData 中找到对应 entry
  2. 高亮列表中的该行
  3. 构建 stream URL: /api/local-music/{id}/stream
  4. 构建 cover URL: /api/local-music/{id}/cover
  5. 赋值 currentLocalSong = { type:'local', name, artist, cover, localKey, localUrl: streamUrl, ... }
  6. 重置播放状态 (trackSwitchToken++, beatMapToken++, resetAudioVisualState(), ...)
  7. 更新 UI: thumb-title, thumb-artist, controlTrackInfo, thumb-wrap
  8. 创建 new Audio(src=streamUrl) → play()
  9. 监听 loadedmetadata → 更新 duration display + prepareLocalBeatAnalysis()
  10. 监听 ended → 清理状态
  11. 如果 entry.hasLyric → fetch /api/local-music/{id}/lyric → parseLyricText → 设置歌词
```

### 5.2 与现有播放器的关系

| 方面 | 现有网易云/QQ 播放 | 本地曲库播放 |
|------|-------------------|-------------|
| 音频来源 | song_url API → 代理 | /api/local-music/:id/stream |
| 封面来源 | 网易/CDN 代理 URL | /api/local-music/:id/cover |
| 歌词来源 | API lyric_new | /api/local-music/:id/lyric |
| 队列管理 | playQueue + currentIdx | currentLocalSong (单曲) |
| 节拍分析 | prepareLocalBeatAnalysis | 复用同一函数 |

### 5.3 歌词加载时序

```
playLocalMusicItem()
  ↓ (并行执行)
fetch(stream) + fetch(lyric)
  ↓              ↓
Audio.play()    parseLyricText()
  ↓              ↓
播放音频         设置 lyricsLines
                lyricsVisible = true
                renderLyrics()
```

---

## 6. 歌词与封面处理

### 6.1 封面处理

| 步骤 | 说明 |
|------|------|
| 扫描时 | `music-metadata` 解析 `common.picture[0]` |
| 提取 | 二进制 buffer → `writeFileSync` 到 `local-covers/{id}.jpg` |
| 索引 | 写入 `hasCover: true` + `coverPath: "{id}.jpg"` |
| 前端 | `<img src="/api/local-music/{id}/cover">` |
| 无封面 | 默认 "♪" 占位符 |
| 缓存 | HTTP `Cache-Control: max-age=86400` |
| 清理 | 删除文件时同步删除 `local-covers/{id}.jpg` |

> Phase 1 不做: 文件夹 cover.jpg 扫描、在线封面下载、图片缩放/裁剪

### 6.2 歌词处理

| 步骤 | 说明 |
|------|------|
| 扫描时 | `fs.existsSync(lrcPath)` 检测同级 `.lrc` 文件 |
| 匹配规则 | 替换音频后缀为 `.lrc`，严格同级同目录同名 |
| 索引 | 写入 `hasLyric: true` + `lyricFilePath` |
| 读取 | `GET /api/local-music/{id}/lyric` → 返回 lrc 纯文本 |
| 编码 | UTF-8 优先，含 � 则退回到 latin1 |
| 前端 | `parseLyricText()` 解析 → 设置 `lyricsLines` → `renderLyrics()` |
| 缓存 | 不做歌词副本缓存，永远实时读取源文件 |

> Phase 1 不做: 内嵌 ID3 歌词块、krc/nrc 加密歌词、在线歌词下载、多歌词切换

---

## 7. 文件变更清单

### 7.1 `package.json`

| 变更 | 说明 |
|------|------|
| +dependency | `music-metadata` — ID3 标签解析 |
| +dependency | `uuid` — 生成歌曲条目 UUID |

两个依赖合计约 ~150KB，无原生编译依赖。

### 7.2 `server.js`

| 变更 | 说明 | 估行 |
|------|------|------|
| +require | `music-metadata`, `uuid` | 2 |
| +常量 | `LOCAL_MUSIC_INDEX`, `LOCAL_COVERS_DIR`, `SUPPORTED_AUDIO_EXT` | 10 |
| +工具函数 | `loadLocalMusicIndex()`, `saveLocalMusicIndex()`, `ensureCoversDir()`, `isPathSafe()`, `extToMime()` | 30 |
| +启动初始化 | 启动时加载索引 + 创建封面缓存目录 | 3 |
| +API: dir | `POST /api/local-music/dir` | 15 |
| +API: scan | `POST /api/local-music/scan` — 递归扫描 + ID3 解析 + 增量对比 + 封面缓存 + lrc 检测 | 80 |
| +API: list | `GET /api/local-music/list` | 4 |
| +API: stream | `GET /api/local-music/:id/stream` — Range 206 + 路径安全校验 | 40 |
| +API: cover | `GET /api/local-music/:id/cover` | 15 |
| +API: lyric | `GET /api/local-music/:id/lyric` — 编码兼容 | 20 |
| +API: patch | `PATCH /api/local-music/:id` | 15 |
| **小计** | | **~230** |

### 7.3 `desktop/main.js`

| 变更 | 说明 | 估行 |
|------|------|------|
| +IPC handler | `select-music-directory` — 调用 `dialog.showOpenDialog` 选目录 | 10 |
| +环境变量 | 设置 `LOCAL_MUSIC_INDEX` 和 `LOCAL_COVERS_DIR` 到 `userData` | 3 |
| **小计** | | **~13** |

### 7.4 `desktop/preload.js`

| 变更 | 说明 | 估行 |
|------|------|------|
| +bridge | `selectMusicDirectory: () => ipcRenderer.invoke('select-music-directory')` | 1 |
| **小计** | | **~1** |

### 7.5 `public/index.html`

| 变更 | 说明 | 估行 |
|------|------|------|
| +CSS | 本地曲库页面全部样式 (~70 行) + body 状态样式 (~3 行) | 75 |
| +HTML | 本地曲库页面结构（header + 搜索栏 + 列表） | 25 |
| +JS: 页面控制 | `openLocalMusicPage()`, `closeLocalMusicPage()` | 15 |
| +JS: 目录选择 | `selectMusicDir()` — IPC + API 调用 | 25 |
| +JS: 扫描 | `scanLocalMusic()` | 20 |
| +JS: 索引加载 | `loadLocalMusicIndex()` | 15 |
| +JS: 搜索过滤 | `filterLocalMusic()` | 15 |
| +JS: 列表渲染 | `renderLocalMusicList()` + `formatLocalDuration()` | 35 |
| +JS: 播放 | `playLocalMusicItem()` — 完整播放流程 | 65 |
| +JS: 工具 | `escHtml()` | 5 |
| +JS: 集成 | 修改原有 `openHomeLocalImport` 调用 → 改为 `openLocalMusicPage` | 1 |
| +JS: 集成 | `fallbackHomeTiles` 文字更新 | 1 |
| **小计** | | **~300** |

### 7.6 总量汇总

| 文件 | 新增行 | 备注 |
|------|--------|------|
| `package.json` | 2 行 + 2 依赖 | 仅新增，不改动现有 |
| `server.js` | ~230 | 新增代码块，不修改现有逻辑 |
| `desktop/main.js` | ~13 | 新增 IPC handler + 环境变量 |
| `desktop/preload.js` | ~1 | 新增一行 bridge |
| `public/index.html` | ~300 | CSS + HTML + JS，零散插入 |
| **总计** | **~550 行** | |

---

## 8. 排期建议

### 迭代 1: 服务端核心（半天）

| # | 任务 | 文件 |
|---|------|------|
| 1 | 安装依赖 `music-metadata` `uuid` | `package.json` |
| 2 | 新增常量 + 工具函数 + 索引读写 | `server.js` |
| 3 | 实现 `/api/local-music/dir` | `server.js` |
| 4 | 实现 `/api/local-music/scan`（递归遍历 + ID3 + 增量） | `server.js` |
| 5 | 实现 `/api/local-music/list` | `server.js` |
| 6 | 实现 `/api/local-music/:id/stream`（Range） | `server.js` |
| 7 | 实现 `/api/local-music/:id/cover` | `server.js` |
| 8 | 实现 `/api/local-music/:id/lyric` | `server.js` |
| 9 | 实现 `/api/local-music/:id` PATCH | `server.js` |

**验证方式:**
```bash
curl -X POST http://localhost:3000/api/local-music/dir -H 'Content-Type: application/json' -d '{"dir":"D:/Music"}'
curl -X POST http://localhost:3000/api/local-music/scan
curl http://localhost:3000/api/local-music/list | head -20
curl http://localhost:3000/api/local-music/{id}/stream > test.flac
```

### 迭代 2: 桌面端 IPC + 前端页面（半天）

| # | 任务 | 文件 |
|---|------|------|
| 1 | IPC `select-music-directory` + 环境变量 | `main.js` |
| 2 | preload bridge | `preload.js` |
| 3 | CSS 样式 | `index.html` |
| 4 | HTML 结构 | `index.html` |
| 5 | JS: 页面控制 + 目录选择 + 扫描 | `index.html` |
| 6 | JS: 列表渲染 + 搜索 | `index.html` |
| 7 | JS: 播放集成 + 歌词加载 | `index.html` |
| 8 | JS: 主页磁贴重定向 | `index.html` |

**验证方式:**
- 启动应用 → 点击"本地曲库" → 选目录 → 扫描 → 看到列表 → 点击播放

---

## 9. Phase 2 预留

以下功能 Phase 1 明确不做，但架构已预留扩展点：

| 功能 | 阻碍因素 | 所需改动 |
|------|---------|---------|
| 文件夹 cover.jpg 扫描 | 额外 IO + 大量边缘情况 | 扫描时增加目录级封面检测，新增 `folderCoverPath` 字段 |
| 在线封面/歌词下载 | 需要联网 + 网易云 API | 新增 Provider 接口，根据 title+artist 搜索并下载 |
| 大目录异步扫描 + 进度 | 需要 WebSocket/SSE | 改为异步扫描，通过 `/api/local-music/scan/status` 轮询或 SSE |
| 虚拟滚动（>5000 首） | 前端渲染性能 | 引入 IntersectionObserver + 只渲染可视区域 50 项 |
| 本地歌单管理 | 需要额外 UI + 数据模型 | 新增 `localMusicPlaylists` 状态和歌单 CRUD API |
| 多目录支持 | 增加模型复杂度 | `musicDir` 改为数组 `musicDirs: string[]` |
| 文件实时监听 | 需要 chokidar | 启动时注册文件监听，变更时自动增量扫描 |
| GBK 编码完美检测 | 需要 iconv-lite | 用 jschardet 检测编码或引入 iconv-lite |
| 图片缩放 | 需要 sharp/pngjs | 限制封面长边 500px 或返回缩略图 |
