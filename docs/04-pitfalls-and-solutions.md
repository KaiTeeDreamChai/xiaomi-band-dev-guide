# 04. 核心避坑红线速查表 (Top 10 Pitfalls & Solutions)

> 本章凝结了数百次真机调试与崩溃分析所得出的**十大血泪经验**，每一条都是曾经踩过的真实深坑与终极解决方案。

---

## 🚫 坑点 1：伪 Web 幻觉（严禁调用标准浏览器 API）

- **症状**：代码中书写了 `window.xxx`、`document.getElementById`、`fetch()` 或 `localStorage`，编译虽然通过，但在手环启动瞬间直接白屏崩溃。
- **原因**：Vela QuickApp 运行于轻量嵌入式 C++ 引擎上，根本没有浏览器宿主环境。
- **解法**：
  - 数据持久化一律使用 `import storage from '@system.storage'`。
  - 网络请求一律使用 `import fetch from '@system.fetch'`。
  - 严禁任何试图操作底层 DOM 或挂载全局对象的代码。

---

## 🚫 坑点 2：在 `<text>` 中嵌套带有内联样式的 `<span>`

- **症状**：希望给局部文字设置下划线或高亮颜色，书写了 `<text>你好<span style="color:red">世界</span></text>`，真机上红色文字直接消失、文本折叠或排版错乱。
- **原因**：Vela 的 C++ 文本渲染器（类似 LVGL Label）只支持扁平单字体的单行/多行绘制，对嵌套 inline-span 的样式解析极不完整。
- **解法**：
  - **采用字符占位拼接法**：例如实现下划线光标，在 JS 层直接拼接字符串：
    ```javascript
    this.displayText = this.currentText.slice(0, this.cursorPos) + '_' + this.currentText.slice(this.cursorPos);
    ```
  - **采用独立组件横向排版**：必须分色显示的内容，拆分为多个平行 `<text>` 并用 `<div style="flex-direction: row">` 包裹。

---

## 🚫 坑点 3：QuickApp 模板循环必须显式使用 `$item` 与 `$idx`

- **症状**：在 `<div for="item in list">` 中访问 `{{item.name}}`，界面渲染全部为空白。
- **原因**：快应用标准的 AST 模板编译器在部分 Vela 版本中不识别别名定义，只向循环体内注入 `$item` 和 `$idx`。
- **解法**：
  - 循环模板必须严格写为：
    ```html
    <div for="{{list}}" onclick="selectItem($idx)">
      <text>{{$item.name}}</text>
    </div>
    ```

---

## 🚫 坑点 4：依赖“离开页面才存盘”导致息屏丢数据

- **症状**：用户在手环上打了一段文字，手腕下垂息屏，再次点亮重新打开应用时，刚才输入的内容全部丢失。
- **原因**：手环系统在息屏时直接调用 `onHide`，随后后台随时可能被 watchdog 杀掉，`onDestroy` 甚至根本来不及执行。
- **解法**：
  - **实行“变动即存盘”原则**：每一次汉字上屏、每一次按键退格、每一次新建便签，在修改内存数据的同时立即调用 `this.saveStorage()`。
  - 在 `onHide` 和 `onDestroy` 生命周期中加一道兜底持久化。

---

## 🚫 坑点 5：滥用多页面路由导致内存溢出与黑屏

- **症状**：应用做了多个独立页面，使用 `router.push` 频繁来回跳转，几分钟后手环突然卡死重启。
- **原因**：每个新页面都会在 C++ 层开辟新的视图上下文与渲染画布，手环总内存有限，页面栈积压将瞬间耗尽 RAM。
- **解法**：
  - **拥抱单页状态机（Single-Page App）**：整个应用只保留一个核心 `pages/index/index.ux`。
  - 通过数据变量（如 `viewMode: 'list' | 'edit'`、`isCandExpanded: boolean`）切换显隐，复用同一个根容器，内存永远保持在水平常数。

---

## 🚫 坑点 6：光标切片删除与 Emoji 代理对乱码

- **症状**：文本中若包含 Emoji 表情，按一次退格删除后，表情变成乱码问号 ``，甚至导致后续渲染崩溃。
- **原因**：JavaScript 中标准字符长度为 1，但大多数 Emoji 属于 UTF-16 代理对（Surrogate Pair），占用 2 个代码单元。单字节切片会直接把 Emoji 劈开。
- **解法**：
  - 退格删除时智能检测前导码元：
    ```javascript
    let delLen = 1;
    if (this.cursorPos >= 2) {
      const code = this.currentText.charCodeAt(this.cursorPos - 1);
      const prevCode = this.currentText.charCodeAt(this.cursorPos - 2);
      if (code >= 0xDC00 && code <= 0xDFFF && prevCode >= 0xD800 && prevCode <= 0xDBFF) {
        delLen = 2; // 识别为代理对，安全删除 2 字节
      }
    }
    this.currentText = this.currentText.slice(0, this.cursorPos - delLen) + this.currentText.slice(this.cursorPos);
    this.cursorPos -= delLen;
    ```

---

## 🚫 坑点 7：安装包膨胀引发真机解析失败（OOM）

- **症状**：本地编译生成的 RPK 超过 100KB，在 AstroBox 传输时握手失败，或者安装后点击启动闪退。
- **原因**：穿戴式快应用安装包解包需要解压至内存缓存区，体积过大直接触发内核 OOM 保护。
- **解法**：
  - **安装包体积红线**：严格压制在 **50KB 以内**（目前本项目维持在 30~36KB）。
  - **字库压缩**：放弃庞大第三方汉字库，采用紧凑拼音索引表与字典字符串。
  - **图片瘦身**：严禁放入未压缩大图，图标必须经过 TinyPNG 极限压缩或采用纯 CSS 矢量绘制。

---

## 🚫 坑点 8：边缘手势与内部横向滑动死锁

- **症状**：在屏幕边缘设计了横向滚动或横向滑动手势，用户滑动时频繁触发手环系统的“全局右滑返回”，根本无法操作内部组件。
- **原因**：系统级右滑手势优先级极高，距离屏幕左侧边缘 0~20px 的滑动直接被系统底层拦截。
- **解法**：
  - 水平排列的控制按钮必须设置至少 `14px` 的外边距。
  - 针对翻页操作，**推荐使用明确的按钮点击（‹ / ›）**，而非依赖不稳定的大幅度水平手势滑动。

---

## 🚫 坑点 9：AstroBox / 侧载工具无法安装 Release 签名包

- **症状**：使用 `npm run release` 打包生成的包，在 AstroBox 或表盘自定义工具上推包报错或手环提示“证书非法”。
- **原因**：零售版手环未写入个人私钥的证书信任链，Release 包通常需要厂商官方签名服务。第三方侧载工具大多利用了固件的“开发调试豁免”，只放行 Debug 签名的包。
- **解法**：
  - 侧载部署**必须优先提供并使用 `npm run build` 生成的 Debug 包**（如 `com.xiaomi.band.notepad.debug.1.6.0.rpk`）。

---

## 🚫 坑点 10：真机无黑盒调试界面的日志盲区

- **症状**：代码在模拟器正常，真机上点击某个功能没反应，不知何处抛出了异常。
- **原因**：普通手环未开启 USB 串口调试，开发者无法打开 Chrome DevTools 查看 Console。
- **解法**：
  - **全链路 try-catch 保护**：所有外部模块调用（如 `vibrator`、`storage`、`JSON.parse`）均用 `try-catch` 包裹，并在 catch 中设置回退默认值。
  - **UI 错误看板法**：在调试阶段，在界面顶部临时放置一个隐藏的 `<text class="debug-log">{{debugMsg}}</text>`，发生异常时直接将 `e.message` 打印在手环屏幕上，真机问题一秒现形！
