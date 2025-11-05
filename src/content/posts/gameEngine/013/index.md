---
title: ImGui 事件处理实现：对接引擎事件系统
published: 2025-11-05
description: "基于cherno Hazel引擎教学"
tags: ["学习", "游戏", "引擎"]
category: 学习
draft: false
---


This blog template is built with [Astro](https://astro.build/). For the things that are not mentioned in this guide, you may find the answers in the [Astro Docs](https://docs.astro.build/).



在游戏引擎开发中，输入检测是连接玩家操作与游戏逻辑的核心纽带。当引擎基于 GLFW 实现初步输入查询功能后，会面临一个隐藏的 “跨平台陷阱”—— 直接依赖 GLFW 原生键值定义，会导致后续扩展平台或更换窗口库时出现兼容性断层。本文将详细解析这一问题的根源，阐述自定义键值方案的设计思路与实现过程，并通过实际测试验证其有效性，为引擎输入模块的跨平台能力打下基础。

# 一、现状与痛点：GLFW 原生键值的局限性

在之前的输入模块开发中，我们通过封装 GLFW 的glfwGetKey等函数，实现了键盘与鼠标输入的全局查询。但此时的输入检测完全**依赖 GLFW 定义的宏**（如GLFW_KEY_A、GLFW_MOUSE_BUTTON_1），在实际开发与跨平台扩展中逐渐暴露两大核心问题：

## 1. 代码耦合度高，依赖边界模糊

直接使用 GLFW 键值时，所有需要检测输入的代码模块（如游戏逻辑层、UI 交互层）都必须包含 GLFW 的头文件（glfw3.h）。这导致引擎核心模块与底层窗口库产生强耦合 —— 若后续因需求变更更换窗口库（如 Windows 平台改用 Win32 API，移动端改用 SDL），所有引用GLFW_KEY_*的代码都需要修改，维护成本极高。
例如，原本检测 “A 键按下” 的代码如下：
```cpp
// 依赖GLFW头文件的写法，耦合度高
#include <GLFW/glfw3.h>
if (Input::IsKeyPressed(GLFW_KEY_A)) {
    // 执行A键对应的逻辑
}
```
一旦脱离 GLFW 环境，GLFW_KEY_A的定义将失效。

## 2. 跨平台键值不统一，兼容性断层

不同窗口库或操作系统的键值定义存在差异。以 “鼠标左键” 为例：GLFW 中定义为GLFW_MOUSE_BUTTON_1，而 Win32 API 中通过MK_LBUTTON标识，macOS 的 Cocoa 框架又有独立的键值编码。若引擎仅依赖 GLFW 键值，当需要**适配多平台**时，将面临 “**一套输入逻辑，多套键值判断**” 的混乱局面，无法实现 “一次编码，多端运行” 的跨平台目标。

# 二、解决方案：自定义键值体系的设计与实现

为打破 GLFW 原生键值的局限，核心思路是**构建引擎专属的键值抽象层**—— 参考 GLFW 的键值映射逻辑，定义一套与底层窗口库无关的自定义键值（如HZ_KEY_A、HZ_MOUSE_BUTTON_LEFT），再通过输入模块的底层实现，将**自定义键值与当前平台的原生键值进行映射**。这样既能保证上层代码的统一性，又能灵活适配不同底层库。
## 1. 自定义键值宏定义：构建统一抽象层
我们分别创建`KeyCodes.h`和`MouseButtonCodes.h`两个头文件，集中定义引擎的自定义键值。键值的数值参考 GLFW 的定义（避免无意义的数值冲突），命名前缀统一为HZ_（代表引擎名称，如 Hazel），确保与底层库键值区分开。
（1）键盘键值定义（KeyCodes.h）
```cpp
//KeyCodes.h
// 键盘键值：参考GLFW定义，构建引擎自定义键值体系
#define HZ_KEY_UNKNOWN            -1

// 打印键区
#define HZ_KEY_SPACE              32
#define HZ_KEY_APOSTROPHE         39  /* ' */
#define HZ_KEY_COMMA              44  /* , */
#define HZ_KEY_MINUS              45  /* - */
#define HZ_KEY_PERIOD             46  /* . */
#define HZ_KEY_SLASH              47  /* / */
#define HZ_KEY_0                  48
#define HZ_KEY_1                  49
// ...（省略数字键、字母键定义，逻辑与GLFW一致）
#define HZ_KEY_A                  65
#define HZ_KEY_B                  66
#define HZ_KEY_C                  67
// ...（省略其余字母键）

// 功能键
#define HZ_KEY_ESCAPE             256
#define HZ_KEY_ENTER              257
#define HZ_KEY_TAB                258
#define HZ_KEY_BACKSPACE          259
#define HZ_KEY_INSERT             260
#define HZ_KEY_DELETE             261
// ...（省略方向键、功能键F1-F12等定义）
```
（2）鼠标键值定义（MouseButtonCodes.h）
```cpp
//MouseButtonCodes.h
// 鼠标键值：同样参考GLFW，统一命名风格
#define HZ_MOUSE_BUTTON_1         0
#define HZ_MOUSE_BUTTON_2         1
#define HZ_MOUSE_BUTTON_3         2
#define HZ_MOUSE_BUTTON_4         3
#define HZ_MOUSE_BUTTON_5         4
#define HZ_MOUSE_BUTTON_6         5
#define HZ_MOUSE_BUTTON_7         6
#define HZ_MOUSE_BUTTON_8         7

// 常用鼠标按键别名，提升代码可读性
#define HZ_MOUSE_BUTTON_LEFT      HZ_MOUSE_BUTTON_1
#define HZ_MOUSE_BUTTON_RIGHT     HZ_MOUSE_BUTTON_2
#define HZ_MOUSE_BUTTON_MIDDLE    HZ_MOUSE_BUTTON_3
```
## 2. 底层映射：实现自定义键值与原生键值的关联

自定义键值的核心价值，在于底层实现中 “无感映射” 到当前平台的**原生键值**。由于当前引擎仍基于 GLFW 开发，我们只需在WindowsInput（或对应平台的输入实现类）中，直接使用自定义键值作为参数 —— 因为自定义键值的数值与 GLFW 完全一致，无需额外转换，即可直接传入 GLFW 函数。

例如，修改IsKeyPressedImpl方法的参数类型（从int隐性适配为自定义键值）：
```cpp
// WindowsInput.cpp
bool WindowsInput::IsKeyPressedImpl(int keycode) {
    // keycode此时传入的是HZ_KEY_*（如HZ_KEY_A），数值与GLFW_KEY_A一致
    auto window = static_cast<GLFWwindow*>(
        Application::Get().GetWindow().GetNativeWindow()
    );
    auto state = glfwGetKey(window, keycode);
    return state == GLFW_PRESS || state == GLFW_REPEAT;
}
```
若后续更换为 Win32 API，只需在新的平台输入类（如Win32Input）中，添加自定义键值到 Win32 键值的映射表即可，上层代码无需修改：
```cpp
// 未来适配Win32时的映射示例（伪代码）
int Win32Input::MapHzKeyToWin32(int hzKeycode) {
    static std::unordered_map<int, int> keyMap = {
        {HZ_KEY_A, 0x41},    // Win32中A键的虚拟键码是0x41
        {HZ_KEY_ENTER, 0x0D}, // Win32中Enter键的虚拟键码是0x0D
        // ...其他键值映射
    };
    return keyMap.count(hzKeycode) ? keyMap[hzKeycode] : -1;
}
```
# 三、测试验证：自定义键值在实际场景中的应用
为确保自定义键值体系能正常工作，我们在ExampleLayer（引擎的示例层）中，分别在输入轮询（OnUpdate）和事件系统（OnEvent）两个核心场景中进行测试，验证自定义键值的有效性。
## 1. 场景 1：输入轮询（OnUpdate）
在引擎主循环的更新阶段，通过Input::IsKeyPressed直接传入自定义键值HZ_KEY_A，检测 A 键是否按下：
```cpp
// SandboxApp.cpp
void ExampleLayer::OnUpdate() {
    // 使用自定义键值HZ_KEY_A检测A键，无需包含GLFW头文件
    if (Hazel::Input::IsKeyPressed(HZ_KEY_A)) {
        HZ_CORE_TRACE("A键按下(pull)");
    }
    // ...其他更新逻辑
}
```

## 2. 场景 2：事件系统（OnEvent）
在事件回调中，通过KeyPressedEvent的GetKeyCode方法获取自定义键值，判断是否为目标按键：
```cpp
// SandboxApp.cpp
void OnEvent(Hazel::Event& event) override 
{
    //HZ_TRACE("{0}",event.ToString());
    if(event.GetEventType() == Hazel::EventType::KeyPressed)
    {
        Hazel::KeyPressedEvent& e = (Hazel::KeyPressedEvent&)event;
        if (e.GetKeyCode() == HZ_KEY_A)
        {
            HZ_TRACE("A键按下(event)");
        }
    }
}
```

## 3. 测试结果
运行引擎后，日志窗口输出如下内容，证明自定义键值在两种场景下均能正常工作：
```plaintext
[16:30:05] HAZEL: A键按下(event)
[16:30:05] HAZEL: A键按下(poll)
[16:30:06] HAZEL: A键按下(poll)
[16:30:06] HAZEL: A键按下(poll)
```
同时，代码中无需包含glfw3.h，彻底解除了对 GLFW 的直接依赖，为后续跨平台扩展扫清障碍。

# 四、总结：自定义键值的核心价值与扩展方向
本次优化通过构建自定义键值体系，成功解决了 GLFW 原生键值带来的耦合与兼容性问题，其核心价值体现在三个方面：
- **解耦底层依赖**：上层代码**仅依赖引擎自定义键值**，无需关注底层窗口库，降低维护成本；
- **统一跨平台接口**：无论后续适配 Win32、SDL 还是移动端库，只需修改底层映射，上层逻辑保持不变；
- **提升代码可读性**：通过HZ_MOUSE_BUTTON_LEFT等别名，让代码意图更清晰（相比GLFW_MOUSE_BUTTON_1更易理解）。

后续可基于该体系进一步扩展：
- **添加键值映射配置文件**：将键值映射逻辑从代码中抽离到配置文件（如 JSON），支持动态修改键值对应关系，适配不同玩家的操作习惯；
- **扩展游戏手柄键值**：参考键盘与鼠标的设计，定义GamepadCodes.h，支持手柄按键（如HZ_GAMEPAD_BUTTON_A）的统一检测；
- **实现键值别名管理**：允许开发者为同一键值设置多个别名（如HZ_KEY_JUMP映射到HZ_KEY_SPACE），让游戏逻辑与物理按键解耦，提升代码灵活性。