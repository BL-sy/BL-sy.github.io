---
title: 事件框架
published: 2025-10-17
description: "基于cherno Hazel引擎教学"
tags: ["学习", "游戏", "引擎"]
category: 学习
draft: false
---


This blog template is built with [Astro](https://astro.build/). For the things that are not mentioned in this guide, you may find the answers in the [Astro Docs](https://docs.astro.build/).


# 事件系统

我们的游戏引擎需要一个窗口，用户在窗口上的操作应该被传递到应用程序，我们使用事件系统完成这个任务，我们先来设计一下可能的事件。

# 一. 规划

以 glfwSetCursorPosCallback 为例，鼠标移动时会调用先前注册好的回调函数，并传入xpos和ypos。而作为接收事件的Application类，最需要的就是xpos和ypos，以进行之后更多的逻辑处理。因此对于鼠标移动这个事件来说，事件系统的任务就是把xpos和ypos从Window类传递到Application类。

我们的事件：

![023](../assets/023.webp)

# 二. 事件框架

## 基事件

我们首先创建一个Event文件夹，用于存储所有的事件处理文件，我们首先处理一个基事件头

```cpp
$Event.h

#include "hzpch.h"
#include "Hazel/Core.h"

namespace Hazel {
	// Hazel 中的事件当前处于阻塞状态，这意味着当事件发生时
	// 立即被调度并且必须立即处理。
	// 对于未来，更好的策略可能是在事件总线中缓冲事件
	// 在更新阶段的“事件”部分对它们进行排队和处理。

	// 事件的所有类型
	enum class EventType
	{
		None = 0,
		WindowClose, WindowResize, WindowFocus, WindowLostFocus, WindowMoved,
		AppTick, AppUpdate, AppRender,
		KeyPressed, KeyReleased,
		MouseButtonPressed, MouseButtonReleased, MouseMoved, MouseScrolled
	};

	// 用于对事件进行分类的位掩码，可以将事件归类为多个类别
	// 用于快速过滤事件
	enum EventCategory
	{
		None = 0,
		EventCategoryApplication = BIT(0),
		EventCategoryInput = BIT(1),
		EventCategoryKeyboard = BIT(2),
		EventCategoryMouse = BIT(3),
		EventCategoryMouseButton = BIT(4)
	};

// 宏定义用于简化事件的类型和类别定义
// 静态类型用于在不创建实例的情况下获取事件类型
// 动态类型用于获取实例的事件类型
// 名称用于调试和日志记录
#define EVENT_CLASS_TYPE(type) static EventType GetStaticType() { return EventType::type; }\
								virtual EventType GetEventType() const override { return GetStaticType(); }\
								virtual const char* GetName() const override { return #type; }
#define EVENT_CLASS_CATEGORY(category) virtual int GetCategoryFlags() const override { return category; }

	class HAZEL_API Event
	{
		// 事件分发器
		friend class EventDispatcher;
	public:
		virtual EventType GetEventType() const = 0;
		virtual const char* GetName() const = 0;
		virtual int GetCategoryFlags() const = 0;
		virtual std::string ToString() const { return GetName(); }

		// 判断事件是否属于某个类别
		inline bool IsInCategory(EventCategory category)
		{
			return GetCategoryFlags() & category;
		}
	protected:
		// 判断事件是否被处理
		// 事件被处理后就不会再传递给其他层
		bool Handled = false;
	};

	// 重载输出运算符，方便打印事件信息
	inline std::ostream& operator<<(std::ostream& os, const Event& e)
	{
		return os << e.ToString();
	}
}

```
**关键**：事件类型的宏定义

这个宏定义包含了**都是返回事件类型的静态函数，虚函数**以及一个返回类型名称的函数

原因：静态事件类型层想要处理的事件类型，动态事件类型是该事件的类型，通过静态函数和虚函数的组合便于**事件分发器识别事件类型**

## 键盘事件

键盘事件注意处理持续按键，想象我们持续按住a，会先显示1个a，过了一小会儿，再持续打印一串a

```cpp
$KeyEvent.h

#include "Hazel/Core.h"
#include "Event.h"

namespace Hazel {
	// 键盘事件基类
	class HAZEL_API KeyEvent : public Event
	{
	public:
		inline int GetKeyCode() const { return m_KeyCode; }

		EVENT_CLASS_CATEGORY(EventCategoryKeyboard | EventCategoryInput)
	protected:
		KeyEvent(int keycode)
			: m_KeyCode(keycode) {}
		int m_KeyCode;
	};

	// 键盘按下事件
	class HAZEL_API KeyPressedEvent : public KeyEvent
	{
	public:
		// repeatCount 表示按键重复的次数（用于处理按键长按的情况）
		KeyPressedEvent(int keycode, int repeatCount)
			: KeyEvent(keycode), m_RepeatCount(repeatCount) {}
		inline int GetRepeatCount() const { return m_RepeatCount; }
		std::string ToString() const override
		{
			std::stringstream ss;
			ss << "KeyPressedEvent: " << m_KeyCode << " (" << m_RepeatCount << " repeats)";
			return ss.str();
		}
		// 宏定义，定义事件的类型
		// 通过该宏可以获取事件的静态类型、动态类型和名称
		EVENT_CLASS_TYPE(KeyPressed)
	private:
		int m_RepeatCount;
	};

	// 键盘释放事件
	class HAZEL_API KeyReleasedEvent : public KeyEvent
	{
	public:
		KeyReleasedEvent(int keycode)
			: KeyEvent(keycode) {}
		std::string ToString() const override
		{
			std::stringstream ss;
			ss << "KeyReleasedEvent: " << m_KeyCode;
			return ss.str();
		}

		EVENT_CLASS_TYPE(KeyReleased)
	};
}
```

## 鼠标事件

```cpp
$MouseEvent.h

#include "Hazel/Core.h"

#include "Event.h"

namespace Hazel {
	
	class HAZEL_API MouseMovedEvent : public Event
	{
	public:
		MouseMovedEvent(float x, float y)
			: m_MouseX(x), m_MouseY(y) {}

		inline float GetX() const { return m_MouseX; }
		inline float GetY() const { return m_MouseY; }

		std::string ToString() const override
		{
			std::stringstream ss;
			ss << "MouseMovedEvent: " << m_MouseX << ", " << m_MouseY;
			return ss.str();
		}
			EVENT_CLASS_TYPE(MouseMoved)
			EVENT_CLASS_CATEGORY(EventCategoryMouse | EventCategoryInput)
	private:
		float m_MouseX, m_MouseY;
	};

	class HAZEL_API MouseScrolledEvent : public Event
	{
	public:
		MouseScrolledEvent(float xOffset, float yOffset)
			: m_XOffset(xOffset), m_YOffset(yOffset) {}

		inline float GetXOffset() const { return m_XOffset; }
		inline float GetYOffset() const { return m_YOffset; }

		std::string ToString() const override
		{
			std::stringstream ss;
			ss << "MouseScrolledEvent: " << GetXOffset() << ", " << GetYOffset();
			return ss.str();
		}

		EVENT_CLASS_TYPE(MouseScrolled)
		EVENT_CLASS_CATEGORY(EventCategoryMouse | EventCategoryInput)
	private:
		float m_XOffset, m_YOffset;
	};

	// 鼠标按钮事件基类
	class HAZEL_API MouseButtonEvent : public Event
	{
	public:
		inline int GetMouseButton() const { return m_Button; }
		EVENT_CLASS_CATEGORY(EventCategoryMouse | EventCategoryInput | EventCategoryMouseButton)
	protected:
		MouseButtonEvent(int button)
			: m_Button(button) {}
		int m_Button;
	};

	// 鼠标按钮按下事件
	class HAZEL_API MouseButtonPressedEvent : public MouseButtonEvent
	{
	public:
		MouseButtonPressedEvent(int button)
			: MouseButtonEvent(button) {}
		std::string ToString() const override
		{
			std::stringstream ss;
			ss << "MouseButtonPressedEvent: " << m_Button;
			return ss.str();
		}
		EVENT_CLASS_TYPE(MouseButtonPressed)
	};

	// 鼠标按钮释放事件
	class HAZEL_API MouseButtonReleasedEvent : public MouseButtonEvent
	{
	public:
		MouseButtonReleasedEvent(int button)
			: MouseButtonEvent(button) {}
		std::string ToString() const override
		{
			std::stringstream ss;
			ss << "MouseButtonReleasedEvent: " << m_Button;
			return ss.str();
		}
		EVENT_CLASS_TYPE(MouseButtonReleased)
	};
}
```

## 应用程序（窗口）事件

```cpp
$ApplicationEvent.h

#include "Hazel/Core.h"
#include "Event.h"

namespace Hazel {

	// 窗口关闭事件
	class HAZEL_API WindowCloseEvent : public Event
	{
	public:
		WindowCloseEvent() {}

		EVENT_CLASS_TYPE(WindowClose)
		EVENT_CLASS_CATEGORY(EventCategoryApplication)
	};

	// 窗口点击事件
	class HAZEL_API AppTickEvent : public Event
	{
	public:
		AppTickEvent() {}

		EVENT_CLASS_TYPE(AppTick)
		EVENT_CLASS_CATEGORY(EventCategoryApplication)
	};

	// 窗口大小改变事件
	class HAZEL_API WindowResizeEvent : public Event
	{
	public:
		WindowResizeEvent(int width, int height)
			: m_Width(width), m_Height(height) {
		}

		inline int GetWidth() const { return m_Width; }
		inline int GetHeight() const { return m_Height; }

		std::string ToString() const override
		{
			std::stringstream ss;
			ss << "WindowResizeEvent: " << m_Width << ", " << m_Height;
			return ss.str();
		}
		EVENT_CLASS_TYPE(WindowResize)
		EVENT_CLASS_CATEGORY(EventCategoryApplication)

	private:
		int m_Width, m_Height;
	};

	// 窗口更新事件
	class HAZEL_API AppUpdateEvent : public Event
	{
	public:
		AppUpdateEvent() {}

		EVENT_CLASS_TYPE(AppUpdate)
			EVENT_CLASS_CATEGORY(EventCategoryApplication)
	};

	// 窗口渲染事件
	class HAZEL_API AppRenderEvent : public Event
	{
	public:
		AppRenderEvent() {}

		EVENT_CLASS_TYPE(AppRender)
		EVENT_CLASS_CATEGORY(EventCategoryApplication)
	};
```

# 三. 事件分发器

```cpp
$Event.h

// 事件分发器，根据类型将事件分发给相应的处理函数
class HAZEL_API EventDispatcher
{
	template<typename T>
	using EventFn = std::function<bool(T&)>;
public:
	EventDispatcher(Event& event)
		: m_Event(event)
	{
	}

	template<typename T>
	bool Dispatch(EventFn<T> func)
	{
		// 定义一个函数类型别名，表示接受事件引用并返回布尔值的函数对象
		if (m_Event.GetEventType() == T::GetStaticType())
		{
			m_Event.Handled = func(*(T*)m_Event);
			return true;
		}
		return false;
	}
private:
	Event& m_Event;
};
```

## 效果

我们模拟调整窗口大小事件

```cpp
$Application.cpp

void Application::Run()
{
	WindowResizeEvent e(1280, 720);
	HZ_TRACE(e.ToString());

	while (true);
}
```

运行结果

![024](../assets/024.webp)

