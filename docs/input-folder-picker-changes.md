# Input Folder File Picker - 修改说明

## 概述

为节点的加载按钮添加从 ComfyUI input 文件夹选择文件的功能，替代浏览器文件选择器。原有上传/拖拽/粘贴机制全部保留。

## 改动文件

仅 JS 文件，无 Python 端改动：
- `js/multi_image_loader.js`
- `js/ltx_director.js`
- `js/load_video_ui.js`

## 使用的 API

均为 ComfyUI 官方公开接口：
- `/object_info/LoadImage` — 获取 input 文件夹图片文件列表
- `/object_info/LoadAudio` — 获取 input 文件夹音频/视频文件列表
- `/view?filename=xxx&type=input` — 获取文件（图片预览、视频播放、音频解码）
- `/ltx_director_get_audio?filename=xxx` — 从视频文件提取音频波形（仅 ltx_director）

### `/object_info` 返回格式差异

LoadImage 和 LoadAudio 的 JSON 结构不同，解析时需分别处理：

**LoadImage**: `{LoadImage: {input: {required: {image: [文件名数组, {image_upload: true}]}}}}`
- 文件名在 `data.LoadImage.input.required.image[0]`

**LoadAudio**: `{LoadAudio: {input: {required: {audio: ["COMBO", {options: 文件名数组, audio_upload: true}]}}}}`
- 文件名在 `data.LoadAudio.input.required.audio[1].options`
- 注意：`[0]` 是字符串 `"COMBO"`，不是数组

---

## 1. multi_image_loader.js

### 按钮文本（第 50 行）

```diff
- uploadBtn.innerText = "Upload Images";
+ uploadBtn.innerText = "Load Image";
```

### 新增 openImagePicker() 函数（插入在 uploadBtn.onclick 赋值之前）

约 180 行新代码。功能：
- 调用 `/object_info/LoadImage` 获取图片列表
- `extractImageListFromObjectInfo()` 解析返回 JSON（先取 `data.LoadImage || data` 再解析）
- 弹窗：48x48 缩略图 + 文件名 + 复选框，支持多选追加
- 搜索过滤、全选/取消全选

### 按钮点击行为

```diff
- uploadBtn.onclick = () => fileInput.click();
+ uploadBtn.onclick = () => openImagePicker();
```

### 未改动

`fileInput`、`handleFiles()`、拖拽上传、粘贴上传均保留。

---

## 2. ltx_director.js

### 按钮点击行为

```diff
- uploadBtn.addEventListener("click", () => this.fileInput.click());
+ uploadBtn.addEventListener("click", () => this.openImagePicker());

- uploadAudioBtn.addEventListener("click", () => this.audioFileInput.click());
+ uploadAudioBtn.addEventListener("click", () => this.openAudioPicker());

- uploadVideoBtn.addEventListener("click", () => this.videoFileInput.click());
+ uploadVideoBtn.addEventListener("click", () => this.openVideoPicker());

- uploadMotionBtn.addEventListener("click", () => this.motionFileInput.click());
+ uploadMotionBtn.addEventListener("click", () => this.openMotionPicker());
```

### 新增 8 个方法（插入在 handleImageUpload / handleVideoUpload 之前）

**图片：**
- `openImagePicker()` — 通过 `/object_info/LoadImage` 获取列表，弹窗带缩略图，多选 → `loadImagesByName()`
- `loadImagesByName(filenames)` — 跳过上传，直接用 `/view` URL 创建图片 segment（含 physics 推挤）

**音频：**
- `openAudioPicker()` — 通过 `/object_info/LoadAudio` 获取列表，弹窗无缩略图（音符图标），多选 → `loadAudiosByName()`
- `loadAudiosByName(filenames)` — 从 `/view` fetch 音频解码波形，创建音频 segment（含 waveform peaks、physics 推挤）

**视频（Add Video）：**
- `openVideoPicker()` — 视频选择弹窗，单选 → `loadVideosByName()`
- `loadVideosByName(filenames)` — 从 `/view` 加载视频，创建 video + audio segment 对，查询 `/ltx_director_get_audio` 获取波形

**运动视频（Add IC Video）：**
- `openMotionPicker()` — 运动视频选择弹窗，单选 → `loadMotionsByName()`
- `loadMotionsByName(filenames)` — 从 `/view` 加载视频，创建 `motion_video` segment（含 videoStrength、previewThumbs）

### 共享辅助方法

- `_fetchVideoFileList()` — 获取视频文件列表（video/motion picker 共用）
- `_showVideoPickerModal(title, onConfirm)` — 视频单选弹窗 UI（video/motion picker 共用）

### 未改动

`fileInput`、`audioFileInput`、`videoFileInput`、`motionFileInput`、`handleImageUpload()`、`handleAudioUpload()`、`handleVideoUpload()`、`handleMotionUpload()`、拖拽上传、粘贴上传均保留。

---

## 3. load_video_ui.js

### 新增按钮（在原有 "choose file to upload" 按钮之后）

```js
this.addWidget("button", "load video", null, () => { openVideoPicker(); });
```

### 新增 openVideoPicker() 函数（约 130 行）

- 通过 `/object_info/LoadAudio` 获取文件列表，过滤视频扩展名
- 单选弹窗（点击行高亮选中），搜索过滤
- 选中后设置 `videoWidget.value` 并更新预览

### 未改动

原有的 "choose file to upload" 按钮及全部上传逻辑（含分块上传）均保留。

---

## 上游合并注意事项

- 所有改动集中在 JS 文件，Python 端零改动，不会有 `.py` 合并冲突
- 新增函数均作为独立代码块插入，与周围逻辑无耦合
- ltx_director.js 中的冲突点通常在按钮 `addEventListener` 绑定处，合并时保留我们的 `openXxxPicker()` 调用，同时合入上游新增的按钮定义和 `this.xxxBtn` 引用
- 若上游修改了 `/object_info/LoadImage` 或 `/object_info/LoadAudio` 返回格式，需调整对应的解析函数
- 若上游重命名或移动了按钮定义，只需同步 `addEventListener` 绑定即可
