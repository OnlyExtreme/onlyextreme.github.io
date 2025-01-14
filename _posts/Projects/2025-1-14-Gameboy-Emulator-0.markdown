# Introduction 简介

在长期苦苦寻找编程实战项目后，昨天晚上终于找到了一个详细的且对我胃口的好项目：[从零开始实现GameBoy模拟器](https://zhuanlan.zhihu.com/p/676908347)。这个作者更是个神仙，整个教程极其详细，正好我也想稍微深入了解一下计算机体系并且稍微提升提升自己的编程能力，尤其是 OOP 能力，毕竟作为信息竞赛生 OOP 没接触一点。而且其中所使用的图形 SDK 完全是自己写的，支持跨平台和多个图形库，设计理念也和我很吻合，遂决定重拾这个博客，督促自己记录学习过程。

# Environment 环境配置

原作者使用了 xmake 作为项目构建工具，貌似这玩意有包管理器，感觉大概率会比 cmake 和 make 好用一些？

不得不说确实方便多了，所有包都自动下载好了，真是神仙工具，我得学一学

# Structure 项目结构

整个项目基于原作者自己写的 [LunaSDK](https://github.com/JX-Master/LunaSDK)，主要包含三个文件夹：

1. Modules: 存放 LunaSDK 模块代码；
2. Programs: 存放基于 LunaSDK 的程序代码；
3. build: 用 xmake 构建程序时，用于储存可执行文件和二进制文件。

其中我们写的代码都在 Programs 文件夹下，初始情况下里面有三个文件：

1. main.cpp: 应用程序入口；
2. App.hpp / App.cpp: 定义了一个 App 类 (OOP 来咯)，用于存放应用程序全局变量和主循环。

从 main.cpp 开始看（注释是我自己写的）:

```cpp
App* g_app;

RV run_app()    // RV = R<void> = Void Result, no values but can return errors. Use R<T> to return potential errors in LunaSDK.
{
    // Error handling methods for LunaSDK
    lutry
    {
        // Add modules.
        luexp(add_modules({module_window(), module_rhi(), module_ahi(), module_imgui()}));
        // Initialize modules.
        luexp(init_modules());
        // Run the application.
        luexp(g_app->init());
        while(!g_app->is_exiting)
        {
            luexp(g_app->update());
        }
    }
    lucatchret;
    return ok;
}

App* g_app;
int main()
{
    bool inited = Luna::init(); // Initialize core functions of LunaSDK.
    if(!inited) return -1;  // Initialization failed
    g_app = memnew<App>();
    RV r = run_app();
    if(failed(r))
    {
        log_error("LunaGB", explain(r.errcode()));
    }
    memdelete(g_app);
    Luna::close();  // Pairs with Luna::init(), to terminate the program and clear up the resources.
    return 0;
}
```

其中 `lutry` 这种错误处理机制也是我未曾接触过的，LunaSDK 模仿 C++ 做了一组宏包来覆盖可能出错的函数调用：

- `lutry { ... }` 尝试语句块，其中代码可能产生错误；
- `lucatch { ... }` 捕获语句块，`lutch` 中语句出错时跳转到这里，并且错误码可以被 `luerr` 捕获；
- `lucatchret` 相当于 `lucatch{ return luerr; }`，把错误交给函数调用方处理。如果要返回错误代码，则本函数返回值需要用 `R<T>` 模板包覆；
- `luexp(...)` 包覆一个返回值为 `RV` 的函数调用，发生错误时跳转到 `lucatch`
- `lulet(a, ...)` 包覆一个返回值为 `R<T>` 的函数调用，调用成功时定义一个类型为 `T` 的局部变量 `a` 接收返回值；错误时跳转到 `lucatch`
- `luset(a, ...)` 和 `lelet(a, ...)` 功能相似，但是这里是将返回值赋予到一个现有变量 `a` 上。

下面看 `App.hpp`，其中定义了一个 App 类和各种必须变量:

```cpp
struct App
{
    //! `true` if the application is exiting. For example, if the user presses the close
    //! button of the window.
    bool is_exiting;

    //! The RHI device used to render interfaces.
    Ref<RHI::IDevice> rhi_device;
    //! The graphics queue index.
    u32 rhi_queue_index;

    //! The application main window.
    Ref<Window::IWindow> window;
    //! The application main window swap chain.
    Ref<RHI::ISwapChain> swap_chain;
    //! The command buffer used to submit draw calls.
    Ref<RHI::ICommandBuffer> cmdbuf;

    RV init();
    RV update();
    ~App();

    void draw_gui();
    void draw_main_menu_bar();
};
```

接着是 `App.cpp`，定义了代码具体逻辑，有一个 `App::init()` 用于初始化，一个 `App::update()` 作为主循环:

```cpp
RV App::update()
{
    lutry
    {
        // Update window events.
        Window::poll_events();
        // Exit the program if the window is closed.
        if (window->is_closed())
	{
	    is_exiting = true;
            return ok;
	}

        // Draw GUI.
        draw_gui();

        // Clear back buffer.
        /*...*/
        // Render GUI.
        luexp(ImGuiUtils::render_draw_data(ImGui::GetDrawData(), cmdbuf, back_buffer));
	
        // Submit render commands and present the back buffer.
	/*...*/
    }
    lucatchret;
    return ok;
}
```

逻辑为：先查询窗口事件（比如关闭窗口之类的），然后调用 Dear ImGui 进行窗口绘图，最后等待这一帧内容绘制完成，将全部内容输出到屏幕上，并等待垂直同步。