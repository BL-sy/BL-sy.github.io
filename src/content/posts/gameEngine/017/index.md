---
title: 上下文渲染封装与三角形
published: 2025-11-10
description: "基于cherno Hazel引擎教学"
tags: ["学习", "游戏", "引擎"]
category: 学习
draft: false
---


This blog template is built with [Astro](https://astro.build/). For the things that are not mentioned in this guide, you may find the answers in the [Astro Docs](https://docs.astro.build/).


# 一、封装目的

在引擎开发中，图形上下文（如 OpenGL 上下文）负责管理渲染 API 的初始化、缓冲区交换等核心操作。传统实现中，GLFW 窗口创建、Glad 加载、缓冲区交换等逻辑分散在`Window`和`Application`中，导致渲染 API 与上层逻辑强耦合，后续扩展 Vulkan/DirectX 时需大量修改代码。

本次封装通过抽象`GraphicsContext`基类，统一不同渲染 API 的核心接口，将 GLFW+Glad 的底层逻辑收拢到`OpenGLContext`实现类中，实现：



* 解耦渲染 API 与上层逻辑，支持后续无缝切换渲染后端；

* 统一上下文初始化、缓冲区交换接口，降低维护成本；

* 收拢底层逻辑，避免代码分散，提升调试效率。

# 二、核心抽象：GraphicsContext 基类

定义纯虚接口，屏蔽不同渲染 API 的实现差异，上层代码仅依赖基类接口。

```cpp
namespace Hazel {
    // 图形上下文基类：统一渲染API的核心接口
    class GraphicsContext
    {
    public:
         // 初始化上下文（如Glad加载、渲染状态配置）
         virtual void Init() = 0;
         // 交换前后缓冲区（双缓冲机制）
         virtual void Swap() = 0;
    };
}
```

## 关键说明：

* 采用纯虚接口设计，强制子类实现`Init`和`Swap`，确保接口统一性；

* `windowHandle`设计为`void*`，兼容 GLFWwindow*/HWND 等不同窗口库句柄，增强通用性。

# 三、OpenGL 实现：OpenGLContext 类

继承`GraphicsContext`，封装 GLFW 窗口绑定、Glad 加载、缓冲区交换等底层逻辑。

## 3.1 头文件（Hazel/src/Hazel/Renderer/OpenGL/OpenGLContext.h）

```cpp
#include "Hazel/Renderer/GraphicsContext.h"

// 前向声明GLFW窗口句柄，避免头文件暴露依赖
struct GLFWwindow;

namespace Hazel {
    // OpenGL上下文实现：封装GLFW+Glad的底层逻辑
    class OpenGLContext : public GraphicsContext
    {
    public:
         // 构造函数：接收GLFW窗口句柄
         OpenGLContext(GLFWwindow* windowHandle);

         // 实现基类纯虚接口
         virtual void Init() override;
         virtual void Swap() override;

    private:
         GLFWwindow* m_WindowHandle = nullptr; // 持有GLFW窗口句柄
    };
}
```

## 3.2 实现文件（Hazel/src/Hazel/Renderer/OpenGL/OpenGLContext.cpp）

```cpp
#include "hzpch.h"

#include "OpenGLContext.h"

#include <GLFW/glfw3.h>
#include <glad/glad.h>

namespace Hazel {
    OpenGLContext::OpenGLContext(GLFWwindow* windowHandle)
         : m_WindowHandle(windowHandle)
    {
         HZ_CORE_ASSERT(windowHandle, "GLFW窗口句柄为空，无法创建OpenGL上下文！");
    }

    // 初始化：绑定窗口上下文+加载Glad
    void OpenGLContext::Init()
    {
         // 1. 绑定当前线程到GLFW窗口上下文
         glfwMakeContextCurrent(m_WindowHandle);

         // 2. 加载Glad（OpenGL函数指针）
         int gladStatus = gladLoadGLLoader((GLADloadproc)glfwGetProcAddress);
         HZ_CORE_ASSERT(gladStatus, "Glad初始化失败！请检查显卡驱动或OpenGL环境");
    }

    // 交换缓冲区：调用GLFW双缓冲交换接口
    void OpenGLContext::Swap()
    {
         glfwSwapBuffers(m_WindowHandle);
    }
}
```

# 四、Window 类修改：集成图形上下文

`Window`类不再直接处理底层渲染逻辑，改为通过基类接口`GraphicsContext`调用，解耦 OpenGL 依赖。


- WindowsWindow 实现修改（Hazel/src/Hazel/Platform/Windows/WindowsWindow.cpp）

仅展示核心修改部分，保留原有 GLFW 窗口创建、事件回调逻辑：



```cpp
#include "hzpch.h"

#include "WindowsWindow.h"

#include "Platform/OpenGL/OpenGLContext.h"

namespace Hazel {
    void WindowsWindow::Init(const WindowProps& props)
    {
         m_Data.Title = props.Title;
         m_Data.Width = props.Width;
         m_Data.Height = props.Height;

         // 原有：GLFW初始化+窗口创建（不变）
         HZ_CORE_INFO("创建窗口：{} ({}, {})", props.Title, props.Width, props.Height);
         if (!glfwInit())
         {
              HZ_CORE_FATAL("GLFW初始化失败！");
              return;
         }

         m_Window = glfwCreateWindow((int)props.Width, (int)props.Height, m_Data.Title.c_str(), nullptr, nullptr);

         // 新增：创建并初始化图形上下文
         m_Context = GraphicsContext::Create(m_Window);
         m_Context->Init(); // 调用OpenGLContext::Init加载Glad

         // 原有：VSync设置+事件回调（不变）
         glfwSetWindowUserPointer(m_Window, \&m_Data);
         SetVSync(true);
         SetupGLFWCallbacks(m_Window);
    }

    void WindowsWindow::OnUpdate()
    {
         glfwPollEvents(); // 原有：窗口事件轮询（不变）
         m_Context->Swap(); // 修改：缓冲区交换交给上下文处理
    }
}
```


# 六、封装核心优势



1. **渲染 API 解耦**：上层代码仅依赖`GraphicsContext`基类，后续扩展 Vulkan/DirectX 时，只需新增`VulkanContext`/`DirectXContext`类，无需修改`Window`和`Application`；

2. **接口统一**：无论哪种渲染 API，均通过`Init()`和`Swap()`接口调用，降低跨 API 开发的学习成本；

3. **逻辑收拢**：Glad 加载、上下文绑定等底层逻辑集中在`OpenGLContext`，避免代码分散，便于问题定位；

4. **兼容性强**：`windowHandle`采用`void*`设计，兼容 GLFW、SDL 等不同窗口库，后续替换窗口库改动极小。

# 七、后续扩展方向



1. **多渲染 API 支持**：在工厂函数中通过预编译宏（如`HZ_RENDERER_VULKAN`）切换上下文实现，支持动态选择渲染后端；

2. **接口扩展**：在`GraphicsContext`中新增`SetClearColor`、`Clear`等接口，将清屏逻辑也封装到上下文，彻底移除上层对 OpenGL 头文件的依赖；

3. **资源管理**：添加上下文销毁接口`Destroy()`，统一管理渲染资源释放，避免内存泄漏；

4. **跨平台适配**：针对 Linux/macOS，扩展`X11Context`/`CocoaContext`，保持基类接口统一。

> （注：文档部分内容可能由 AI 生成）