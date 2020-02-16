---
title: WA2的逆向与外挂（二） 实现鼠标自动化操作
author: ayanamists
date: 2020-02-16
---

## 实现自动操作的几个想法

实现自动操作，大概有几个方法：

+ 直接调用某些WA2的函数，强行推进
+ 使用鼠标自动化方法，模拟人的行为

这里，我们选择了第二个方案。因为在一番逆向过后，我并没有完全理解WA2剧情推进的机制，只能等日后逆向能力更强后再实践。

## Windows平台下鼠标自动化的方法

据我所知，要实现UI的自动化，大概有几个方法：

+ 使用win32 api，直接根据windows的消息机制进行注入。
+ 使用vbs脚本
+ 使用微软官方的[UI Automation](https://docs.microsoft.com/zh-cn/windows/win32/winauto/uiauto-uiautomationoverview)库
+ 使用python的某些库
+ 使用自己写的驱动，实现硬件级别模拟
+ ...

这里，我们选择了第一种使用WIN API的方法，原因有两个：

+ 相对比较简单直接，不需要进行内核编程，开发效率高
+ 能力相对较强，能够处理绝大多数场景

## 初次尝试的折戟沉沙

我们知道，Windows的应用程序基于的是消息机制。什么是消息机制呢？MSDN上有一段令我印象深刻的解说：

> Unlike MS-DOS-based applications, Windows-based applications are event-driven. They do not make explicit function calls (such as C run-time library calls) to obtain input. Instead, they wait for the system to pass input to them.

每个Windows App的**GUI线程**都有一个**消息队列**，至于**GUI线程**是什么，我没有再MSDN上查到明确说法，个人认为是调用了**CreateWindow**的线程。

应用程序使用**GetMessage**函数从自己的消息队列中取出消息，使用**DispatchMessage**将消息传给对应的WindowProc，使用**WaitMessage**函数等待外部传入消息。

外部向应用程序发送消息，有两个办法：

1. 使用PostMessage()函数，将一个消息压入某个窗口对应线程的消息队列中。

   ```c++
    BOOL PostMessageA(
    HWND   hWnd,
    UINT   Msg,
    WPARAM wParam,
    LPARAM lParam
    );
   ```

2. 使用SendMessage()函数，直接调用某个窗口的WindowProc。

    ```c++
    LRESULT SendMessage(
    HWND   hWnd,
    UINT   Msg,
    WPARAM wParam,
    LPARAM lParam
    );
    ```

我们一向是实践第一，马上在写代码的时候发现了盲点：如何得到那个**HWND**类型的窗口句柄呢？这就要救助于FindWindow()函数了：

```c++
HWND FindWindowA(
  LPCSTR lpClassName,
  LPCSTR lpWindowName
);
```

就差一点点了！可是又如何得到主窗口的ClassName和WindowName呢？

这里介绍一个工具，就是大名鼎鼎的spy++.

我们可以在vs2019的工具里直接使用它：

![1](https://pic.downk.cc/item/5e49095348b86553eed809e1.png)

通过这个工具，我们就可以得到WA2主窗口的ClassName和WindowName了：

![2](https://pic.downk.cc/item/5e490a0748b86553eed82f5f.png)

![3](https://pic.downk.cc/item/5e490a0748b86553eed82f61.png)

使用Spy++工具，我们还可以得知这个窗口所有的消息，进而得知鼠标点击的坐标：

![4](https://pic.downk.cc/item/5e490b2348b86553eed86a17.png)

注：这里的坐标是窗口坐标，也就是相对于这个窗口左上角的坐标。

那么，我们只要向这个窗口发送LBUTTONDOWN和LBUTTONUP消息，不就可以模拟鼠标的行为了吗？

写出以下代码：

```c++
	if ( !PostMessage(main_window, WM_LBUTTONDOWN, wparam, lparam) ) {
    OnCriticalError("Post Down fail");
	};
	Sleep(20);
	
	wparam = 0;
	lparam = MAKELPARAM(600, 376);
	if ( !PostMessage(main_window, WM_LBUTTONUP, wparam, lparam) ) {
		OnCriticalError("Post Up fail");
	}
```

实验发现，完全没啥用。第一回合，我败下阵来。

## 分析原因，重整旗鼓

为什么刚才的尝试失败了呢？我想，这个问题只能向逆向工程里要答案了。

我们研究一下White AlbumCN这个窗口类的WindowProcess。它处理WM_LBUTTONDOWN或者WM_LBUTTONUP消息的代码在哪里呢？

![6](https://pic.downk.cc/item/5e4925c948b86553eede746f.png)

呵呵！原来如此！这个函数直接Return 0，根本不处理这个消息！

我感到很迷惑，既然如此，它是如何接收输入的呢？

我们先来看看主消息循环：

![7](https://pic.downk.cc/item/5e4926ba48b86553eedeae70.png)

注意被我标红的一条执行路径。被我命名为maybe_handle_click的函数，是这样的：

```c++

// line 37
  GetCursorPos(&Point);
  ScreenToClient(hWnd, &Point);

//line 105 - 129
    if ( !byte_B601DF )
    {
      v21 = GetAsyncKeyState;
      if_leftButtonPress = (GetAsyncKeyState(1) & 0x8001) != 0;
      if_rightButtonPress = (GetAsyncKeyState(2) & 0x8001) != 0;
LABEL_47:
      byte_B601AB = (v21(4) & 0x8001) != 0;
      goto LABEL_48;
    }
    if ( if_leftButtonPress )
    {
      v22 = GetAsyncKeyState;
      if_leftButtonPress = (GetAsyncKeyState(1) & 0x8001) != 0;
    }
    else
    {
      if ( !if_rightButtonPress && !byte_B601AB )
      {
        v21 = GetAsyncKeyState;
        if_leftButtonPress = (GetAsyncKeyState(1) & 0x8001) != 0;
        if ( !if_leftButtonPress )
        {
          if_rightButtonPress = (GetAsyncKeyState(2) & 0x8001) != 0;
          if ( !if_rightButtonPress )
            goto LABEL_47;
```

这个函数里反复调用了一个叫做GetAsyncKeyState的函数，查MSDN，描述如下：

> Determines whether a key is up or down at the time the function is called, and whether the key was pressed after a previous call to GetAsyncKeyState.

当这个函数的参数为1时，检测的就是鼠标左键。网上的资料还说，这个函数相当底层，基本是检测硬件的。

几乎可以确定，这个函数就是检测输入的。

那么，我们手里还有什么函数，可以在底层注入鼠标消息呢？

有，就是SendInput()，函数原型如下：

```c++
UINT SendInput(
  UINT    cInputs,
  LPINPUT pInputs,
  int     cbSize
);
```

这个函数也工作在底层，相当于直接用键盘/鼠标进行操作。

## 再次尝试，再次失败

我们写出以下代码：

```c++
	if (IsIconic(main_window))
	{
		ShowWindow(main_window, SW_RESTORE);
	}

	if (!SetForegroundWindow(main_window)) {
		OnCriticalError("Set window fail");
	}
	
	if (!Inject::EnablePrivilege()) {
		OnCriticalError("Privilege promt fail");
	}

	BeginEvent::SendBeginEvent(main_window);
```

其中SendBeginEvent()函数定义如下：
```c++
namespace BeginEvent {
	const DWORD beginX = 600;
	const DWORD beginY = 376;
	void SendBeginEvent(HWND main_window) {
		InputInjecter::SendMouseEvent(main_window, beginX, beginY);
	}
};

namespace InputInjecter {
	void SendMouseEvent(HWND main_window, int x, int y) {
		POINT p;
		p.x = x;
		p.y = y;

		ClientToScreen(main_window, &p);
		SetCursorPos(p.x, p.y);

		LPMOUSEINPUT input = new MOUSEINPUT;
		input->dwFlags = MOUSEEVENTF_ABSOLUTE | MOUSEEVENTF_MOVE | MOUSEEVENTF_LEFTDOWN | MOUSEEVENT_LEFTUP;
		input->dx = Helper::CalculateAbsoluteCoordinateX(p.x);
		input->dy = Helper::CalculateAbsoluteCoordinateY(p.y);
		input->mouseData = 0;
		input->dwExtraInfo = 0;
		input->time = 0;

		INPUT input_seal[1];
		input_seal[0].type = INPUT_MOUSE;
		input_seal[0].mi = *input;

		if (!SendInput(1, input_seal, sizeof(INPUT))) {
			OnCriticalError("Send Down fail");
		}

		Sleep(10);
	}
};
```

结果很诡异，用spy++可以看到消息是正确的，但桌面上的WA2就是纹丝不动。

为啥又失败了？我百思不得其解。

## 猜测和成功

我们观察到了两个现象：

+ 那个may_be_handle_click函数有很多反复横跳的地方，检测变量更是变来变去，强行分析似乎很麻烦。
+ 玩游戏的实际体验是，按下左键就会触发，而不是抬起时才触发。

我们猜想，是不是只注入DOWN的消息，不注入UP的消息，我们就会成功呢？

修改代码如下：

```c++
input->dwFlags = MOUSEEVENTF_ABSOLUTE | MOUSEEVENTF_MOVE | MOUSEEVENTF_LEFTDOWN | MOUSEEVENT_LEFTUP;
```

修改为：

```c++
input->dwFlags = MOUSEEVENTF_ABSOLUTE | MOUSEEVENTF_MOVE | MOUSEEVENTF_LEFTDOWN;
```

结果是成功的！效果如下：

![gif](../assets/what1.gif)

现在想到这件事，我的脸上仍然会浮现开心的笑容。这就是逆向的魅力所在吧！

现在，我们获得了自动化的“手”，下面，我们要获得自动化的“眼睛”。
