# 模块七：产物系统与屏幕录制详细设计

> 本文档详细描述产物（Artifacts）的生命周期管理、屏幕录制机制、视频处理管线、产物上传与展示。

---

## 1. 产物系统架构

### 1.1 整体架构

```
┌─────────────────────────────────────────────────────────┐
│                   Artifacts Pipeline                     │
│                                                         │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐           │
│  │ Creator  │──▶│ Storage  │──▶│ Uploader │──▶ CDN/S3  │
│  │ (Agent)  │   │ (Local)  │   │ (Async)  │           │
│  └──────────┘   └──────────┘   └──────────┘           │
│       │                                                 │
│  ┌────▼─────┐   ┌──────────┐   ┌──────────┐           │
│  │ Screen   │──▶│ Encoder  │──▶│ Post-    │           │
│  │ Recorder │   │ (ffmpeg) │   │ Process  │           │
│  └──────────┘   └──────────┘   └──────────┘           │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│                    Web UI Renderer                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────────┐  │
│  │ Image    │  │ Video    │  │ Text/Log             │  │
│  │ Viewer   │  │ Player   │  │ Viewer               │  │
│  └──────────┘  └──────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### 1.2 本地存储布局

```
/opt/cursor/artifacts/
├── screenshot_login_page.png          # Agent 截图
├── screenshot_dashboard_after.webp    # Agent 截图
├── demo_submit_button.mp4            # 屏幕录制（原始）
├── demo_submit_button_demo.mp4       # 屏幕录制（demo 版本）
├── build_output.log                  # 构建日志
├── test_results.log                  # 测试结果
└── error_trace.txt                   # 错误追踪
```

---

## 2. 产物创建

### 2.1 产物类型与创建方式

| 产物类型 | 创建方式 | 文件格式 |
|---------|---------|---------|
| 截图 | computerUse 子代理自动截取 | png, webp, jpg |
| 屏幕录制 | RecordScreen 工具控制 | mp4, webm |
| 日志文件 | Agent 手动写入 | log, txt |
| 任意文件 | Agent 复制到 artifacts 目录 | * |

### 2.2 截图产物

```python
class ScreenshotManager:
    """截图管理"""

    ARTIFACTS_DIR = "/opt/cursor/artifacts"
    MAX_SCREENSHOTS = 50         # 单会话最大截图数
    IMAGE_QUALITY = 85           # WebP 质量
    MAX_RESOLUTION = (1920, 1080)

    def capture(self, filename: str = None) -> str:
        """截取当前屏幕"""
        if not filename:
            filename = f"screenshot_{int(time.time())}.png"

        filepath = os.path.join(self.ARTIFACTS_DIR, filename)

        # 使用不同方式截图
        if self._has_display():
            # X11 环境
            subprocess.run([
                "import", "-window", "root", filepath
            ])
        else:
            # 虚拟帧缓冲
            subprocess.run([
                "xdotool", "getactivewindow", "screenshot", filepath
            ])

        return filepath

    def optimize(self, filepath: str) -> str:
        """优化截图文件大小"""
        if filepath.endswith(".png"):
            # 转换为 WebP 减小文件
            webp_path = filepath.rsplit(".", 1)[0] + ".webp"
            subprocess.run([
                "cwebp", "-q", str(self.IMAGE_QUALITY), filepath, "-o", webp_path
            ])
            os.remove(filepath)
            return webp_path
        return filepath
```

### 2.3 日志产物

```python
class LogArtifact:
    """日志产物创建"""

    @staticmethod
    def save_command_output(
        name: str,
        command: str,
        output: str,
    ) -> str:
        """保存命令输出为日志产物"""
        filepath = os.path.join("/opt/cursor/artifacts", f"{name}.log")

        content = f"""# Command: {command}
# Timestamp: {datetime.now().isoformat()}
# Exit Code: {exit_code}

{output}
"""
        with open(filepath, "w") as f:
            f.write(content)

        return filepath
```

---

## 3. 屏幕录制系统

### 3.1 录制引擎

```python
class ScreenRecorder:
    """屏幕录制引擎"""

    ARTIFACTS_DIR = "/opt/cursor/artifacts"
    TEMP_DIR = "/tmp/recordings"

    # 录制参数
    FRAMERATE = 30                  # 帧率
    VIDEO_SIZE = "1920x1080"        # 分辨率
    CODEC = "libx264"               # 编码器
    PRESET = "ultrafast"            # 编码速度（录制时用最快）
    CRF = 23                        # 质量（0=无损, 51=最差）
    PIXEL_FORMAT = "yuv420p"        # 像素格式（兼容性最好）

    def __init__(self):
        self.process = None
        self.temp_file = None
        self.recording = False
        self.start_time = None
        os.makedirs(self.TEMP_DIR, exist_ok=True)

    def start(self) -> str:
        """开始录制"""
        if self.recording:
            return "Error: Already recording"

        self.temp_file = os.path.join(
            self.TEMP_DIR,
            f"rec_{int(time.time())}.mp4"
        )

        # ffmpeg 录制命令
        cmd = [
            "ffmpeg",
            "-y",                           # 覆写输出
            "-f", "x11grab",                # X11 抓取
            "-video_size", self.VIDEO_SIZE,
            "-framerate", str(self.FRAMERATE),
            "-i", os.environ.get("DISPLAY", ":0.0"),
            "-codec:v", self.CODEC,
            "-preset", self.PRESET,
            "-crf", str(self.CRF),
            "-pix_fmt", self.PIXEL_FORMAT,
            self.temp_file,
        ]

        self.process = subprocess.Popen(
            cmd,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
        )

        self.recording = True
        self.start_time = time.time()

        return "Screen recording started"

    def save(self, filename: str = None) -> str:
        """停止录制并保存"""
        if not self.recording:
            return "Error: Not recording"

        # 发送 'q' 信号优雅停止 ffmpeg
        self.process.stdin.write(b"q")
        self.process.wait(timeout=10)
        self.recording = False

        duration = time.time() - self.start_time

        # 确定输出文件名
        if filename:
            safe_name = self._sanitize_filename(filename)
            output_name = f"{safe_name}.mp4"
        else:
            output_name = f"recording_{int(time.time())}.mp4"

        output_path = os.path.join(self.ARTIFACTS_DIR, output_name)

        # 后处理
        self._post_process(self.temp_file, output_path)

        # 生成 demo 版本
        demo_path = output_path.replace(".mp4", "_demo.mp4")
        self._create_demo_version(output_path, demo_path)

        # 清理临时文件
        os.remove(self.temp_file)

        return f"Recording saved: {output_path} (duration: {duration:.1f}s)"

    def discard(self) -> str:
        """停止录制并丢弃"""
        if not self.recording:
            return "Error: Not recording"

        self.process.terminate()
        self.process.wait(timeout=5)
        self.recording = False

        if os.path.exists(self.temp_file):
            os.remove(self.temp_file)

        return "Recording discarded"

    def _post_process(self, input_path: str, output_path: str):
        """后处理：压缩优化"""
        subprocess.run([
            "ffmpeg",
            "-y",
            "-i", input_path,
            "-codec:v", "libx264",
            "-preset", "medium",           # 更好的压缩
            "-crf", "28",                  # 稍微降低质量换取小文件
            "-pix_fmt", "yuv420p",
            "-movflags", "+faststart",     # Web 友好（moov atom 前置）
            output_path,
        ], check=True)

    def _create_demo_version(self, input_path: str, output_path: str):
        """创建 demo 版本（可能裁剪/加速）"""
        # demo 版本用于 videoReview 子代理分析
        subprocess.run([
            "ffmpeg",
            "-y",
            "-i", input_path,
            "-vf", "scale=1280:720",       # 降低分辨率
            "-codec:v", "libx264",
            "-preset", "fast",
            "-crf", "30",
            output_path,
        ], check=True)

    def _sanitize_filename(self, name: str) -> str:
        """清理文件名"""
        import re
        name = re.sub(r'[^\w\-_]', '_', name)
        return name[:100]  # 限制长度
```

### 3.2 录制状态机

```python
class RecordingStateMachine:
    """录制状态管理"""

    states = ["idle", "recording", "saving", "saved", "discarded"]

    transitions = {
        "idle":       {"START_RECORDING": "recording"},
        "recording":  {"SAVE_RECORDING": "saving", "DISCARD_RECORDING": "discarded"},
        "saving":     {"complete": "saved", "error": "idle"},
        "saved":      {},  # 终态
        "discarded":  {},  # 终态
    }

    def transition(self, action: str) -> str:
        """执行状态转换"""
        valid_actions = self.transitions.get(self.current_state, {})
        if action not in valid_actions:
            raise InvalidStateError(
                f"Cannot {action} in state {self.current_state}"
            )
        self.current_state = valid_actions[action]
        return self.current_state
```

---

## 4. 产物上传管线

### 4.1 上传流程

```
Agent 会话结束
    │
    ▼
┌──────────────────┐
│ 扫描 artifacts/  │  列出所有文件
└────────┬─────────┘
         ▼
┌──────────────────┐
│ 文件过滤与验证    │  检查大小/类型/完整性
└────────┬─────────┘
         ▼
┌──────────────────┐
│ 并行上传到 S3     │  分块上传大文件
└────────┬─────────┘
         ▼
┌──────────────────┐
│ 生成访问 URL      │  CDN 预签名 URL
└────────┬─────────┘
         ▼
┌──────────────────┐
│ 更新会话元数据    │  产物列表 + URL
└──────────────────┘
```

### 4.2 上传实现

```python
class ArtifactUploader:
    """产物上传服务"""

    MAX_FILE_SIZE = 500_000_000   # 500MB 单文件上限
    MAX_TOTAL_SIZE = 2_000_000_000  # 2GB 总量上限
    CHUNK_SIZE = 8_000_000         # 8MB 分块大小

    def upload_all(self, session_id: str) -> List[ArtifactMeta]:
        """上传所有产物"""
        artifacts = []

        for filepath in Path("/opt/cursor/artifacts").iterdir():
            if not filepath.is_file():
                continue

            size = filepath.stat().st_size
            if size > self.MAX_FILE_SIZE:
                continue  # 跳过过大文件

            meta = self.upload_file(session_id, filepath)
            artifacts.append(meta)

        return artifacts

    def upload_file(self, session_id: str, filepath: Path) -> ArtifactMeta:
        """上传单个文件"""
        key = f"artifacts/{session_id}/{filepath.name}"

        if filepath.stat().st_size > self.CHUNK_SIZE:
            # 分块上传
            self._multipart_upload(filepath, key)
        else:
            # 直接上传
            self.s3.upload_file(str(filepath), self.bucket, key)

        url = self.cdn.get_url(key)

        return ArtifactMeta(
            filename=filepath.name,
            size=filepath.stat().st_size,
            mime_type=self._get_mime_type(filepath),
            url=url,
            uploaded_at=datetime.utcnow(),
        )
```

---

## 5. 产物不可变性

### 5.1 不可变设计

```
规则: 产物一旦上传即不可修改或删除

原因:
├── 审计追踪: 保留完整的工作证据
├── 引用完整性: 最终回复中的引用始终有效
└── 防篡改: 确保展示给用户的证据真实

实现:
├── 文件系统层: artifacts/ 目录写入后不可覆写
├── 存储层: S3 对象锁定 (Object Lock)
└── API 层: 不提供 delete/update 端点
```

### 5.2 命名冲突处理

```python
def get_unique_filename(directory: str, desired_name: str) -> str:
    """确保文件名唯一（不可覆写）"""
    filepath = os.path.join(directory, desired_name)
    if not os.path.exists(filepath):
        return filepath

    # 添加数字后缀
    base, ext = os.path.splitext(desired_name)
    counter = 1
    while os.path.exists(filepath):
        filepath = os.path.join(directory, f"{base}_{counter}{ext}")
        counter += 1

    return filepath
```

---

## 6. UI 产物渲染

### 6.1 渲染格式映射

| Agent 输出 | UI 渲染 |
|-----------|---------|
| `<img src="path" alt="desc" />` | 图片预览器（可点击放大） |
| `<video src="path" controls></video>` | 视频播放器（带控制条） |
| `<TextReference path="path" start={s} end={e} alt="desc">` | 代码/日志查看器（带行号高亮） |
| 内联文件路径 | 可点击下载链接 |

### 6.2 图片预览器

```tsx
interface ImageViewerProps {
  src: string;
  alt: string;
}

function ImageViewer({ src, alt }: ImageViewerProps) {
  const [fullscreen, setFullscreen] = useState(false);

  return (
    <>
      <img
        src={src}
        alt={alt}
        className="artifact-image"
        onClick={() => setFullscreen(true)}
        style={{
          maxWidth: '100%',
          borderRadius: '8px',
          cursor: 'pointer',
          border: '1px solid #e0e0e0',
        }}
      />
      {fullscreen && (
        <Lightbox src={src} alt={alt} onClose={() => setFullscreen(false)} />
      )}
    </>
  );
}
```

### 6.3 视频播放器

```tsx
interface VideoPlayerProps {
  src: string;
}

function VideoPlayer({ src }: VideoPlayerProps) {
  return (
    <video
      src={src}
      controls
      style={{
        maxWidth: '100%',
        borderRadius: '8px',
        border: '1px solid #e0e0e0',
      }}
    >
      Your browser does not support video playback.
    </video>
  );
}
```

### 6.4 日志查看器

```tsx
interface TextReferenceProps {
  path: string;
  start: number;
  end: number;
  alt: string;
}

function TextReference({ path, start, end, alt }: TextReferenceProps) {
  const [content, setContent] = useState<string[]>([]);

  useEffect(() => {
    fetchFileLines(path, start, end).then(setContent);
  }, [path, start, end]);

  return (
    <div className="text-reference">
      <div className="header">{alt}</div>
      <pre>
        {content.map((line, i) => (
          <div key={i} className="line">
            <span className="line-number">{start + i}</span>
            <span className="line-content">{line}</span>
          </div>
        ))}
      </pre>
    </div>
  );
}
```

---

## 7. 视频审核流程

### 7.1 videoReview 子代理使用

```
Agent 保存录制后:
    │
    ├── 1. 保存录制 (SAVE_RECORDING)
    │
    ├── 2. 启动 videoReview 子代理
    │      prompt: "我认为视频展示了 X, 请验证:
    │               1. 是否展示了 Y?
    │               2. 是否有 Z 问题?"
    │      attachments: ["/opt/cursor/artifacts/demo_demo.mp4"]
    │
    ├── 3. 子代理分析视频帧, 返回验证结果
    │
    └── 4. Agent 根据结果决定是否引用视频
```

### 7.2 使用 demo 版本

```
原始录制:  demo_submit_button.mp4  (高分辨率, 大文件)
Demo 版本: demo_submit_button_demo.mp4 (720p, 小文件)

videoReview 子代理使用 demo 版本（更快传输和分析）
用户展示使用原始版本（更高质量）
```
