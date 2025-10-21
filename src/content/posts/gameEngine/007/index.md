---
title: 窗口事件
published: 2025-10-21
description: "基于cherno Hazel引擎教学"
tags: ["学习", "游戏", "引擎"]
category: 学习
draft: false
---


This blog template is built with [Astro](https://astro.build/). For the things that are not mentioned in this guide, you may find the answers in the [Astro Docs](https://docs.astro.build/).


# 一. 数据流设计

我们已经构建了事件框架，抽象窗口类，Windows窗口类（GLFW窗口），现在要把他们联系起来。

我们需要处理窗口的各种事件，比如关闭窗口、调整窗口大小等。我们可以使用GLFW提供的回调函数来捕获这些事件，并将它们传递给我们的事件系统。

下面是我们的数据流设计：

```txt
1. GLFW 检测到原生事件
   ↓
2. 调用注册的回调函数→在回调中使用GLFW提供的用户指针 获取窗口数据 data→创建对应的 Hazel 事件对象→调用 data.EventCallback(事件对象)
   ↓
3. 将Hazel事件传递给应用层，EventCallback 指向 Application::OnEvent
   ↓
4. Application 应用层处理事件
```

# 二. 窗口事件设计

这里以关闭窗口事件为例，一步一步设计：

## 一) GLFW 检测到原生事件

GLFW窗口

## 二) 窗口回调注册

在 WindowsWindow 类中，我们注册 GLFW 回调函数：
```cpp
$WindowsWindow.cpp

// 设置Windows窗口回调函数
void WindowsWindow::SetCallbacks()
{
    // 窗口关闭事件回调
    glfwSetWindowCloseCallback(m_Window, [](GLFWwindow* window)
        {
            WindowData& data = *(WindowData*)glfwGetWindowUserPointer(window);// 获取窗口数据
            WindowCloseEvent event;// 创建关闭事件对象  
            data.EventCallback(event);// 调用事件回调
        });
}
```

## 三) 将Hazel事件传递给应用层

```cpp
$WindowsWindow.h

// 设置事件回调函数 
// 将事件回调函数(callback)存储在窗口数据中
inline void SetEventCallback(const EventCallbackFn& callback) override { m_Data.EventCallback = callback; };
```

```cpp
$Application.cpp

// 绑定事件函数的宏定义
// 将成员函数绑定到当前对象，并占位第一个参数
#define BIND_EVENT_FN(x) std::bind(&Application::x, this, std::placeholders::_1)

Application::Application()
{
    // Application创建窗口
    m_Window = std::unique_ptr<Window>(Window::Create());
    // Application设置窗口事件的回调函数
    // EventCallback指向Application::OnEvent
    m_Window->SetEventCallback(BIND_EVENT_FN(OnEvent));
}

// 回调glfw窗口事件的函数
void Application::OnEvent(Event& e)
{
}
```

## 四) Application 应用层处理事件   

在 Application 类中，我们实现 OnEvent 方法来处理传递过来的事件：
```cpp
$Application.cpp

void Application::OnEvent(Event& e)
{
    EventDispatcher dispatcher(e);
    dispatcher.Dispatch<WindowCloseEvent>(BIND_EVENT_FN(Application::OnWindowClose));
}
```
通过这种方式，我们实现了从 GLFW 检测到原生事件，到将事件传递给应用层进行处理的完整数据流。

# 三. 数据流验证

```txt
// 测试完整流程
int main() {
    Application app;
    
    // 数据流开始：
    // 1. 用户按下ESC键 → GLFW检测到原生事件
    // 2. GLFW调用我们通过glfwSetKeyCallback注册的Lambda函数
    // 3. 在Lambda中：glfwGetWindowUserPointer获取WindowData
    // 4. 创建KeyPressedEvent事件对象
    // 5. 调用data.EventCallback(keyPressedEvent)
    // 6. EventCallback指向Application::OnEvent
    // 7. Application::OnEvent处理KeyPressedEvent，设置m_Running = false
    
    app.Run();  // 循环直到m_Running为false
    
    return 0;
}
```

# Tip

## 一. 回调函数

### 什么是回调函数

**回调函数**是一种通过**函数指针**调用的函数，其核心思想是**将函数作为参数**传递给另一个函数，并在特定条件下由接收方调用。这种机制在解耦、异步处理和提高代码灵活性方面具有重要作用。

```cpp
// 回调函数就是"你告诉我发生某件事时该调用什么函数"
// 类似于："有事打我电话"

// 定义一个函数指针类型
using CallbackFunction = std::function<void(int)>;

class EventSystem {
    CallbackFunction m_Callback;  // 保存"电话号码"
    
public:
    // 设置回调函数 - "告诉我你的电话号码"
    void SetCallback(const CallbackFunction& callback) {
        m_Callback = callback;
    }
    
    // 当事件发生时 - "有事发生了，我要打电话通知你"
    void OnEventHappened(int eventData) {
        if (m_Callback) {
            m_Callback(eventData);  // "打电话通知"
        }
    }
};
```

### 理解 GLFW 回调机制

GLFW 的工作方式：

```cpp
// GLFW 提供了这些回调设置函数：
glfwSetKeyCallback(window, key_callback);
glfwSetMouseButtonCallback(window, mouse_button_callback);  
glfwSetCursorPosCallback(window, cursor_position_callback);
glfwSetScrollCallback(window, scroll_callback);
glfwSetWindowSizeCallback(window, window_size_callback);
glfwSetWindowCloseCallback(window, window_close_callback);

// 这些函数的特点：
// 1. 需要 C 风格函数指针（不能直接使用 C++ 成员函数）
// 2. 回调函数必须是静态的或全局的
// 3. 没有上下文信息（不知道是哪个窗口实例）
```

**问题**： 如何在静态函数中访问我们的 C++ 对象？

## 二. 用户指针

回调函数需要当前窗口数据，我们使用用户指针保存窗口数据

GLFW提供了 **glfwSetWindowUserPointer** 和 **glfwGetWindowUserPointer**

```cpp
// 1. 创建窗口时，把我们的 C++ 对象指针保存到 GLFW
// 2. 在静态回调函数中，从 GLFW 获取这个指针
// 3. 通过指针调用实例方法

class WindowsWindow {
private:
    glfwWindow* m_Window;
    
    // 窗口数据
    struct WindowData {
        // ... 其他数据
        std::function<void(Event&)> EventCallback;
    };
    
    WindowData m_Data;

public:
    WindowsWindow() {
        // 创建 GLFW 窗口
        m_Window = glfwCreateWindow(...);
        
        // 关键：保存我们的数据到 GLFW
        glfwSetWindowUserPointer(m_Window, &m_Data);
        
        // 设置 GLFW 回调
        SetCallbacks();
    }
};
```

## 三. Lambda匿名函数

### 什么是 Lambda

**Lambda 表达式**（或称为匿名函数）是一种轻量级的函数定义方式，允许在代码中直接定义和使用函数，而无需为其命名。Lambda 通常用于需要短小函数的场景，如回调函数、算法中的自定义操作等。

### Lambda 的语法

一个完整的 Lambda 表达式的组成如下：
```txt
[ capture-list ] ( params ) mutable(optional) exception(optional) attribute(optional) -> ret(optional) { body } 
```

- **capture-list**：指定 Lambda 可以访问哪些外部变量。
- **params**：Lambda 的参数列表。
- **mutable**：允许修改捕获的变量（默认是不可变的）。
- **exception**：指定 Lambda 是否会抛出异常。
- **attribute**：为 Lambda 添加属性。
- **ret**：指定返回类型（通常可以省略，编译器会自动推断）。
- **body**：Lambda 的函数体。

### 我们为什么要用 Lambda

GLFW是**C库**，因此 GLFW 要求回调函数是**静态的或全局的**，不能是**成员函数**。

我们选择在 GLFW 回调中我们使用**无捕获lambda函数**
