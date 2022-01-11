# Testbed 简介

Box2D 在 testbed 中提供了大量的应用实例，覆盖了 Box2D 的方方面面，阅读这部分代码上手 Box2D 非常合适。

Testbed 由 `OpenGL` 渲染，`ImGUI` 提供交互菜单，`sajson` 保存配置文件。这里我们先介绍 Testbed 除实例之外的代码，也就是 Testbed 的框架部分。

# 初始化部分

不管是什么代码我都先从 main 函数来分析（现在我们只关心在 windows 下会发生什么）：

```C++
int main(int, char**)
{
#if defined(_WIN32)
	// Enable memory-leak reports
	_CrtSetDbgFlag(_CRTDBG_LEAK_CHECK_DF | _CrtSetDbgFlag(_CRTDBG_REPORT_FLAG));
#endif

	char buffer[128];

	s_settings.Load();
	SortTests();

	glfwSetErrorCallback(glfwErrorCallback);

	g_camera.m_width = s_settings.m_windowWidth;
	g_camera.m_height = s_settings.m_windowHeight;

	if (glfwInit() == 0)
	{
		fprintf(stderr, "Failed to initialize GLFW\n");
		return -1;
	}

    glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
	glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3);
	glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE);
	glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);

	sprintf(buffer, "Box2D Testbed Version %d.%d.%d", b2_version.major, b2_version.minor, b2_version.revision);

	bool fullscreen = false;
	if (fullscreen)
	{
		g_mainWindow = glfwCreateWindow(1920, 1080, buffer, glfwGetPrimaryMonitor(), NULL);
	}
	else
	{
		g_mainWindow = glfwCreateWindow(g_camera.m_width, g_camera.m_height, buffer, NULL, NULL);
	}

	if (g_mainWindow == NULL)
	{
		fprintf(stderr, "Failed to open GLFW g_mainWindow.\n");
		glfwTerminate();
		return -1;
	}

	glfwMakeContextCurrent(g_mainWindow);
```

有关 GLFW 的 API 我们都不做阐述，因为我们可以在 [LearnOpenGL](https://learnopengl.com/) 上学习到这些东西。

testbed 首先用 `_CrtSetDbgFlag` 函数打开了程序在退出时自动检测内存泄露并生成错误报告的位开关，然后调用了 s_setting.Load 和 SortTest 两个函数。

在我们点进去查看这两个函数的实现时，不得不先提一提 Box2D 的命名规范。Box2D 对于所有静态变量都会加上 s_ 的前缀，全局变量则是 g_，成员变量则是 m_。

静态变量 s_setting 是一个 Settings 类，里面的成员变量则是 testbed 每次启动和退出时所需要加载和保存的配置，例如 `当前的测试实例索引(testIndex)`、`窗口长宽(windowWidth, windowHeight)` 等，方法实现则是调用了 `sajson` 中的 API 完成的。

再是 SortTests 这个函数，它负责整理储存 TestEntry 结构的数组 g_testEntries，所有测试实例在注册成功后都会保存在这里面。

TestEntry 结构有三个变量：

+ category (const char *)：测试实例的类别，譬如 `Benchmark`、`Bugs` 等
+ name (const char *)：测试实例的名称，譬如 `add_pair`、`tiles` 等
+ createFcn ( Test* (*p)() )：测试实例的实例化函数

由于所有注册过程都是静态完成的，所以一开始所有的实例都会注册完毕，CompareTests 再通过 category 和 name 来对他们进行排序。

再接下来，用 GLAD 加载完 OpenGL 的所有函数后，testbed 开始用 GLFW 注册回调函数：

```C++
	// Load OpenGL functions using glad
	int version = gladLoadGL(glfwGetProcAddress);
	printf("GL %d.%d\n", GLAD_VERSION_MAJOR(version), GLAD_VERSION_MINOR(version));
	printf("OpenGL %s, GLSL %s\n", glGetString(GL_VERSION), glGetString(GL_SHADING_LANGUAGE_VERSION));

	glfwSetScrollCallback(g_mainWindow, ScrollCallback);
	glfwSetWindowSizeCallback(g_mainWindow, ResizeWindowCallback);
	glfwSetKeyCallback(g_mainWindow, KeyCallback);
	glfwSetCharCallback(g_mainWindow, CharCallback);
	glfwSetMouseButtonCallback(g_mainWindow, MouseButtonCallback);
	glfwSetCursorPosCallback(g_mainWindow, MouseMotionCallback);
```

`ScrollCallback` 和 `KeyCallback` 这两个回调函数有一个很有意思的地方，它们分别检测了 `ImGui::GetIO().WantCaptureMouse` 和 `ImGui::GetIO().WantCaptureKeyboard` 来判断 ImGui 是否想要获取到鼠标和键盘的事件，如果为 true 则这些回调函数提前 return，防止用户与菜单交互时和场景交互时发生冲突。

接下来 testbed 调用了 `g_debugDraw.Create`，这个全局变量的类型为 `DebugDraw`，继承于 `b2Draw`。这有一个问题，Box2D 不是不负责渲染功能吗，为什么会有这么一个类呢？

实际上当你查看这个类的时候，你会发现 b2Draw 里除了一些位指定是否绘制某一对象以外，其余的渲染方法全都是纯虚函数，而 testbed 中用 OpenGL 覆盖实现了这些方法。

DebugDraw 有四个成员变量：

+ m_showUI (bool)：是否渲染 UI 界面
+ m_points (GLRenderPoints*)：OpenGL 渲染点
+ m_lines (GLRenderLines*)：OpenGL 渲染线
+ m_triangles (GLRenderTriangles*)：OpenGL 渲染三角形

`g_debugDraw.Create` 负责实例化后面三个变量并调用他们的 Create 方法，这三者的 Create 方法负责设置好 OpenGL 状态机的状态让其绘制出指定的图像，这其中包括硬编码的 GLSL 程序。

点、线、三角形作为基础图形可以绘制出 testbed 中各种图形，包括圆，如果你放大点看会发现其实绘制出来的圆是个正多边形。

再接下来，testbed 调用了 CreateUI 函数初始化 ImGui，并加载了所需的字体，然后就准备进入 testbed 的循环了。

```C++
	s_settings.m_testIndex = b2Clamp(s_settings.m_testIndex, 0, g_testCount - 1);
	s_testSelection = s_settings.m_testIndex;
	s_test = g_testEntries[s_settings.m_testIndex].createFcn();

	// Control the frame rate. One draw per monitor refresh.
	//glfwSwapInterval(1);

	glClearColor(0.2f, 0.2f, 0.2f, 1.0f);

	std::chrono::duration<double> frameTime(0.0);
	std::chrono::duration<double> sleepAdjust(0.0);
```

这段首先调用 b2Clamp 将场景索引限制在正确的范围内，这里体现了 Box2D 的神奇的一个地方，就是它造了非常多自己的轮子。

接下来选择好了配置文件中指定的场景实例并实例化，设置好 OpenGL 清除屏幕所用的颜色，并初始化 `frameTime` 和 `sleepAdjust` 这两个用于控制帧率的变量后，testbed 就进入了循环部分。

# 循环部分

这里先来看看 testbed 控制帧率的部分，也是非常有意思的：

```C++
	while (!glfwWindowShouldClose(g_mainWindow))
	{
		std::chrono::steady_clock::time_point t1 = std::chrono::steady_clock::now();

        // 该帧的渲染以及事件更新
        // ...

        std::chrono::steady_clock::time_point t2 = std::chrono::steady_clock::now();
		std::chrono::duration<double> target(1.0 / 60.0);
		std::chrono::duration<double> timeUsed = t2 - t1;
		std::chrono::duration<double> sleepTime = target - timeUsed + sleepAdjust;
		if (sleepTime > std::chrono::duration<double>(0))
		{
			std::this_thread::sleep_for(sleepTime);
		}

		std::chrono::steady_clock::time_point t3 = std::chrono::steady_clock::now();
		frameTime = t3 - t1;

		// Compute the sleep adjustment using a low pass filter
		sleepAdjust = 0.9 * sleepAdjust + 0.1 * (target - frameTime);
    }
```

每帧开头结尾获取时间 t1 和 t2 来决定延迟的时间是一般的写法，然而这里还有一个 t3，和 sleepAdjust 一起负责调整每帧应当休眠的时间。

由于 sleep_for 函数并不一定精确，而且有可能因为某一帧工作量太大，消耗的时间会远远超出我们期望的时间（target，1.0f / 60.0f），这时候 sleepAdjust 便派上了用场。

在每一帧所有工作包括延时完成后，testbed 再次获取当前时间并赋值给 t3，并计算出这一帧的实际用时（frameTime），最后 `sleepAdjust` 等于 0.9 * `原先的 sleepAdjust` 加上 0.1 * `当帧期望用时与当帧实际用时的差`。

其中用 0.1 乘上这个差可以防止有几帧出现巨大波动时会大大影响到后面几帧，0.9 乘上原来的 sleepAdjust 又可以让帧率波动在接下来的几帧内慢慢调整过来。于是乎，testbed 的帧率在调整中趋于稳定。

接下来让我们再看看每一帧工作的内容：

```C++
	glfwGetWindowSize(g_mainWindow, &g_camera.m_width, &g_camera.m_height);
        
    int bufferWidth, bufferHeight;
    glfwGetFramebufferSize(g_mainWindow, &bufferWidth, &bufferHeight);
    glViewport(0, 0, bufferWidth, bufferHeight);

	glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
```

这一部分负责的是当窗口大小改变时 testbed 需要做的工作。其中，摄像机的拍摄场景大小将被重新设置，帧缓冲区的像素长宽被用来重新设置视口，这是因为 OpenGL 使用的是双缓冲技术，所以你要给屏幕上的那一帧和缓冲区的那一帧都重新设置好大小。

重要的是这个摄像机，让我们查看一下这个摄像机结构的实现细节：

```C++
struct Camera
{
	Camera();

	void ResetView();
	b2Vec2 ConvertScreenToWorld(const b2Vec2& screenPoint);
	b2Vec2 ConvertWorldToScreen(const b2Vec2& worldPoint);
	void BuildProjectionMatrix(float* m, float zBias);

	b2Vec2 m_center;
	float m_zoom;
	int32 m_width;
	int32 m_height;
};
```

这个摄像机的四个成员不用多说，最重要的是 `BuildProjectionMatrix` 这个方法，他构造了一个投影矩阵，可以让 OpenGL 按照正确的缩放比例渲染出整个画面。

`glClear` 清空了所有的颜色缓冲和深度缓冲，接下来：

```C++
	ImGui_ImplOpenGL3_NewFrame();
	ImGui_ImplGlfw_NewFrame();

	ImGui::NewFrame();

	if (g_debugDraw.m_showUI)
	{
		ImGui::SetNextWindowPos(ImVec2(0.0f, 0.0f));
		ImGui::SetNextWindowSize(ImVec2(float(g_camera.m_width), float(g_camera.m_height)));
		ImGui::SetNextWindowBgAlpha(0.0f);
		ImGui::Begin("Overlay", nullptr, ImGuiWindowFlags_NoTitleBar | ImGuiWindowFlags_NoInputs | ImGuiWindowFlags_AlwaysAutoResize | ImGuiWindowFlags_NoScrollbar);
		ImGui::End();

		const TestEntry& entry = g_testEntries[s_settings.m_testIndex];
		sprintf(buffer, "%s : %s", entry.category, entry.name);
		s_test->DrawTitle(buffer);
	}

	s_test->Step(s_settings);

	UpdateUI();
	
    if (g_debugDraw.m_showUI)
	{
		sprintf(buffer, "%.1f ms", 1000.0 * frameTime.count());
		g_debugDraw.DrawString(5, g_camera.m_height - 20, buffer);
	}

	ImGui::Render();
	ImGui_ImplOpenGL3_RenderDrawData(ImGui::GetDrawData());
```

这一段首先初始化了 ImGui 的新一帧，在判断 m_showUI 后，先创建了一个覆盖了整个屏幕的图层 `"Overlay"`，这个图层用于显示文字，在 testbed 中绘制了左上角的 `测试实例标题` 与左下角的 `当前帧率`。

然后是 `UpdateUI` 函数，它负责绘制了 `Tools` 交互菜单，关于这个交互菜单的实现细节我们不做阐述。有趣的是里面还调用了 `s_test->UpdateUI`，某些特定的测试实例譬如 `sensors` 有自己的数据需要调整，只需要覆盖重写这个函数即可。

最后，调用测试实例的 Step 方法来更新当前实例，所有测试实例类均继承于 Test 类，在这里调用的这个 Step 方法便是整个实例的精髓，里面除了对 Box2D 对象的操作，另外的就是渲染整个场景，并更新统计表（统计与 Box2D 原理相关的变量）。这些我们可以在后面实例分析中慢慢看。

```C++
	glfwSwapBuffers(g_mainWindow);

	if (s_testSelection != s_settings.m_testIndex)
	{
		s_settings.m_testIndex = s_testSelection;
		delete s_test;
		s_test = g_testEntries[s_settings.m_testIndex].createFcn();
		g_camera.ResetView();
	}

	glfwPollEvents();
```

`glfwSwapBuffers` 交换帧缓冲画面将其渲染出来，

然后检测当前是否更换实例，如果更换了测试实例，便删除当前实例并构造新的那个，

最后调用 `glfwPollEvents` 更新事件，至此，单帧的任务就完成了。

# 退出部分

```C++
	delete s_test;
	s_test = nullptr;

	g_debugDraw.Destroy();
	ImGui_ImplOpenGL3_Shutdown();
	ImGui_ImplGlfw_Shutdown();
	glfwTerminate();

	s_settings.Save();

	return 0;
```

没有什么需要多说的，析构所有的对象，并调用 glfw 与 ImGui 的退出函数，最后保存当前所有的配置，整个框架便以 `return 0` 画上了句号。

这就是 testbed 整个框架的介绍，接下来我会慢慢阅读具体的实例来学习 Box2D。