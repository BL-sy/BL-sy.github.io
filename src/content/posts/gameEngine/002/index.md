---
title: 入口点与 DLL 导入导出实践
published: 2025-10-13
description: "基于cherno Hazel引擎教学"
tags: ["学习", "游戏", "引擎"]
category: 学习
draft: false
---


This blog template is built with [Astro](https://astro.build/). For the things that are not mentioned in this guide, you may find the answers in the [Astro Docs](https://docs.astro.build/).



# 一. 什么是入口点

在 C/C++ 应用程序中，“入口点” 是程序**开始执行的起始位置**，最常见的就是main()函数 —— 操作系统启动程序时，会自动找到并执行入口点函数，进而触发后续逻辑。

## 1.1 为什么不建议在游戏项目中写入口点？

我们曾在 Sandbox（游戏项目）中写过简单的入口点用于测试：

```cpp
// Sandbox项目中的临时入口点
void main() {
    Hazel::Print();
}
```

但这种写法存在明显问题：
- **应用程序无需关心引擎细节**：游戏项目（如 Sandbox）的核心是实现游戏逻辑（比如角色、关卡），而引擎的启动流程（如渲染模块初始化、窗口创建、内存分配）属于底层细节，不该让应用程序处理；
- **跨平台兼容性差**：不同操作系统的入口点可能不同（比如 Windows 是main()，部分嵌入式系统是entry()），若入口点写在游戏项目中，需为每个平台单独适配，而引擎统一管理入口可避免这一问题。

因此，**入口点应由引擎定义和提供**，游戏项目只需专注于自身逻辑。

## 1.2 引擎层定义 Application 类（入口核心载体）

为了让引擎管控入口流程，我们先在 Hazel 中定义Application类 —— 它是引擎的 “启动管理器”，封装初始化、主循环（Run()）等核心逻辑。

```cpp
//Hazel/Application.h

namespace Hazel {
    // 引擎核心启动类，封装入口相关逻辑
    class Application
    {
        Application();
        virtual ~Application();

        void Run();
    };
}
```

```cpp
//Hazel/Application.cpp

#include "Application.h"
namespace Hazel
{
    Application::Application()
    {
    }

    Application::~Application()
    {
    }

    void Application::Run()
    {
        while (true);
    }
}
```


# 二. 动态链接库DLL(Hazel)的导入导出

Hazel 作为动态库（DLL），需要与 Sandbox（游戏 EXE）共享Application类等资源 —— 
这就需要*__declspec(dllimport)*和*__declspec(dllexport)*两个关键字，实现 “跨模块资源交互”。

## 2.1 两个关键字的作用

__declspec(dllexport)：用于**导出**DLL 中的类 / 函数 / 数据（比如 Hazel 的Application类），告诉编译器 “这些资源要对外提供”；
__declspec(dllimport)：用于**导入**DLL 中的资源（比如 Sandbox 用Application类），告诉编译器 “这些资源在外部 DLL 中定义”。

简单说，它们是 DLL 和 EXE 之间的 “桥梁”，确保资源能正确共享，避免编译时 “未定义符号” 错误。

## 2.2 用宏定义统一管理（避免重复代码）

若每个需要导出的类 / 函数前都写__declspec(...)，会非常繁琐。我们可以通过**条件编译宏**HAZEL_API，让编译器自动切换 “导入 / 导出” 模式。

### 核心宏定义文件

我们先在Hazel中新建一个Hazel文件夹，存放核心文件，在这个文件夹中新建一个Core.h。
我们将在Core中宏定义**HAZEL_API**，用来区分是导入还是导出

- 在Core.h中

  ```cpp
  #pragma once
  // 仅支持Windows平台（后续可扩展其他平台）
  #ifdef HZ_PLATFORM_WINDOWS
    #ifdef HZ_BUILD_DLL
        #define HAZEL_API __declspec(dllexport)
    #else
        #define HAZEL_API __declspec(dllimport)
    #endif
  #else
    #error Hazel only supports Windows!
  #endif
  ```

### 配置预处理器宏
  
需为 Hazel 和 Sandbox 项目分别配置宏，确保HAZEL_API生效：

Hazel 项目：右键项目→「属性→C/C++→预处理器→预处理器定义」，添加HZ_BUILD_DLL;HZ_PLATFORM_WINDOWS；

Sandbox 项目：同上路径，仅添加HZ_PLATFORM_WINDOWS（无需HZ_BUILD_DLL，因为是导入方）

### 导出 Application 类（修改 //Hazel/Application.h）

在Application类前添加HAZEL_API，实现引擎类的导出：

  ```cpp
  #include "Core.h"

  namespace Hazel {
      // 用HAZEL_API标记类，自动适配导入/导出
      class HAZEL_API Application
      {
          Application();
          virtual ~Application();

          void Run();
      };
  }
  ```

## 2.3 简化头文件引用（让游戏项目更易用）

若 Sandbox 项目直接写#include "Hazel/Application.h"，不仅路径冗长，还容易因目录调整导致引用错误。
我们可以创建 “统一入口头文件”Hazel.h，让游戏项目只需引用这一个文件。

### 统一入口头文件（$Hazel.h）

在 Hazel 引擎根目录下新建Hazel.h，内部包含核心头文件：

```cpp
//Hazel.h

// For use by Hazel applications
#include "Hazel/Application.h"
```

### 配置 Sandbox 的包含目录
让 Sandbox 能找到Hazel.h：右键 Sandbox 项目→「属性→C/C++→常规→附加包含目录」，添加$(SolutionDir)Hazel/src（$(SolutionDir)是解决方案根目录，确保路径不硬编码）。

### 游戏项目中使用 Application 类
此时 Sandbox 可直接通过#include <Hazel.h>使用引擎类，代码更简洁：

```cpp
//Sandbox/src/Application.cpp
#include <Hazel.h>

// 游戏项目自定义类，继承引擎的Application
class Sandbox : public Hazel::Application
{
public:
  Sandbox() {}   // 自定义构造逻辑（后续扩展）
  ~Sandbox() {}  // 自定义析构逻辑（后续扩展）
};

// 临时入口点（后续会迁移到引擎）
void main()
{
  Hazel::Application* app = new Sandbox();
  app->Run();    // 调用引擎主循环
  delete app;    // 释放资源
}
```

# 三. 统一入口点：迁移 main 函数到引擎

虽然 Sandbox 能使用Application类，但入口点（main()）仍在游戏项目中 —— 不符合 “引擎管控入口” 的初衷。
我们需要将main()迁移到 Hazel，并通过 “工厂函数” 让游戏项目提供自定义Application实例。

## 3.1 核心思路：用工厂函数实现 “灵活适配”

引擎无法预知游戏项目会定义怎样的Application派生类（比如 Sandbox、DemoGame 等），因此：

- 引擎声明一个工厂函数CreateApplication()（仅声明，不实现）；
- 游戏项目（Sandbox）实现这个函数，返回自定义的Sandbox实例；
- 引擎的main()函数调用CreateApplication()获取实例，启动主循环。

## 3.2 具体实现步骤

### 引擎声明工厂函数（修改 //Hazel/Application.h）

在Hazel命名空间中添加CreateApplication()的声明，注明 “由客户端（游戏项目）实现”：

```cpp
//Hazel/Application.h
#include "Core.h"

namespace Hazel {
  // 工厂函数：由游戏项目实现，返回自定义Application实例
  // To be defined in CLIENT (e.g. Sandbox)
  Application* CreateApplication();
}
```

### 引擎定义统一入口（新建 //Hazel/EntryPoint.h）

在 Hazel 中新建EntryPoint.h，编写main()函数 —— 这是引擎的统一入口，同时支持命令行参数（argc/argv，为后续扩展预留）：

```cpp
//Hazel/EntryPoint.h

#ifdef HZ_PLATFORM_WINDOWS

// 声明工厂函数（与Application.h中的声明对应）
extern Hazel::Application* Hazel::CreateApplication();

int main(int argc, char** argv) {
    // 调用游戏项目实现的工厂函数，获取自定义实例
    auto app = Hazel::CreateApplication();
    app->Run();
    delete app;
}

#endif
```

### 游戏项目实现工厂函数（//Sandbox/SandboxApp.cpp）

Sandbox 实现CreateApplication()，返回Sandbox实例：

```cpp
//Sandbox/SandboxApp.cpp
// 实现引擎声明的工厂函数
Hazel::Application* Hazel::CreateApplication() 
{
    return new Sandbox();
}
```

### 整合入口头文件（修改 $Hazel.h）

在Hazel.h中包含EntryPoint.h，让游戏项目引用Hazel.h时，自动引入引擎的入口逻辑（无需额外引用EntryPoint.h）：

```cpp
//Hazel.h

#pragma once
// For use by Hazel applications

#include "Hazel/Application.h"

// -----EntryPoint----------------------
#include "Hazel/EntryPoint.h"
// ---------------------------------------
```

此时 Sandbox 项目无需编写main()函数，只需实现CreateApplication()和自定义Sandbox类 —— 引擎通过统一入口启动程序，既管控了底层启动逻辑（如后续添加初始化、错误处理），又保留了游戏项目的灵活性。
