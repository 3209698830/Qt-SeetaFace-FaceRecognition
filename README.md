# 瞳记 · 人脸识别私密便签

基于 **Qt 5.14 + OpenCV 4.5.2 + SeetaFace2** 的 C/S 架构桌面应用。用户通过摄像头完成人脸注册与人脸登录，登录后可管理自己的私密便签，并可用人脸识别进行二次解锁；服务端负责统一存储用户特征与便签数据，并提供连接监控面板。

![Qt](https://img.shields.io/badge/Qt-5.14.0-green)
![OpenCV](https://img.shields.io/badge/OpenCV-4.5.2-blue)
![SeetaFace](https://img.shields.io/badge/SeetaFace-2.x-orange)
![Platform](https://img.shields.io/badge/Platform-Windows-lightgrey)

---

## 目录

- [功能特性](#功能特性)
- [项目结构](#项目结构)
- [环境要求](#环境要求)
- [快速开始](#快速开始)
- [通信协议](#通信协议)
- [数据库设计](#数据库设计)
- [注意事项与已知限制](#注意事项与已知限制)
- [第三方组件](#第三方组件)

---

## 功能特性

### 客户端（`project/`）

| 模块 | 说明 |
| --- | --- |
| 登录界面 `login` | 启动即连接服务端；提供「人脸登录」与「注册」入口 |
| 注册界面 `register_win` | 打开摄像头 → 抓拍照片 → 人脸检测与特征提取 → 上传服务端入库 |
| 人脸识别 `face_in` | 摄像头实时取帧，SeetaFace 提取特征后与已注册用户做 1:N 比对，比对成功即登录 |
| 主界面 `main_win` | 便签列表、新建/编辑/删除、关键字搜索、按日期筛选 |
| 隐私锁 | 一键给主界面盖上遮罩，点击解锁后进入人脸验证，只有当前用户本人才能解除遮罩 |

核心能力：

- **人脸注册**：姓名 + 实时抓拍，自动检测人脸并提取特征向量（维度由 `GetExtractFeatureSize()` 决定，以逗号分隔字符串存储）。
- **人脸登录**：默认阈值下的 1:N 检索，命中即跳转主界面。
- **私密便签**：每个账号的数据互相隔离，所有读写都带 `user_id` 校验。
- **二次解锁**：锁定模式下只接受「当前登录用户本人」的人脸，避免他人解锁。
- **日期检索**：按 `update_time` 前缀匹配某一天的便签。

### 服务端（`server/`）

- 基于 `QTcpServer` 的 JSON 命令服务，默认监听 `7777` 端口。
- 自动建表（`users` / `user_faces` / `notes`），数据落地到 SQLite。
- 实时显示在线客户端列表（IP、端口、状态）与操作日志。
- 统一处理注册、特征下发、便签增删改查与检索请求。

---

## 项目结构

```text
Qt-SeetaFace-FaceRecognition-main/
├── project/                          # 客户端（Qt Widgets 应用）
│   ├── main.cpp                      # 程序入口
│   ├── login.{h,cpp,ui}              # 登录界面 + 全局 TCP Socket / 同步请求工具
│   ├── register_win.{h,cpp,ui}       # 注册界面（摄像头抓拍 + 特征提取）
│   ├── face_in.{h,cpp,ui}            # 人脸识别界面（登录 / 解锁两种模式）
│   ├── main_win.{h,cpp,ui}           # 便签主界面
│   ├── image.qrc                     # 图标与背景资源
│   ├── image/                        # 图标、锁、保存、搜索等素材
│   └── project.pro
├── server/                           # 服务端（监控管理 + 数据服务）
│   ├── main.cpp
│   ├── admin_win.{h,cpp,ui}
│   └── server.pro
└── opencv-SeetaFace-qt5.14.0minGW/   # 预编译第三方库（MinGW 64-bit）
    ├── opencv452/                    # OpenCV 4.5.2（include / x64/mingw/lib / bin）
    └── SeetaFace/
        ├── include/                  # seeta/*.h
        ├── lib/                      # libSeetaFace*.a
        └── bin/                      # DLL + model/*.dat 模型文件
```

---

## 环境要求

| 项目 | 版本 |
| --- | --- |
| 操作系统 | Windows 10 / 11（64 位） |
| Qt | 5.14.0，**MinGW 64-bit** 套件（需包含 `widgets`、`multimedia`、`multimediawidgets`、`network`、`sql`） |
| OpenCV | 4.5.2（MinGW 预编译版，随仓库提供） |
| SeetaFace | SeetaFace2（随仓库提供 DLL、静态库与模型） |
| 编译器 | MinGW 7.3.0 64-bit（Qt 5.14.0 自带） |
| 其他 | 可用的 USB / 内置摄像头；服务端与客户端需处于同一局域网 |

> 工程使用 `.pro` + qmake 构建，直接用 Qt Creator 打开即可。

---

## 快速开始

### 1. 克隆 / 解压项目

将整个目录放到本地任意位置，例如：

```text
D:/four_project/Facial_recognition/
```

### 2. 修改硬编码的绝对路径（重要）

当前代码中所有依赖路径都写死为 `D:/four_project/Facial_recognition/...`。若你的目录不同，必须逐一修改：

| 文件 | 位置 | 需要修改的内容 |
| --- | --- | --- |
| `project/project.pro` | `INCLUDEPATH` / `LIBS` | OpenCV 与 SeetaFace 的头文件、库目录 |
| `project/register_win.cpp` | `init_seetaface_reg()` 中的 `model_dir` | 模型目录 `SeetaFace/bin/model/` |
| `project/register_win.cpp` | 抓拍保存路径 | `D:/four_project/Facial_recognition/take_picture/register/1.jpg` |
| `project/face_in.cpp` | `init_seetaface()` 中的 `model_dir` | 模型目录 |
| `project/face_in.cpp` | 构造函数中的 `setDatabaseName` | SQLite 文件路径（与服务端保持一致） |
| `project/login.cpp` | 构造函数 | 服务端 IP 与端口 |
| `server/admin_win.cpp` | 构造函数 | 监听 IP、端口 |
| `server/admin_win.cpp` | `initDatabase()` | SQLite 文件路径 |

推荐把 `.pro` 中的绝对路径改为相对路径，便于迁移：

```qmake
THIRDPARTY = $$PWD/../opencv-SeetaFace-qt5.14.0minGW

INCLUDEPATH += $$THIRDPARTY/opencv452/include \
               $$THIRDPARTY/SeetaFace/include

LIBS += -L$$THIRDPARTY/opencv452/x64/mingw/lib \
        -lopencv_core452 -lopencv_imgproc452 -lopencv_imgcodecs452 \
        -lopencv_objdetect452 -lopencv_highgui452 -lopencv_videoio452 \
        -L$$THIRDPARTY/SeetaFace/bin \
        -lSeetaFaceDetector -lSeetaFaceLandmarker -lSeetaFaceRecognizer \
        -lSeetaFaceTracker -lSeetaNet -lSeetaQualityAssessor
```

### 3. 准备模型文件

确认以下模型存在：

```text
opencv-SeetaFace-qt5.14.0minGW/SeetaFace/bin/model/
├── fd_2_00.dat          # 人脸检测
├── pd_2_00_pts5.dat     # 5 点关键点
├── pd_2_00_pts81.dat    # 81 点关键点
└── fr_2_10.dat          # 人脸特征提取
```

### 4. 准备运行时目录

在数据库与抓拍路径所在目录创建文件夹，例如：

```text
D:/four_project/Facial_recognition/
├── note_db.db                     # 首次运行由服务端自动创建
└── take_picture/register/         # 注册抓拍图片存放目录
```

### 5. 编译并运行

1. 用 Qt Creator 打开 `server/server.pro`，选择 **Desktop Qt 5.14.0 MinGW 64-bit**，构建并运行 → 服务端窗口出现，日志显示“服务器启动成功”。
2. 用 Qt Creator 打开 `project/project.pro`，同样选择 MinGW 64-bit 套件，构建并运行。
3. 将 `opencv-SeetaFace-qt5.14.0minGW/opencv452/x64/mingw/bin/*.dll` 与 `SeetaFace/bin/*.dll` 复制到客户端可执行文件所在目录（若未加入 `PATH`），否则会提示缺少 DLL。

### 6. 使用流程

```text
启动客户端
   └─ 注册：输入用户名 → 开启摄像头 → 抓拍 → 点击注册
   └─ 人脸登录：正对摄像头 → 识别成功进入主界面
          └─ 新建 / 编辑 / 删除便签
          └─ 关键字搜索、按日期筛选
          └─ 点击「锁定」→ 遮罩 → 点击解锁 → 人脸验证
```

---

## 通信协议

- 传输层：TCP，端口 `7777`
- 报文格式：单条紧凑 JSON（`QJsonDocument::toJson(Compact)`），服务端处理后回写同结构 JSON
- 客户端通过 `login::sendSyncRequest()` 发送并同步等待回包（超时 3 秒）

请求示例：

```json
{ "action": "register", "username": "alice", "features": "0.12,0.34,...", "create_time": "2026-04-07 10:00:00" }
```

支持的 `action`：

| action | 关键字段 | 响应字段 | 说明 |
| --- | --- | --- | --- |
| `register` | `username`、`features`、`create_time` | `status`、`msg` | 新建用户并写入人脸特征 |
| `load_features` | — | `status`、`users[]` | 下发全部用户的 `user_id` / `username` / `face_feature` |
| `load_notes` | `user_id` | `notes[]` | 按更新时间倒序返回便签列表 |
| `load_note_content` | `note_id`、`user_id` | `content` | 读取单条便签正文 |
| `save_note` | `is_new`、`user_id`、`note_id`、`title`、`content`、`time` | `status` | `is_new=1` 新增，否则更新 |
| `delete_note` | `note_id`、`user_id` | `status` | 删除便签 |
| `search_notes` | `user_id`、`keyword` | `notes[]` | 标题或正文模糊匹配 |
| `load_notes_by_date` | `user_id`、`date`（`yyyy-MM-dd`） | `notes[]` | 按日期前缀筛选 |

所有响应统一包含 `action_reply` 与 `status`（`success` / `error`）。

---

## 数据库设计

SQLite 文件默认：`d:/four_project/Facial_recognition/note_db.db`，由服务端首次启动时自动建表。

```sql
users (user_id INTEGER PRIMARY KEY AUTOINCREMENT,
       username TEXT UNIQUE,
       create_time TEXT)

user_faces (face_id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER,
            face_feature TEXT,        -- 逗号分隔的特征向量
            create_time TEXT,
            FOREIGN KEY(user_id) REFERENCES users(user_id))

notes (note_id INTEGER PRIMARY KEY AUTOINCREMENT,
       user_id INTEGER,
       title TEXT,
       content TEXT,
       create_time TEXT,
       update_time TEXT,
       FOREIGN KEY(user_id) REFERENCES users(user_id))
```

---

## 注意事项与已知限制

- **绝对路径**：模型、数据库、抓拍图片、服务端 IP 均硬编码，迁移环境前务必修改。
- **仅有人脸认证**：无密码 / 验证码兜底，注册时不校验重名以外的信息；特征阈值使用 SeetaFace 默认值。
- **明文传输**：JSON 通过裸 TCP 传输，特征与人脸数据未加密，仅适合课程设计 / 局域网演示。
- **同步阻塞请求**：`sendSyncRequest()` 使用 `waitForReadyRead`，等待期间 UI 会短暂卡顿。
- **无粘包处理**：服务端以 `readAll()` 直接解析，要求“一请求一回包”，高并发或大数据量需增加长度前缀协议。
- **单实例假设**：客户端使用静态 `QTcpSocket`，同一进程内共享连接。

---

## 第三方组件

| 组件 | 用途 | 许可 |
| --- | --- | --- |
| [Qt 5.14](https://www.qt.io/) | GUI、网络、多媒体、SQL | LGPLv3 / 商业许可 |
| [OpenCV 4.5.2](https://opencv.org/) | 图像处理、摄像头采集 | Apache-2.0 |
| [SeetaFace2](https://github.com/seetafaceengine/SeetaFace2) | 人脸检测、关键点、特征识别 | BSD-2-Clause |

仓库内 `opencv-SeetaFace-qt5.14.0minGW/` 仅为 MinGW 64-bit 预编译产物，版权归各上游项目所有。

---

如果这个项目对你有帮助，欢迎 Star ⭐
