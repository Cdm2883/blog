---
date: 2024-10-06
categories:
  - 技术
tags:
  - Kotlin
  - Compose
---

# 将 Compose Multiplatform 移植到 OpenHarmony 的可行性研究

随着鸿蒙系统的宣发，国内有越来越多的行业选择拥抱鸿蒙应用生态。但全新的生态，若从原生迁移构建新的应用，则需要大量的时间、学习、人力等成本。
但好在为了解决这种问题，业内涌现了许多优秀的多平台开发框架，其中不乏有 [Qt](https://qt.io)、[Flutter](https://flutter.dev)、[React Native](https://reactnative.dev) 等等。
鸿蒙的 Flutter 支持似乎是目前最受欢迎的 *~~（以至于官方 ArkUI 代码中都带有 Flutter 字样）~~*，多平台框架的支持既可以减少开发的压力，又可以的在短时间内快速补充鸿蒙的生态。
但我们今天探讨的主角是由 [Jetpack Compose](https://developer.android.com/develop/ui/compose) 演化而来新兴多平台 UI 框架 —— [Compose Multiplatform](https://jetbrains.com/lp/compose-multiplatform)。

<!-- more -->

## Compose 实现多平台的奥秘

> 本文不会深谈 iOS 相关的内容，若有误请不要喷我 qwq *~~（因为我没干过 Apple 的开发）~~*

Compose 的概念最初出现在 Android 上，作为 Jetpack 的一部分发布，叫做 Jetpack Compose。
作为一个现代的 UI 构建框架，得益于 Kotlin 优秀的语言特性，它能够使用更少、更直观的代码构建原生 Android 页面。

那么我们在使用 Jetpack Compose 构建和运行应用的时候到底发生了什么？

```mermaid
graph LR
    subgraph 构建 
        ComposeCompiler["Compose Compiler"]
    end
    subgraph 运行
        ComposeRuntime["Compose Runtime"]
        ComposeUI["Compose UI"]
        Other["&lt;OTHER COMPONENT LIBRARIES&gt;"]
        UI["User Interface"]
    end
    ComposeCompiler --> ComposeRuntime --> ComposeUI --> Other --> UI
```

=== "Compose Compiler"

    Kotlin 编译器插件。用于静态检查优化和将 `@Composable` 函数转换为 Composition 的生成。

=== "Compose Runtime"

    对 Composition 进行管理（状态表的管理和渲染树的生成）。

=== "Compose UI"

    渲染树管理的实现，再将生成后的渲染树进行布局和绘制。

=== "<OTHER COMPONENT LIBRARIES\>"

    基于 Compose UI 提供库代码和组件等等，例如：

    - Compose Fundation
    - Compose Material
    - [OreCompose](https://github/Cdm2883/OreCompose) <small>~~*夹带私货*~~</small>

从整个过程来看 Compose UI 之前的部分都是平台无关的，驱动着一棵节点树的更新，支持着整个 Compose 的运转。
而 Compose UI 则是与当前平台所关联的，包装不同平台的差异，处理输入，管理真正的渲染树和将生成的渲染树给画出来。
<small>*（是不是很像虚拟 Dom 和真实 Dom 的关系 XD。其实从这里也可以看出 Compose 不止可以用来构建 UI。
还记得在 Compose 诞生之初有人拿 Compose 做了一个测试框架，但具体的仓库好像忘记了 :P）*</small>

JetBrains 团队创建的 Compose Multiplatform 项目则利用这一点，借助 Kotlin 的多平台能力，厚积薄发，为基于 Skia 的 Compose UI 提供各个平台相应的绑定。
由于都用的同一个渲染引擎（Skia）、同一套节点树和流程，这样就实现了多平台统一风格样式组件的 UI，甚至可以借用 Android *(Jetpack Compose)* 那边[已有的](https://github.com/JetBrains/compose-multiplatform-core)通用组件库。

```mermaid
graph LR
    ComposeCompiler["Compose Compiler"]
    ComposeRuntime["Compose Runtime"]
    ComposeUI["Compose UI"]
    ComposeCompiler --> ComposeRuntime -- Layout Node --> ComposeUI
    ComposeUI --> Android["Android (Jetpack Compose)"]
    ComposeUI --> Desktop["Desktop (Skiko)"]
    ComposeUI --> iOS["Desktop (Skiko)"]
    ComposeUI --> Web["Web (Skia via Kotlin/Wasm)"]
```

---

Compose 实现多平台还有另一种方式 —— **使用平台原生的页面元素**。  
事实上曾经的 Compose Web *（[现 Compose HTML](https://github.com/JetBrains/compose-multiplatform/commit/59eda00380981b2555cd62d26e8d6f4122a13c40)）*就是这样做的。
<small>*（通过改名也能看出，JetBrains 团队不希望在 Web 上使用 Compose 会与其他平台过于割裂）*</small>
```kotlin title="Written in Compose HTML"
fun main() = renderComposable(rootElementId = "root") { Body() }

@Composable
fun Body() {
    var counter by remember { mutableStateOf(0) }
    Div(attrs = {
        style {  // css style, not modifier
            width(20.percent)
            height(10.percent)
        }
    }) {
        Text("Clicked: ${counter}")
    }
    Button(attrs = {
        style { property("padding", "0px 0px 0px 16px") }
        onClick { _ -> counter++ }
    }) {
        Text("Click")
    }
}
```
从这一段代码很容易看出，这里虽然沿用了 Compose 的状态管理，但是还是采用了浏览器原生 Dom 来构建的 UI。
这样做，由于每个平台的差异性，又无法做到 UI 共用一个代码了。

有什么办法消除这种差异？聪明的你很容易就能想到，可以用抽象的思想！把每个组件抽象化，提取通用部分，再具体在每个平台进行实现。
而 [Redwood](https://github.com/cashapp/redwood) 就是这么做的：

??? example annotate "示例"

    ```kotlin title="Schema (1)" linenums="1"
    @Widget(1)
    data class Button(
        @Property(1)
        val text: String?,
    
        @Property(2)
        @Default("true")
        val enabled: Boolean,
    
        @Property(3)
        val onClick: (() -> Unit)? = null,
    )
    ```

    === "Android"
    
        ```kotlin linenums="1" hl_lines="2"
        internal class AndroidButton(
            override val value: android.widget.Button,
        ) : Button<View> {
            override var modifier: /*app.cash.redwood.*/Modifier = Modifier
            override fun text(text: String?) {
                value.text = text
            }
            override fun enabled(enabled: Boolean) {
                value.isEnabled = enabled
            }
            override fun onClick(onClick: (() -> Unit)?) {
                value.setOnClickListener(onClick?.let { { onClick() } })
            }
        }
        ```
    
    === "Web"
    
        ```kotlin linenums="1" hl_lines="2"
        internal class HtmlButton(
            override val value: HTMLButtonElement,
        ) : Button<HTMLElement> {
            override var modifier: /*app.cash.redwood.*/Modifier = Modifier
            override fun text(text: String?) {
                value.textContent = text
            }
            override fun enabled(enabled: Boolean) {
                value.disabled = !enabled
            }
            override fun onClick(onClick: (() -> Unit)?) {
                value.onclick = onClick?.let { { onClick() } }
            }
        }
        ```
    
    === "Desktop"
    
        ```kotlin linenums="1" hl_lines="8-16"
        internal class ComposeUiButton : Button<@Composable () -> Unit> {
            private var text by mutableStateOf("")
            private var isEnabled by mutableStateOf(false)
            private var onClick by mutableStateOf({})
        
            override var modifier: /*app.cash.redwood.*/Modifier = Modifier
        
            override val value = @Composable {
                androidx.compose.material.Button(
                    onClick = onClick,
                    enabled = isEnabled,
                    modifier = androidx.compose.ui.Modifier.fillMaxWidth(),
                ) {
                    Text(text)
                }
            }
        
            override fun text(text: String?) {
                this.text = text ?: ""
            }
            override fun enabled(enabled: Boolean) {
                this.isEnabled = enabled
            }
            override fun onClick(onClick: (() -> Unit)?) {
                this.onClick = onClick ?: {}
            }
        }
        ```
    
    === "iOS"
    
        ```kotlin linenums="1" hl_lines="5-7"
        // NOTE: This class must be public for the click selector to work.
        class IosButton : Button<UIView> {
            override var modifier: /*app.cash.redwood.*/Modifier = Modifier
            
            override val value = UIButton().apply {
                backgroundColor = UIColor.grayColor
            }
    
            override fun text(text: String?) {
                value.setTitle(text, UIControlStateNormal)
            } 
            override fun enabled(enabled: Boolean) {
                value.enabled = enabled
            }
        
            private val clickedPointer = sel_registerName("clicked")
            @ObjCAction
            fun clicked() {
                onClick?.invoke()
            }
            private var onClick: (() -> Unit)? = null
            override fun onClick(onClick: (() -> Unit)?) {
                this.onClick = onClick
                if (onClick != null) {
                    value.addTarget(this, clickedPointer, UIControlEventTouchUpInside)
                } else {
                    value.removeTarget(this, clickedPointer, UIControlEventTouchUpInside)
                }
            }
        }
        ```

1. Redwood 会自动生成类型安全的 API 供包装。例如这个会生成的接口：
   ```kotlin
   // ...
   interface Button<W : Any> : Widget<W> {
       // ...
       fun text(text: String?)
       fun enabled(enabled: Boolean)
       fun onClick(onClick: (() -> Unit)?)
   }
   // ...
   ```
   <br/>
   完整示例代码详情请看：[samples/counter/schema/src/main/kotlin/com/example/redwood/counter/schema.kt](https://github.com/cashapp/redwood/blob/71fc67243dbc39fc3a6d2b579ef10a07e451e7b8/samples/counter/schema/src/main/kotlin/com/example/redwood/counter/schema.kt)

使用原生组件，理所应当会更贴近原生的体验。但很显然，这样做工作量可不小；由于每个人可能对底层抽象模式有不同的标准，组件库也很难做到通用。

## 将 Compose UI 移植到鸿蒙

截至到这篇博文发布，其实已经有个人，甚至许多大厂在探索自己的解决方案。

我找到了一个使用原生包装方案的项目 —— [compose-ez-ui](https://github.com/Compose-for-OpenHarmony/compose-ez-ui)，它也是通过 Redwood 实现的。
纵然这是一次有趣的尝试，但大家更想要的一定会是兼容现有 Compose Multiplatform 生态的实现。
也就是说，我们需要移植 Compose UI，用 skia canvas 在鸿蒙上进行自绘制。

据未经证实的消息，上上上上个月(1)腾讯在深圳的演讲，透露了他们的团队正在为 OpenHarmony 做 Skia 的 Binding，并计划在 2025 年开源。
此外，腾讯视频等应用的鸿蒙版本中早已运用了 Kotlin + Compose 的技术，据传美团也有在做相关的研究。
快手团队也在探索 KMP 在鸿蒙上的可能，现已在快影等应用应用了相关技术……
{ .annotate }

1. 啊啊啊这篇博文托更好几个月了<small>*（因为学业和我太懒）逃）*</small>，这个时间反复改了好几次

相对而言，Compose UI 的移植相对会简单不少<small>*（由于许多通用代码和包装）*</small>，
所以接下来我们就探讨一下对设备 Skia 进行绑定（Skiko）的几个容易想到的方法：

### 使用原生 Skia

但在一切开始之前，我想先提一个在 KotlinConf'24 中由 [Jake Wharton](https://github.com/JakeWharton)(1) 分享的一个有趣的故事……
{ .annotate }

1. Jake Wharton（Cash App Android 工程师）。同时，Redwood 也是 Cash App 的开源项目。

<!--suppress CssUnusedSymbol, SpellCheckingInspection -->
<style>
#composeui-lightswitch-figcaption .md-annotation__index:after {
    margin-left: -.94ch;
}
</style>
<figure style="width: 100%;margin-top: 2em;" class="annotate">
    <!--suppress HtmlUnknownAttribute, HtmlDeprecatedAttribute -->
    <iframe
        style="border-radius: .1rem;aspect-ratio: 16 / 9;"
        src="//player.bilibili.com/player.html?isOutside=true&aid=1956437488&bvid=BV1ky411e7ox&cid=1648024007&p=1"
        width="100%"
        scrolling="no"
        border="0"
        frameborder="no"
        framespacing="0"
        allowfullscreen="allowfullscreen">
    </iframe>
    <figcaption id="composeui-lightswitch-figcaption" style="max-width: none;">在智能电灯开关上运行 Compose UI：探索 Compose 的嵌入式应用(1)</figcaption>
</figure>

1. 一些相关的链接：[Github](https://github.com/JakeWharton/composeui-lightswitch)、
   [BiliBili](https://www.bilibili.com/video/BV1ky411e7ox)、
   [Youtube](https://youtu.be/D0P5Lb-2uCY)、
   [Home Assistant Community](https://community.home-assistant.io/t/500842)。

视频较长，在这里我就简要地描述一下：

讲师的朋友在亚马逊发现了一个存在未加密 ADB 接口的智能开关设备，并且可以轻松地获得 Root 权限。
这引发了讲师的兴趣，促使了他购买该设备并尝试探索在其上使用 Compose 构建出自己的用户界面[^1]。

探索的过程中他发现设备其实运行的是一个简化的 Linux 系统，而不是安卓。所以他首先尝试在设备上运行 JVM，
并测试了简单的 "Hello World" 程序，证明了设备可以支持 JVM，这让他信心倍增。

但在尝试直接在设备上运行 Compose Desktop (JVM) 时，他遇到了诸多挑战。
首先 [Skiko](https://github.com/JetBrains/skiko) 很快发出了不满的声音：
`libGL.so.1: connot open shared object file: No such file or directory`。
这说明设备上并没有常规或完整的 OpenGL。
并且 Compose Desktop 还使用了 Swing (AWT)，AWT Linux 默认情况下需要依赖于 X11 等桌面环境，显然这个小小的开关是没有这些东西的[^2]。

那该智能开关的原界面是怎样绘制的？
他之前就已发现该智能开关的原界面其实是通过 Flutter 构建的。Flutter 同样使用 Skia 作为图形引擎，而在当前设备上用的 OpenGL ES 加速绘制，并最终通过 DRM 直接输出渲染结果到显示设备。

几经转折，讲师找到了 [Linux_DRM_OpenGLES.c](https://gist.github.com/Miouyouyou/89e9fe56a2c59bce7d4a18a858f389ef)
并成功在设备上运行了这段绘制代码。但这些都是 C 代码，而这里是 **Kotlin**Conf，
所以讲师又尝试在设备上跑了 Kotlin/Native 的 Hello World，事实证明这可以编译运行，这使他大致知道了他应该怎么做。

他又花费了几周的时间用 Kotlin/Native 重写了全部逻辑，一切好似又回到了开头，但这次是使用 Kotlin 来构建所需的一切。
是时候让事情变得有趣了！为了在 Kotlin/Native 方便地使用 Skia 同时为后面对接 Compose UI 做准备，还是回到了 Skiko 项目。  
Skiko 是什么？
它自称是 Skia 的 Kotlin 多平台绑定，不仅支持常规的 Kotlin/JVM，
甚至支持用 WASM 在浏览器中运行和用 Kotlin/Native 在苹果设备中运行。
Compose UI 自身的多平台渲染同样也是归功于 Skiko 的强大赋能。所以在这台设备上成功部署 Skiko 至关重要。

但不幸的是，它不能就这样被立起来直接用，Skiko 目前不支持 Kotlin/Native 在 Linux 平台的绑定。
既然支持用 Kotlin/Native 在 iOS 和 macOS 中运行，Linux 应该不会太难吧？讲师这样想着。  
与 Skiko 关联的还有一个重要的仓库 [skia-pack](https://github.com/JetBrains/skia-pack)。
它使用 Gtihub Actions 来为 Skiko 构建所需的产物。但他们只构建 OpenGL，而不是 OpenGL ES。
所以讲师自己动手 Fork 了仓库，修改了构建脚本，一切顺利，所以我们可以回到 Skiko 并尝试集成它。

讲师又 Fork 了 Skiko 并“照猫画虎”地将 `GL` 字样用 `EGL` 补充了并增加了 Linux ARM 作为编译目标。
通常 Skiko 的构建脚本会从 Skia 包仓库（skia-pack）下载所需的依赖项，所以他又手动指定了让脚本从他 Fork 的分支中下载。
最后一件事，它是如何实际将 C++ 代码为 ARM Linux 编译的？它原本被设定为 clang++，但现在不得不将其更改为不同的脚本。
因为基本上 Skia 的编译设置只能让宿主机做为编译目标。所以在他的 Mac 上可以为 macOS 编译，在他的 Linux X64 服务器上可以为 Linux X64 编译。
那么好吧，他需要编译 Linux ARM，结果在他的树莓派上尝试后发现 Kotlin **本身**不能在 Linux ARM 上运行。
因此讲师创建了一个 docker 容器，用 QUME 模拟 Linux ARM，在这里进行编译。  
虽然几个月的时间都被花费在了这里，但庆幸的是最终它经过许多痛苦后成功了。

渲染固然重要，但缺少触摸交互就失去了不少趣味。讲师花了一点时间弄清这个设备的触摸事件，并用 Kotlin/Native 捕获。
现在，是时候把一切都组合在一起了！他 Fork 了 [compose-multiplatform-core](https://github.com/JetBrains/compose-multiplatform-core)，
并顺着依赖树逐步实现该平台的 `actuals`，终于到达了能够编译的时候。
他回到了他的应用，将低级的 Skia 调用替换了 Compose，并将触摸事件传递给 Compose。很快，智能开关上的第一个 Compose 页面诞生了！
他继续优化和美化了页面设计，但这还不是终点。

讲师表示这就像虽然这段旅程接近尾声，但还没有越过终点线，因为它最终还是一个开关。这是一些串口通信的事，不在本文的讨论范围，所以就不多加赘述了。

最后是关于 Skiko Linux 支持的事情。Skiko 对其他平台有绑定是因为其他平台有相对统一的桌面管理/显示系统，
例如 JVM 有 AWT，苹果有 SwiftUI 等等。
讲师也不希望能够直接支持，因为 Linux 的情况要复杂得多，可能存在多种显示系统，比如 X11、Wayland，甚至直接渲染（DRM）或其他更独特的方式。

---

至此，演讲结束。这段演讲不仅是探索 Compose 在嵌入式的应用，也很好地展示了为 Compose Multiplatform 兼容一个新平台的大概步骤。
我们是否能使用类似的方法，为鸿蒙系统提供 Compose Multiplatform 的支持？

很遗憾，我目前并未看到走此路线的已公开项目。笔者的电脑也无法构建像 Skiko 这样如此庞大的项目(1)，因此这里我无法拿出具体的代码进行演示。
{ .annotate }

1. 我也曾多次尝试为 Window X64 编写 Compose Multiplatform 的 Kotlin/Native 支持（就像 macOS 的预览功能一样）， 
   但是在拉取 Skiko 编译时，发现其中有使用 JetBrains 的内部插件处理代码发布的任务，我无法拉取相关插件。
   在去除相关代码后，还是因为电脑空间不足被迫停止了研究（笑）。

[^1]:

    朋友原话：

    btw. new home automation side project:
    I bought [one of these](javascript:footer_link_one_of_these()) and have been trying to get my own app installed on it.
    Super cheap hardware with exactly the design I want,
    but Chinese servers and no Home Assistant support (hence the custom app).

    Someone discovered it has an ADB server running with no password and root access,
    so getting into the device is simple. Turns out though, I don't think it's AOSP.
    It's some stripped down Linux install that happens to have adbd running.

    Been a fun project so far. I might have questions..

<script>
function footer_link_one_of_these() {
    alert("你其实可以从本文的其他地方拿到这个链接")
}
</script>

[^2]:
   **Eric Firestone:** I assume skiko is skia? Which Flutter also uses, right? So it's probably on here
   <br/>
   **Jake Wharton:** kotlin skia wrapper, yes
   <br/>
   **Eric Firestone:** I'll keep poking. IibGLES[12].so is on the system. Wonder if that's usable.
   <br/>
   **Jake Wharton:** unfortunately it looks like the JVM support for Compose UI relies on AWT which requires X11
   <br/>
   **Jake Wharton:** i think we need to set the jvm to headless mode and then somehow initialize a gl context for the whOIdisplay and bind to that  
   because the normal codepaths just aren't going to work
   <br/>
   **eric:** That was exactly my thought too. Didn't know about headless mode, but for the OpenGL context.

### 在 ArkUI 层实现 Skia

本文到现在已经至少出现了有 15 次 `Skia` 字样！那么你可能会想问：Skia 到底是什么？

> Skia 是一个开源 2D 图形库，它提供适用于各种硬件和软件平台的通用 API。
> 它被用作 Google Chrome 和 ChromeOS、Android、Flutter 和许多其他产品的图形引擎。
> <footer>—— 译自 [Skia](https://skia.org) 官网首页</footer>

由于具备跨平台运行能力、支持多种后端以及出色的兼容性，Skia 也被选为 Compose Multiplatform 的渲染引擎。

Skia 甚至能够作为 WASM 模块在浏览器中运行，这也是支撑 Compose Web 运行的法宝。

事实上绝大部分浏览器都是用的 Skia 作为图形引擎。但是由于脚本的缘故，这些接口并未在用户脚本层暴露出来，所以我们无法在 JS 中使用相关功能。
所以说再在用户层重写实现一个 Skia 还是有必要的。

我们是否也能在 ArkUI 层实现一个 Skia 的 API 呢？哪怕只是一个接口的简单 wrapper？  
理论上是可行的，但是目前在鸿蒙上只能是个**玩具**。JS 的**运行效率**相较于原生真是太差劲了！更何况这是一个可能计算密集的图形引擎。

<figure markdown="span">
    ![Kotlin/Wasm performance](https://kotlinlang.org/docs/images/wasm-performance-compose.png)
    <figcaption>[Kotlin/Wasm performance](https://kotlinlang.org/docs/wasm-overview.html#kotlin-wasm-performance)</figcaption>
</figure>

Compose Web 其实尝试过用 JS 作为 Skia 的实现语言，但是很快就取消了。
正是因为性能，Compose Multiplatform 现在还是换成了 WASM 实现。
就连在 V8 引擎上运行的 CMP/JS 最终都被抛弃了，更何况使用 QuickJS 的 ArkUI？

更遗憾的是，ArkUI 现在还不支持 WASM，无法走像现在 Compose Web 一样的道路。
非要写这个“玩具”也会很麻烦，曾经的 CMP/JS 的代码现在根本找不到了，或者说可能从未公开过。
如果想使用 [binaryen](https://github.com/WebAssembly/binaryen) 工具集的 `wasm2js` 工具实验性的将 WASM 转为 JS，
你也会发现根本无法使用，因为 WASM 的内存管理机制与 JS 的不同，转换后需要使用额外的 WASM 包来完成相关操作。而鸿蒙并没有这个包。

所以说这条道路理论上也是可行的，但我相信应该没人会这样做。这会付出非常大的经历来写这个 wrapper，而最终结果的性能也会不如意。

## 最后的总结

使用原生的 Skia 或许是在鸿蒙上运行 Compose Multiplatform 的最佳选择。
但仅仅实现了图形还不够，还有各种系统事件的处理，以及周边库的支持。

这项工作量非常庞大，这些东西单凭个人的力量是很难完成并持续发展的。
现在看来，或许将这一切交给时间是更为实际的选择。
等待未来某个时刻，有公司、团队、或一群热爱 Compose 的开发者组成的开源组织愿意接下这个沉重的任务。

希望这篇文章能成为一个引子，吸引更多充满热情的开发者加入开源贡献的行列！
