# 模块十一：测试与调试工作流详细设计

> 本文档详细描述 Agent 的测试策略引擎、证据收集机制、调试工作流、质量验证等。

---

## 1. 测试策略引擎

### 1.1 架构

```
┌────────────────────────────────────────────────────────┐
│                Testing Strategy Engine                   │
│                                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │ Change       │  │ Strategy     │  │ Evidence     │ │
│  │ Analyzer     │──▶ Planner      │──▶ Collector    │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
│                                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │ Test         │  │ Result       │  │ Iteration    │ │
│  │ Executor     │  │ Validator    │  │ Controller   │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
└────────────────────────────────────────────────────────┘
```

### 1.2 测试决策流程

```python
class TestingStrategyEngine:
    """测试策略引擎"""

    def plan(self, change_set: ChangeSet) -> TestPlan:
        """生成测试计划"""

        plan = TestPlan()

        # 1. 分析变更类型
        change_type = self.change_analyzer.analyze(change_set)

        # 2. 确定测试类型
        match change_type:
            case ChangeType.UI_CHANGE:
                plan.add(TestType.GUI_MANUAL, priority="required")
                plan.add(TestType.AUTOMATED, priority="recommended")
                plan.require_video_artifact = True

            case ChangeType.BUG_FIX:
                plan.add(TestType.REPRODUCE_BEFORE, priority="required")
                plan.add(TestType.VERIFY_AFTER, priority="required")
                plan.add(TestType.AUTOMATED, priority="recommended")

            case ChangeType.API_CHANGE:
                plan.add(TestType.TERMINAL_DRIVEN, priority="required")
                plan.add(TestType.AUTOMATED, priority="required")

            case ChangeType.PERFORMANCE:
                plan.add(TestType.BENCHMARK_BEFORE, priority="required")
                plan.add(TestType.BENCHMARK_AFTER, priority="required")
                plan.require_metrics_comparison = True

            case ChangeType.CONFIG_ONLY:
                plan.add(TestType.SMOKE_TEST, priority="recommended")

            case ChangeType.DOCS_ONLY:
                plan.add(TestType.NONE, priority="skip")

        # 3. 确定自动化测试范围
        existing_tests = self.find_relevant_tests(change_set)
        if existing_tests:
            plan.automated_tests = existing_tests

        # 4. 确定是否需要新测试
        if self.should_add_tests(change_set):
            plan.add(TestType.WRITE_NEW_TESTS, priority="recommended")

        return plan
```

### 1.3 变更分析器

```python
class ChangeAnalyzer:
    """分析代码变更类型"""

    UI_EXTENSIONS = {".tsx", ".jsx", ".html", ".css", ".scss", ".less", ".styl", ".vue"}
    TEST_EXTENSIONS = {".test.ts", ".test.js", ".spec.ts", ".spec.js", "_test.go", "_test.py"}
    DOC_EXTENSIONS = {".md", ".rst", ".txt", ".adoc"}
    CONFIG_EXTENSIONS = {".json", ".yaml", ".yml", ".toml", ".ini", ".env"}

    def analyze(self, change_set: ChangeSet) -> ChangeType:
        """分析变更集的类型"""

        file_types = set()
        for file in change_set.changed_files:
            ext = self._get_extension(file)
            if ext in self.UI_EXTENSIONS:
                file_types.add("ui")
            elif ext in self.TEST_EXTENSIONS:
                file_types.add("test")
            elif ext in self.DOC_EXTENSIONS:
                file_types.add("doc")
            elif ext in self.CONFIG_EXTENSIONS:
                file_types.add("config")
            else:
                file_types.add("code")

        # 优先级判断
        if "ui" in file_types:
            return ChangeType.UI_CHANGE
        elif file_types == {"doc"}:
            return ChangeType.DOCS_ONLY
        elif file_types == {"config"}:
            return ChangeType.CONFIG_ONLY
        elif "code" in file_types:
            return ChangeType.CODE_CHANGE
        else:
            return ChangeType.GENERAL

    def find_relevant_tests(self, change_set: ChangeSet) -> List[str]:
        """找到与变更相关的已有测试"""
        relevant_tests = []

        for changed_file in change_set.changed_files:
            # 常见测试文件命名模式
            base = os.path.splitext(changed_file)[0]
            test_patterns = [
                f"{base}.test.*",
                f"{base}.spec.*",
                f"{base}_test.*",
                f"test_{os.path.basename(base)}.*",
                f"tests/{os.path.basename(base)}.*",
                f"__tests__/{os.path.basename(base)}.*",
            ]

            for pattern in test_patterns:
                matches = glob.glob(pattern)
                relevant_tests.extend(matches)

        return list(set(relevant_tests))
```

---

## 2. 测试执行器

### 2.1 自动化测试执行

```python
class AutomatedTestExecutor:
    """自动化测试执行器"""

    def execute(self, test_config: TestConfig) -> TestResult:
        """执行自动化测试"""

        # 检测测试框架
        framework = self.detect_framework()

        # 构建测试命令
        match framework:
            case "jest":
                cmd = ["npx", "jest", "--coverage"]
                if test_config.specific_files:
                    cmd.extend(test_config.specific_files)

            case "pytest":
                cmd = ["python", "-m", "pytest", "-v"]
                if test_config.specific_files:
                    cmd.extend(test_config.specific_files)

            case "go":
                cmd = ["go", "test", "./..."]

            case "cargo":
                cmd = ["cargo", "test"]

            case _:
                return TestResult(status="skip", reason="Unknown test framework")

        # 执行
        result = self.shell.execute(
            command=" ".join(cmd),
            timeout=300000,  # 5 分钟
        )

        return self.parse_result(result, framework)

    def detect_framework(self) -> str:
        """检测项目使用的测试框架"""
        if os.path.exists("package.json"):
            pkg = json.load(open("package.json"))
            deps = {**pkg.get("dependencies", {}), **pkg.get("devDependencies", {})}
            if "jest" in deps:
                return "jest"
            if "vitest" in deps:
                return "vitest"
            if "mocha" in deps:
                return "mocha"

        if os.path.exists("pytest.ini") or os.path.exists("pyproject.toml"):
            return "pytest"

        if os.path.exists("go.mod"):
            return "go"

        if os.path.exists("Cargo.toml"):
            return "cargo"

        return "unknown"
```

### 2.2 手动测试执行（通过 computerUse）

```python
class ManualTestExecutor:
    """手动 GUI 测试执行"""

    def execute(self, test_plan: ManualTestPlan) -> ManualTestResult:
        """通过 computerUse 子代理执行手动测试"""

        # 1. 确保开发服务器运行
        self.ensure_dev_server_running()

        # 2. 设置录屏（如果需要视频证据）
        if test_plan.require_video:
            self.screen_recorder.start()

        # 3. 启动 computerUse 子代理
        result = self.task_tool.execute(
            subagent_type="computerUse",
            prompt=self._build_manual_test_prompt(test_plan),
        )

        # 4. 保存录屏
        if test_plan.require_video:
            if self._test_passed(result):
                self.screen_recorder.save(f"demo_{test_plan.name}")
            else:
                self.screen_recorder.discard()

        return self._parse_manual_result(result)

    def _build_manual_test_prompt(self, plan: ManualTestPlan) -> str:
        """构建手动测试指令"""
        return f"""
请在浏览器中测试以下功能:

测试目标: {plan.description}

步骤:
{plan.format_steps()}

预期结果:
{plan.expected_outcome}

完成后请截屏保存关键状态。
"""
```

---

## 3. 证据收集器

### 3.1 证据类型

```python
class EvidenceType(Enum):
    SCREENSHOT = "screenshot"        # 截图
    VIDEO = "video"                  # 录屏
    LOG = "log"                      # 日志输出
    TEST_RESULT = "test_result"      # 测试结果
    DIFF = "diff"                    # 代码差异
    METRICS = "metrics"              # 性能指标

class Evidence:
    type: EvidenceType
    path: str                        # 产物路径
    description: str
    timestamp: datetime
    is_conclusive: bool              # 是否具有结论性
```

### 3.2 证据质量评估

```python
class EvidenceValidator:
    """证据质量评估"""

    CONCLUSIVE_EVIDENCE = {
        # 好的证据
        "test_suite_pass": True,           # 测试套件全部通过
        "before_after_comparison": True,    # 修复前后对比
        "feature_demo_video": True,        # 功能演示视频
        "api_response_correct": True,      # API 响应正确
        "performance_improvement": True,    # 性能提升数据
    }

    INCONCLUSIVE_EVIDENCE = {
        # 不充分的证据
        "code_compiles": False,            # 代码可编译 ≠ 正确
        "app_starts": False,               # 应用可启动 ≠ 变更有效
        "homepage_loads": False,           # 首页可加载 ≠ 新功能工作
        "no_errors_in_console": False,     # 无控制台错误 ≠ 功能正确
    }

    def assess(self, evidence: List[Evidence]) -> QualityAssessment:
        """评估证据集的质量"""
        conclusive = [e for e in evidence if e.is_conclusive]
        inconclusive = [e for e in evidence if not e.is_conclusive]

        if not conclusive:
            return QualityAssessment(
                level="insufficient",
                message="没有结论性证据。需要更深入的测试。"
            )

        if len(conclusive) >= 2:
            return QualityAssessment(
                level="strong",
                message="充分的证据表明变更正确工作。"
            )

        return QualityAssessment(
            level="moderate",
            message="有一些证据，但可以更充分。"
        )
```

---

## 4. 测试迭代控制器

### 4.1 迭代循环

```
┌─────────────────┐
│ 执行测试         │
└───────┬─────────┘
        │
        ▼
┌─────────────────┐      失败
│ 验证结果         │─────────┐
└───────┬─────────┘         │
        │ 成功               ▼
        │              ┌─────────────┐
        ▼              │ 调查根因     │
   ┌─────────┐        └──────┬──────┘
   │ 收集证据 │               ▼
   └─────────┘        ┌─────────────┐
                       │ 修复问题     │
                       └──────┬──────┘
                              │
                              └──▶ (回到执行测试)
```

### 4.2 迭代规则

```python
class IterationController:
    """测试迭代控制"""

    MAX_ITERATIONS = 5        # 最大迭代次数
    MAX_SAME_ERROR = 3        # 同一错误最大重试

    def should_continue(self, history: List[TestAttempt]) -> Decision:
        """判断是否继续迭代"""

        if len(history) >= self.MAX_ITERATIONS:
            return Decision(
                continue_=False,
                reason="达到最大迭代次数"
            )

        # 检查是否在重复同一错误
        recent_errors = [h.error for h in history[-3:] if h.error]
        if len(set(recent_errors)) == 1 and len(recent_errors) >= self.MAX_SAME_ERROR:
            return Decision(
                continue_=False,
                reason="反复遇到相同错误，需要不同的解决方案"
            )

        # 检查是否有进展
        if len(history) >= 3:
            recent = history[-3:]
            if all(h.status == "failed" for h in recent):
                if all(h.error == recent[0].error for h in recent):
                    return Decision(
                        continue_=False,
                        reason="连续失败且无进展"
                    )

        return Decision(continue_=True)
```

---

## 5. Lint 检查集成

### 5.1 Lint 工具检测

```python
class LintDetector:
    """检测项目使用的 Lint 工具"""

    def detect(self) -> List[LintTool]:
        tools = []

        # ESLint
        if any(os.path.exists(f) for f in [
            ".eslintrc", ".eslintrc.js", ".eslintrc.json",
            ".eslintrc.yml", "eslint.config.js", "eslint.config.mjs"
        ]):
            tools.append(LintTool(
                name="eslint",
                command="npx eslint .",
                fixable=True,
                fix_command="npx eslint . --fix",
            ))

        # Prettier
        if any(os.path.exists(f) for f in [
            ".prettierrc", ".prettierrc.json", "prettier.config.js"
        ]):
            tools.append(LintTool(
                name="prettier",
                command="npx prettier --check .",
                fixable=True,
                fix_command="npx prettier --write .",
            ))

        # TypeScript
        if os.path.exists("tsconfig.json"):
            tools.append(LintTool(
                name="tsc",
                command="npx tsc --noEmit",
                fixable=False,
            ))

        # Python - ruff
        if os.path.exists("ruff.toml") or self._check_pyproject("ruff"):
            tools.append(LintTool(
                name="ruff",
                command="ruff check .",
                fixable=True,
                fix_command="ruff check . --fix",
            ))

        # Python - mypy
        if os.path.exists("mypy.ini") or self._check_pyproject("mypy"):
            tools.append(LintTool(
                name="mypy",
                command="mypy .",
                fixable=False,
            ))

        return tools
```

### 5.2 Lint 修复策略

```
Agent 引入 lint 错误时:

1. 运行 lint 检查, 发现错误
2. 如果工具支持自动修复 (--fix):
   a. 运行自动修复
   b. 检查修复结果
   c. 如果仍有错误 → 手动修复
3. 如果不支持自动修复:
   a. 分析错误信息
   b. 手动修复每个错误
4. 重新运行 lint 验证
```

---

## 6. 调试工作流

### 6.1 调试决策树

```
Bug 报告
    │
    ├── 根因明显？
    │   ├── 是 → 直接修复 → 测试验证
    │   └── 否 ↓
    │
    ├── 可复现？
    │   ├── 否 → 代码审查 + 日志分析 → 猜测性修复
    │   └── 是 ↓
    │
    ├── 复杂度？
    │   ├── 简单 (单文件/明显路径) → 直接调试
    │   └── 复杂 (多文件/隐蔽路径) ↓
    │
    └── 使用 Debug 子代理
        │
        ├── 1. 发送 Bug 描述和上下文
        ├── 2. 子代理提出假设 + 插桩
        ├── 3. 复现 Bug (computerUse / Shell)
        ├── 4. 反馈 "Issue reproduced"
        ├── 5. 子代理分析日志
        ├── 6. 子代理定位根因 / 新假设
        │       (循环 3-6 直到定位)
        ├── 7. 实施修复
        ├── 8. 验证修复
        └── 9. 清理插桩代码
```

### 6.2 插桩策略

```python
class InstrumentationStrategy:
    """调试插桩策略"""

    LOG_LEVELS = {
        "entry": "函数入口",
        "exit": "函数出口",
        "state": "状态变量",
        "branch": "条件分支",
        "error": "错误捕获",
    }

    def plan_instrumentation(
        self,
        hypothesis: str,
        relevant_files: List[str],
    ) -> List[InstrumentPoint]:
        """规划插桩点"""
        points = []

        for file in relevant_files:
            # 分析代码结构
            ast = self.parse_ast(file)

            # 在关键位置添加日志
            for func in ast.functions:
                if self.is_relevant(func, hypothesis):
                    points.append(InstrumentPoint(
                        file=file,
                        line=func.start_line,
                        type="entry",
                        log=f"[DEBUG] Entering {func.name}, args: {{args}}",
                    ))

                    for branch in func.branches:
                        points.append(InstrumentPoint(
                            file=file,
                            line=branch.line,
                            type="branch",
                            log=f"[DEBUG] Branch {branch.condition}: {{result}}",
                        ))

        return points

    def generate_cleanup_instructions(
        self,
        points: List[InstrumentPoint],
    ) -> str:
        """生成清理指令"""
        instructions = "请移除以下调试代码:\n\n"
        for point in points:
            instructions += f"- {point.file}:{point.line}: {point.log}\n"
        return instructions
```

---

## 7. 测试报告

### 7.1 最终报告格式

```python
class TestReport:
    """测试报告"""

    def format_for_user(self) -> str:
        """格式化测试报告供用户查看"""

        lines = ["**Testing**"]

        for test in self.tests:
            emoji = self._get_emoji(test.status)
            lines.append(f"* {emoji} `{test.command}`")
            if test.note:
                lines.append(f"  {test.note}")

        return "\n".join(lines)

    def _get_emoji(self, status: str) -> str:
        return {
            "pass": "✅",
            "warning": "⚠️",
            "fail": "❌",
        }[status]

    # 示例输出:
    # **Testing**
    # * ✅ `npm run build`
    # * ✅ `npm run lint`
    # * ✅ `npm run test -- --coverage`
    # * ⚠️ `npm run e2e` (环境限制: 无外部数据库)
```

### 7.2 证据展示

```markdown
**Walkthrough**
<video src="/opt/cursor/artifacts/demo_feature.mp4" controls></video>
表单提交后显示加载动画，数据正确保存。

<img src="/opt/cursor/artifacts/before_fix.webp" alt="修复前" />
<img src="/opt/cursor/artifacts/after_fix.webp" alt="修复后" />

**Summary**
* 修复了表单提交时的空指针异常
* 添加了加载状态指示器

**Testing**
* ✅ `npm run test`
* ✅ 手动测试：表单提交功能正常（见视频）
```
