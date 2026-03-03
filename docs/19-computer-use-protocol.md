# 模块十九：Computer Use 与浏览器自动化协议详细设计

> 本文档详细描述 computerUse 子代理的操作协议、视觉感知机制、浏览器自动化、
> 坐标系统、等待策略、以及与主 Agent 的协作模式。

---

## 1. Computer Use 架构

```
┌─────────────────────────────────────────────────────────────┐
│                  computerUse Subagent                         │
│                                                             │
│  ┌──────────────┐     ┌──────────────┐     ┌─────────────┐ │
│  │ Vision       │     │ Action       │     │ Verification│ │
│  │ Model        │────▶│ Executor     │────▶│ Loop        │ │
│  │ (截屏分析)    │     │ (操作执行)    │     │ (结果验证)   │ │
│  └──────────────┘     └──────────────┘     └─────────────┘ │
│         ▲                    │                    │         │
│         │                    ▼                    │         │
│         │              ┌──────────────┐           │         │
│         └──────────────│ Screenshot   │◀──────────┘         │
│                        │ (截屏服务)    │                     │
│                        └──────────────┘                     │
│                                                             │
│  操作目标: VM 桌面 (X11, 1920x1080)                          │
│  浏览器: Google Chrome (预装)                                │
│  窗口管理器: Fluxbox                                        │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 操作原语 (Action Primitives)

### 2.1 完整操作列表

```python
class ComputerAction:
    """computerUse 子代理可执行的操作"""

    # === 鼠标操作 ===

    def mouse_move(self, x: int, y: int):
        """移动鼠标到指定坐标"""
        # 坐标系: 左上角 (0,0), 右下角 (1920,1080)

    def left_click(self, x: int, y: int):
        """左键单击"""

    def right_click(self, x: int, y: int):
        """右键单击 (上下文菜单)"""

    def double_click(self, x: int, y: int):
        """双击"""

    def middle_click(self, x: int, y: int):
        """中键单击"""

    def mouse_down(self, x: int, y: int, button: str = "left"):
        """按下鼠标按键 (不松开, 用于拖拽)"""

    def mouse_up(self, x: int, y: int, button: str = "left"):
        """松开鼠标按键"""

    # === 键盘操作 ===

    def type_text(self, text: str):
        """输入文本 (逐字符, 支持 Unicode)"""

    def key_press(self, key: str):
        """按键 (单个键)"""
        # 支持: enter, tab, escape, backspace, delete,
        #       up, down, left, right,
        #       home, end, pageup, pagedown,
        #       f1-f12, space

    def hotkey(self, *keys: str):
        """组合键"""
        # 示例: hotkey("ctrl", "c"), hotkey("ctrl", "shift", "i")

    # === 滚动 ===

    def scroll_up(self, amount: int = 3):
        """向上滚动"""

    def scroll_down(self, amount: int = 3):
        """向下滚动"""

    # === 截屏 ===

    def screenshot(self) -> str:
        """截取当前屏幕, 返回图片路径"""

    # === 等待 ===

    def wait(self, seconds: float):
        """等待指定秒数"""

    # === Shell 命令 ===

    def bash(self, command: str) -> str:
        """在 VM 中执行 bash 命令"""
```

### 2.2 坐标系统

```
(0,0) ────────────────────────────── (1920,0)
  │                                      │
  │        ┌──────────────────┐          │
  │        │  Chrome Browser  │          │
  │        │                  │          │
  │        │  URL Bar  (0, ~90)         │
  │        │                  │          │
  │        │  Page Content    │          │
  │        │  (x, y)         │          │
  │        │                  │          │
  │        └──────────────────┘          │
  │                                      │
  │        ┌──────────────────┐          │
  │        │  Taskbar (底部)   │          │
  │        └──────────────────┘          │
  │                                      │
(0,1080) ──────────────────────── (1920,1080)
```

---

## 3. 视觉感知循环

### 3.1 操作-观察循环

```
┌────────────────────────────────────────────────┐
│          computerUse 主循环                      │
│                                                │
│  1. 截屏 screenshot()                           │
│     ↓                                          │
│  2. 视觉模型分析截屏                             │
│     ├── 识别 UI 元素 (按钮, 输入框, 文本)        │
│     ├── 确定目标元素的坐标                       │
│     └── 判断当前页面状态                         │
│     ↓                                          │
│  3. 决策: 下一步操作                             │
│     ├── 点击某个按钮?                           │
│     ├── 在输入框中键入?                          │
│     ├── 滚动查找元素?                           │
│     └── 等待页面加载?                            │
│     ↓                                          │
│  4. 执行操作                                    │
│     ↓                                          │
│  5. 等待 (500ms - 3s)                           │
│     ↓                                          │
│  6. 截屏验证操作结果                             │
│     ├── 成功 → 继续下一步                        │
│     └── 失败 → 调整策略重试                      │
│     ↓                                          │
│  (回到步骤 1, 循环直到任务完成)                   │
└────────────────────────────────────────────────┘
```

### 3.2 视觉分析示例

```
截屏内容 (LLM 视觉模型看到的):

┌──────────────────────────────────────────┐
│ Chrome - Login Page                       │
│ ◀ ▶ 🔄  localhost:3000/login             │
├──────────────────────────────────────────┤
│                                          │
│        ╔══════════════════════╗          │
│        ║     Welcome Back     ║          │
│        ╠══════════════════════╣          │
│        ║                      ║          │
│        ║  Email:              ║          │
│        ║  ┌────────────────┐  ║  ← (960, 400)
│        ║  │                │  ║          │
│        ║  └────────────────┘  ║          │
│        ║                      ║          │
│        ║  Password:           ║          │
│        ║  ┌────────────────┐  ║  ← (960, 480)
│        ║  │                │  ║          │
│        ║  └────────────────┘  ║          │
│        ║                      ║          │
│        ║  ┌────────────────┐  ║          │
│        ║  │   Log In       │  ║  ← (960, 560)
│        ║  └────────────────┘  ║          │
│        ╚══════════════════════╝          │
│                                          │
└──────────────────────────────────────────┘

LLM 分析:
  - 当前在登录页面
  - 邮箱输入框大约在 (960, 400)
  - 密码输入框大约在 (960, 480)  
  - 登录按钮大约在 (960, 560)
  
下一步操作:
  1. click(960, 400)    # 点击邮箱框
  2. type("test@...")    # 输入邮箱
  3. click(960, 480)    # 点击密码框
  4. type("password")   # 输入密码
  5. click(960, 560)    # 点击登录
```

---

## 4. 浏览器管理

### 4.1 Chrome 启动配置

```python
class ChromeManager:
    """Chrome 浏览器管理"""

    def launch(self, url: str = None):
        """启动 Chrome"""
        cmd = [
            "google-chrome",
            "--no-sandbox",              # 容器内必需
            "--disable-gpu",             # 无 GPU
            "--disable-dev-shm-usage",   # 使用 /tmp
            "--window-size=1920,1080",   # 全屏尺寸
            "--window-position=0,0",
            "--disable-extensions",       # 禁用扩展
            "--disable-popup-blocking",   # 允许弹窗
            "--disable-translate",        # 禁用翻译
            "--no-first-run",            # 跳过首次运行
            "--no-default-browser-check",
        ]

        if url:
            cmd.append(url)

        subprocess.Popen(
            cmd,
            env={**os.environ, "DISPLAY": ":0"},
        )

    def navigate(self, url: str):
        """导航到 URL (通过键盘快捷键)"""
        hotkey("ctrl", "l")        # 聚焦地址栏
        wait(0.3)
        hotkey("ctrl", "a")        # 全选
        type_text(url)             # 输入 URL
        key_press("enter")        # 回车

    def open_devtools(self):
        """打开开发者工具"""
        hotkey("ctrl", "shift", "i")

    def refresh(self):
        """刷新页面"""
        hotkey("ctrl", "r")

    def close_tab(self):
        """关闭当前标签页"""
        hotkey("ctrl", "w")

    def new_tab(self):
        """新建标签页"""
        hotkey("ctrl", "t")
```

---

## 5. 等待策略

### 5.1 智能等待

```python
class WaitStrategy:
    """智能等待策略"""

    def wait_for_page_load(self, max_wait: float = 10.0):
        """等待页面加载完成"""
        start = time.time()
        last_screenshot = None

        while time.time() - start < max_wait:
            current = self.screenshot()

            if last_screenshot and self.images_similar(last_screenshot, current):
                # 页面稳定了 (两次截屏相同)
                return True

            last_screenshot = current
            time.sleep(1.0)

        return False  # 超时

    def wait_for_element(
        self,
        description: str,
        max_wait: float = 10.0,
    ) -> Optional[tuple]:
        """等待特定元素出现"""
        start = time.time()

        while time.time() - start < max_wait:
            screenshot = self.screenshot()
            # 使用 LLM 视觉模型检测元素
            result = self.detect_element(screenshot, description)

            if result.found:
                return result.coordinates

            time.sleep(1.0)

        return None  # 未找到

    def wait_for_text(
        self,
        text: str,
        max_wait: float = 10.0,
    ) -> bool:
        """等待特定文本出现在页面上"""
        start = time.time()

        while time.time() - start < max_wait:
            screenshot = self.screenshot()
            if self.ocr_contains(screenshot, text):
                return True
            time.sleep(1.0)

        return False
```

---

## 6. 与主 Agent 的协作协议

### 6.1 请求格式

```python
# 主 Agent → computerUse 子代理

prompt = """
请在浏览器中执行以下操作:

前提条件:
- Chrome 已打开, 当前在 localhost:3000

操作步骤:
1. 点击导航栏中的 "Settings" 链接
2. 在 "Display Name" 输入框中输入 "Test User"
3. 点击 "Save" 按钮
4. 等待成功提示出现
5. 截屏保存最终状态

期望结果:
- 显示 "Settings saved successfully" 提示
- Display Name 字段显示 "Test User"

请将关键截图保存到 /opt/cursor/artifacts/
"""
```

### 6.2 响应格式

```python
# computerUse 子代理 → 主 Agent

response = """
操作已完成。以下是执行摘要:

1. ✅ 点击了 "Settings" 链接, 页面导航到设置页
2. ✅ 在 "Display Name" 框中输入了 "Test User"
3. ✅ 点击了 "Save" 按钮
4. ✅ 出现了 "Settings saved successfully" 绿色提示
5. ✅ 截屏已保存

截图路径:
- /tmp/screenshots/screenshot_1709420100.png (设置页面)
- /tmp/screenshots/screenshot_1709420105.png (保存成功)
- /opt/cursor/artifacts/settings_saved.png (最终状态)
"""
```

---

## 7. 错误处理与恢复

```python
class ComputerUseErrorHandler:
    """computerUse 错误处理"""

    def handle_click_miss(self):
        """点击位置不准确"""
        # 1. 重新截屏
        # 2. 重新分析元素位置
        # 3. 调整坐标重试
        # 通常: 元素可能移动了位置 (动画/加载后)

    def handle_page_not_loaded(self):
        """页面未加载"""
        # 1. 等待更长时间
        # 2. 刷新页面
        # 3. 检查控制台错误

    def handle_element_not_found(self):
        """找不到目标元素"""
        # 1. 滚动页面寻找
        # 2. 检查是否在不同标签页
        # 3. 检查弹窗/对话框是否遮挡

    def handle_unexpected_dialog(self):
        """出现意外的对话框"""
        # 1. 截屏分析对话框内容
        # 2. 点击适当的按钮 (确定/取消/关闭)
        # 3. 继续原操作

    def handle_crash(self):
        """浏览器崩溃"""
        # 1. 重新启动 Chrome
        # 2. 导航回之前的 URL
        # 3. 重试操作
```

---

## 8. 录屏与 computerUse 的协调

```
时序图: 录屏 + GUI 测试

主 Agent                  RecordScreen           computerUse
    │                          │                      │
    │ 1. START_RECORDING       │                      │
    │─────────────────────────▶│                      │
    │                          │ (开始录制桌面)         │
    │                          │                      │
    │ 2. Task(computerUse, "测试步骤...")              │
    │─────────────────────────────────────────────────▶│
    │                                                  │
    │                          │  (子代理操作浏览器,     │
    │                          │   录屏同步捕获)        │
    │                          │                      │
    │ 3. 子代理返回结果                                 │
    │◀─────────────────────────────────────────────────│
    │                          │                      │
    │ 4. 判断: 测试成功?        │                      │
    │                          │                      │
    ├── 成功:                  │                      │
    │   SAVE_RECORDING         │                      │
    │─────────────────────────▶│                      │
    │                          │ → /opt/cursor/artifacts/
    │                          │                      │
    └── 失败:                  │                      │
        DISCARD_RECORDING      │                      │
        ──────────────────────▶│                      │
                               │ (丢弃)               │
        修复问题 → 重试                                │
```
