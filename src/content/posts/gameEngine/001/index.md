---
title: 初识游戏引擎
published: 2025-10-13
description: "基于 Cherno Hazel 引擎的入门教学，涵盖引擎核心概念、项目搭建、动态链接实践与链接方式对比，适合游戏开发新手"
tags: ["游戏开发", "Hazel 引擎", "C++", "引擎入门", "Visual Studio"]
category: 学习
draft: false
---



This blog template is built with [Astro](https://astro.build/). For unmentioned details, refer to the [Astro Docs](https://docs.astro.build/).

# 一、引言

本文基于 Cherno 的 Hazel 引擎教学系列，面向游戏开发初学者，系统讲解：



1. 游戏引擎的核心定义与组成模块

2. Hazel 引擎（动态库）与 Sandbox 游戏项目的工程搭建

3. 动态 / 静态链接的原理与对比

   帮助你快速掌握引擎开发基础，建立规范的工程化思维。


# 二、什么是游戏引擎？

游戏引擎是**开发游戏的底层工具框架**（如 Unity、Unreal 等商业引擎），核心作用是：

`读取资产文件（模型/纹理）→ 转换格式 → 渲染到屏幕 → 提供交互能力`

让开发者无需从零实现渲染、物理等复杂功能，专注于游戏逻辑与内容创作。

## 2.1 游戏引擎的核心模块



| 模块（Attribute）             | 功能描述（Description）                           |
| ------------------------- | ------------------------------------------- |
| `Entry Point`             | 程序启动入口，初始化渲染、内存等核心模块，触发主循环                  |
| `Application Layer`       | 应用层，封装引擎接口，协调场景管理、生命周期控制等工作                 |
| `Window Layer`            | 系统窗口层，处理鼠标 / 键盘输入、窗口事件分发                    |
| `Renderer`                | 渲染器，将资产转换为屏幕像素，实现光照、纹理等视觉效果                 |
| `Render API Abstraction`  | 渲染 API 抽象层，兼容 DirectX/Vulkan/OpenGL，实现跨平台渲染 |
| `Debugging Support`       | 调试支持，提供日志、帧率监控、内存分析工具，定位错误与性能瓶颈             |
| `Scripting Language`      | 脚本语言（如 Lua/C#），快速编写游戏逻辑，无需修改引擎源码            |
| `Memory System`           | 内存系统，负责分配 / 释放 / 复用（如内存池），避免泄漏与碎片           |
| `Entity-Component System` | ECS 系统，通过 “实体 + 组件 + 系统” 解耦对象设计，提升开发灵活性     |
| `Physics`                 | 物理系统，模拟重力、碰撞、运动等现实物理规则（如角色跳跃、物体坠落）          |
| `File I/O`                | 文件输入输出，支持虚拟文件系统（VFS），统一管理本地 / 压缩包 / 网络资源    |
| `Build System`            | 构建系统，自动化编译、链接、打包，支持 Windows/Linux/macOS 多平台 |


# 三、Hazel 引擎项目设置

本节讲解如何搭建 “引擎 + 游戏” 的工程结构，养成规范开发习惯。


## 3.1 基础工程规范

### 3.1.1 指定单独的输出目录

在项目属性中设置独立编译目录（如`bin/Debug-windows-x86_64`），避免产物与源码混杂，目录结构示例：

```
Hazel-Engine/
├─ bin/          # 编译输出目录
│  ├─ Debug-windows-x86_64/     # Debug模式：.exe/.dll/.lib
│  └─ Release-windows-x86_64/   # Release模式：优化后的产物
│  └─ Dist-windows-x86_64/   # Dist模式：发行版本
├─ src/          # 源码目录
│  ├─ Hazel/     # 引擎源码
│  └─ Sandbox/   # 游戏项目源码
└─ Hazel.sln     # Visual Studio解决方案
```

![020](../assets/020.webp)


### 3.1.2 新建`src`文件夹存储源码

将所有代码集中在`src`目录，按 “引擎（Hazel）” 和 “游戏（Sandbox）” 拆分，提升可维护性。


## 3.2 项目属性配置

### 3.2.1 创建两个核心项目

在 Visual Studio 中新建 C++ 项目，分工如下：



* **Hazel**：配置为「动态库（DLL）」，提供引擎核心功能

* **Sandbox**：配置为「控制台应用（EXE）」，基于 Hazel 开发游戏逻辑

### 3.2.2 设置启动项目


1. 打开`Hazel.sln`解决方案

2. 右键`Sandbox` → 「设为启动项目」，确保调试时优先运行游戏

这里用记事本打开Solusion,更改默认启动项目

![019](../assets/019.webp)

### 3.2.3 链接 Hazel 引擎到 Sandbox


 右键`Sandbox` → 「项目依赖项」→ 勾选`Hazel`（确保先编译引擎）


> 关键说明：

动态库（DLL）为何需要`.lib`文件？


* `.lib`：导入库，存储 DLL 中导出函数的 “名称 + 内存地址索引”（无实际逻辑）

* `.dll`：包含函数实现与数据

* Sandbox 编译时通过`.lib`定位函数入口，运行时加载`.dll`执行



# 四、项目链接测试

通过简单函数调用，验证 Sandbox 是否成功链接 Hazel 引擎。

## 4.1 代码实现

### 4.1.1 Hazel 引擎：导出测试函数



```
//Hazel/src/Test.h
//#include <stdio.h>
// 约定：所有Hazel代码均在Hazel命名空间下

namespace Hazel {
    __declspec(dllexport)：标记函数为DLL导出（供外部调用）
    __declspec(dllexport) void Print();
}
```



```
//Hazel/src/Test.cpp

#include "Test.h"

namespace Hazel {
   void Print() {
       printf("Hazel Engine Link Success! \n");
   }
}
```

### 4.1.2 Sandbox 项目：导入并调用函数



```
//Sandbox/src/Application.cpp

namespace Hazel {
   //__declspec(dllimport)：标记函数为DLL导入（从Hazel.dll获取）
   __declspec(dllimport) void Print();
}

// 标准C++入口（避免void main()）

int main() {
   Hazel::Print();  // 调用引擎函数
   system("pause"); // 暂停控制台查看结果
   return 0;
}
```

## 4.2 测试步骤


1. 编译 Hazel：生成`Hazel.dll`和`Hazel.lib`（路径：`bin/Debug`）

2. 复制`Hazel.dll`到 Sandbox 输出目录（`bin/Debug/Sandbox`）

3. 运行 Sandbox：控制台输出`Hazel Engine Link Success! `，证明链接成功

# 五、核心知识：静态链接 vs 动态链接



| 对比维度 | 静态链接（Static Linking）  | 动态链接（Dynamic Linking）       |
| ---- | --------------------- | --------------------------- |
| 原理   | 编译时将库函数嵌入`exe`，运行时无依赖 | 编译时仅存索引，运行时加载`.dll`/`.so`链接 |
| 执行速度 | 快（无运行时加载）             | 慢（启动时加载链接）                  |
| 资源占用 | 高（多 exe 冗余副本）         | 低（多 exe 共享库）                |
| 更新维护 | 繁琐（库改则所有 exe 重编）      | 便捷（仅替换.dll）                 |
| 部署依赖 | 无（exe 独立运行）           | 有（需携带.dll）                  |

> Hazel 选择动态链接的原因：


1. 引擎更新时，仅需替换`Hazel.dll`，无需重新编译 Sandbox

2. 多个游戏项目（如 Sandbox1、Sandbox2）可共享同一`Hazel.dll`，节省空间


# 六、后续优化方向


1. **自动复制 DLL**：通过 Preamke5 配置，编译后自动复制`Hazel.dll`到 Sandbox 目录

2. **统一头文件**：将 Hazel 公共头文件整理到`include`目录，Sandbox 通过 “附加包含目录” 引用

3. **Debug 宏定义**：添加`HAZEL_DEBUG`宏，区分 Debug（日志输出）与 Release（性能优化）模式
