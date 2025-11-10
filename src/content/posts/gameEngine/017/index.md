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
         glfwSetWindowUserPointer(m_Window, &m_Data);
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


# 五、绘制第一个三角形

## 核心目标

通过 OpenGL 的 VAO（顶点数组对象）、VBO（顶点缓冲对象）、IBO（索引缓冲对象）和基础 Shader，实现三角形渲染，理解现代 OpenGL 渲染管线的核心流程，为后续复杂渲染器的创建提供思路。

要绘制三角形，需串联以下 5 个核心组件，遵循 “状态绑定 + 数据传输 + 着色器驱动” 的渲染逻辑：



1. **VAO（Vertex Array Object）**：管理顶点属性布局（如位置、颜色），记录 VBO 与顶点属性的关联关系；

2. **VBO（Vertex Buffer Object）**：GPU 显存中的缓冲区，存储顶点数据（如三维坐标）；

3. **IBO（Index Buffer Object）**：存储顶点索引，通过索引复用顶点，减少数据传输开销；

4. **Shader（着色器）**：运行在 GPU 上的程序，分为顶点着色器（处理顶点位置）和片段着色器（处理像素颜色），是渲染的核心驱动；（我们先使用OpenGL自带的渲染管线）

5. **渲染流程**：绑定 VAO→激活 Shader→调用绘制函数→交换缓冲区。

## 代码修改与补充：完整实现三角形渲染

在`Application.cpp`中添加 Shader 编译、链接的工具函数，以及顶点着色器、片段着色器源码（基础功能，仅实现位置传递和固定颜色）：


```cpp
#include "hzpch.h"

#include "Application.h"
#include "Hazel/Log.h"
#include <glad/glad.h>  // 依赖OpenGL头文件

namespace Hazel {

    Application* Application::s_Instance = nullptr;

    Application::Application()
    {
        HZ_CORE_ASSERT(!s_Instance, "Application already exists!");
        s_Instance = this;

        // 1. 创建窗口（原有逻辑不变）
        m_Window = std::unique_ptr<Window>(Window::Create());
        m_Window->SetEventCallback(BIND_EVENT_FN(OnEvent));

        // 2. 创建ImGuiLayer（原有逻辑不变）
        m_ImGuiLayer = new ImGuiLayer();
        PushOverlay(m_ImGuiLayer);

        // 3. 渲染初始化：VAO、VBO、IBO（用户原有代码，补充Shader创建）
        // 3.1 顶点数据（三角形三个顶点的三维坐标）
        float vertices[3 * 3] = {
            -0.5f, -0.5f, 0.0f,  // 顶点0：左下
             0.5f, -0.5f, 0.0f,  // 顶点1：右下
             0.0f,  0.5f, 0.0f   // 顶点2：上中
        };

        // 3.2 索引数据（复用顶点，按顺序绘制三角形）
        unsigned int indices[3] = { 0, 1, 2 };

        // 3.3 创建VAO（管理顶点属性）
        glGenVertexArrays(1, &m_VertexArray);
        glBindVertexArray(m_VertexArray);

        // 3.4 创建VBO（存储顶点数据）
        glGenBuffers(1, &m_VertexBuffer);
        glBindBuffer(GL_ARRAY_BUFFER, m_VertexBuffer);
        glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);  // 静态数据（不频繁修改）

        // 3.5 配置顶点属性（位置属性：索引0，3个float，无归一化，步长3*float，偏移0）
        glEnableVertexAttribArray(0);
        glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), nullptr);

        // 3.6 创建IBO（存储索引数据）
        glGenBuffers(1, &m_IndexBuffer);
        glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, m_IndexBuffer);
        glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices), indices, GL_STATIC_DRAW);
    }

    // 主循环（用户原有代码）
    void Application::Run()
    {
        while (m_Running)
        {
            // 1. 清屏（原有逻辑不变）
            glClearColor(0.1f, 0.1f, 0.1f, 1.0f);
            glClear(GL_COLOR_BUFFER_BIT);

            // 2. 渲染三角形（核心修改：绑定VAO+绘制）
            glBindVertexArray(m_VertexArray);  // 绑定VAO（自动关联VBO、IBO和顶点属性）
            glDrawElements(GL_TRIANGLES, 3, GL_UNSIGNED_INT, nullptr);  // 绘制三角形（3个索引）

            // 3. Layer更新+ImGui渲染（原有逻辑不变）
            for (Layer* layer : m_LayerStack)
                layer->OnUpdate();

            if (m_ImGuiLayer)
            {
                m_ImGuiLayer->Begin();
                for (Layer* layer : m_LayerStack)
                    layer->OnImGuiRender();
                m_ImGuiLayer->End();
            }

            // 4. 交换缓冲区（原有逻辑不变）
            m_Window->OnUpdate();
        }

        // 程序退出时，释放OpenGL资源（新增）
        glDeleteVertexArrays(1, &m_VertexArray);
        glDeleteBuffers(1, &m_VertexBuffer);
        glDeleteBuffers(1, &m_IndexBuffer);
    }
    // 原有事件处理逻辑（OnEvent、OnWindowClose等）不变
}
```

## 核心流程解析：三角形渲染的完整链路

### 3.1 初始化阶段（Application 构造函数）



1. **创建 Shader 程序**：

* 编写顶点着色器（传递顶点位置到 GPU）和片段着色器（输出固定橙色）；

* 编译单个着色器，链接为 Shader 程序，存储程序 ID（`m_ShaderProgram`）。

2. **创建 VAO/VBO/IBO**：

* VAO：生成并绑定，后续顶点属性配置会记录到 VAO 中；

* VBO：生成并绑定，将 CPU 端的顶点数据（`vertices`）上传到 GPU 显存；

* 配置顶点属性：告诉 OpenGL 顶点数据的布局（3 个 float 为一个位置，无偏移，步长 3*float）；

* IBO：生成并绑定，上传索引数据（`indices`），指定顶点绘制顺序。

### 3.2 渲染阶段（Application::Run 主循环）



1. **清屏**：清除颜色缓冲区，避免上一帧画面残留；

2. **激活 Shader**：`glUseProgram(m_ShaderProgram)`，后续绘制会使用该 Shader；

3. **绑定 VAO**：`glBindVertexArray(m_VertexArray)`，自动关联 VBO、IBO 和顶点属性配置；

4. **绘制三角形**：`glDrawElements(GL_TRIANGLES, 3, GL_UNSIGNED_INT, nullptr)`，OpenGL 根据索引顺序绘制 3 个顶点组成的三角形；

5. **ImGui 渲染 + 缓冲区交换**：保持原有逻辑，最终将渲染结果显示到窗口。

### 3.3 资源释放阶段（Application 析构函数）

释放 VAO、VBO、IBO 和 Shader 程序的 GPU 资源，避免内存泄漏（OpenGL 对象需手动释放）。


## 运行效果

编译并运行 Sandbox 项目，窗口将显示一个**三角形**，背景为深灰色，ImGui Demo 窗口正常显示（可拖拽悬靠），三角形居中显示，窗口缩放时三角形会自适应大小（因视口已同步更新）。

![031](../assets/031.webp)