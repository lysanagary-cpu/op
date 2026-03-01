# AI 视觉驱动的电脑自动操作设计

## 1. 总体流程

```text
用户下达任务
  ↓
截屏（当前桌面）
  ↓
AI Vision 分析（识别 UI 元素 + 任务阶段）
  ↓
动作规划（下一步点击/输入/滚动）
  ↓
执行动作（鼠标键盘控制）
  ↓
再次截屏并校验结果
  ↓
循环，直到完成任务或触发异常处理
```

该系统本质是一个 **Observe → Think → Act** 闭环：

- **Observe（观察）**：通过截图获取环境状态。
- **Think（思考）**：通过视觉模型理解界面与任务进度。
- **Act（执行）**：通过鼠标键盘控制器执行下一步。

## 2. 核心模块

### 2.1 任务管理器（Task Manager）

负责接收用户自然语言目标，并拆分为可执行子目标。

- 输入：用户任务（如“帮我发一封邮件给张三”）
- 输出：阶段化目标（打开邮箱 → 点击写信 → 填写内容 → 发送）

### 2.2 屏幕感知模块（Perception）

负责截图、元素识别、状态抽取。

- 全屏截图 / 指定区域截图
- OCR（文本识别）
- UI 元素检测（按钮、输入框、菜单、弹窗）
- 关键状态结构化输出（JSON）

示例输出：

```json
{
  "window": "Gmail - Inbox",
  "elements": [
    {"type": "button", "text": "Compose", "bbox": [120, 180, 210, 220]},
    {"type": "input", "label": "To", "bbox": [430, 240, 890, 280]}
  ],
  "alerts": []
}
```

### 2.3 决策与规划模块（Planner）

根据当前界面状态，推理“下一步最小动作”。

- 动作类型：`click` / `double_click` / `type` / `hotkey` / `scroll` / `wait`
- 目标定位：元素文本 + 坐标 + 置信度
- 失败回退：重试、换策略、请求用户确认

### 2.4 执行器（Actuator）

将高层动作转换为系统级输入事件。

- 鼠标移动与点击（支持偏移点击防遮挡）
- 键盘输入（支持中文输入法状态检查）
- 滚动控制（固定步长/自适应滚动）

### 2.5 结果校验器（Validator）

每步执行后做“动作后验证”。

- 预期按钮是否消失/新页面是否出现
- 输入框文字是否正确填入
- 是否出现错误提示弹窗

若校验失败：
1. 触发局部重试
2. 超过阈值后切换备用路径
3. 仍失败则暂停并向用户报告

## 3. 控制循环（伪代码）

```python
while not task.done:
    screen = capture_screen()
    state = vision_analyze(screen)

    plan = planner.next_action(task, state)
    if plan.requires_confirmation:
        ask_user(plan.reason)
        continue

    execute(plan.action)

    new_screen = capture_screen()
    result = validate(plan, new_screen)

    if result.ok:
        task.advance()
    else:
        recovery.handle(result)
```

## 4. 关键工程点

1. **坐标鲁棒性**：
   - 不依赖固定像素点，优先文本和语义定位。
   - 支持分辨率变化与窗口缩放。

2. **安全边界**：
   - 高风险动作（二次确认）：转账、删除、发送、提交。
   - 敏感信息脱敏显示与日志加密存储。

3. **可观测性**：
   - 每一步保留截图、动作、推理理由、执行结果。
   - 支持回放与审计。

4. **异常处理**：
   - 弹窗干扰、网络延迟、页面卡死、元素遮挡等需专门策略。

## 5. MVP 建议

第一版可先支持单一场景（如浏览器中的表单填写）：

- 仅支持 `click/type/scroll/wait`
- 限制在单窗口内执行
- 使用规则 + 视觉模型混合决策
- 失败后立即请求用户接管

这样可以快速验证“截图→理解→操作→校验”的闭环可行性，再逐步扩展到多应用和复杂任务。
