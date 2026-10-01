<div align="center">

# ⌚ Xiaomi Band Dev Guide (小米手环开发指南)

**专为小米手环 (Xiaomi Vela OS / HyperOS) 打造的表盘与快应用开发指南、避坑手册与最佳实践**
<br />
*The comprehensive engineering handbook, best practices, and battle-tested pitfalls for Xiaomi Smart Band watchface and QuickApp development.*

[![Device: Xiaomi Band](https://img.shields.io/badge/Device-Xiaomi%20Smart%20Band-FF6900?style=flat-square&logo=xiaomi&logoColor=white)](https://www.mi.com)
[![OS: Xiaomi Vela](https://img.shields.io/badge/OS-Xiaomi%20Vela%20%2F%20HyperOS-blue?style=flat-square)](https://iot.mi.com/vela/)
[![Framework: QuickApp](https://img.shields.io/badge/Framework-QuickApp%20(Vela%20JS)-00C4B4?style=flat-square)](https://iot.mi.com/vela/quickapp/)
[![Toolkit: aiot-toolkit](https://img.shields.io/badge/Toolkit-aiot--toolkit-orange?style=flat-square)](https://www.npmjs.com/package/aiot-toolkit)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg?style=flat-square)](LICENSE)
[![Vibe Coding](https://img.shields.io/badge/Built%20with-Vibe%20Coding-blueviolet?style=flat-square)](https://github.com/KaiTeeDreamChai)

</div>

---

## 📌 项目简介

在小米穿戴生态（以 **小米手环 9 Pro / 8 Pro / 小米手环 9** 等搭载 Xiaomi Vela OS / HyperOS 的设备为代表）上开发表盘或快应用（QuickApp）时，开发者普遍面临**官方资料零散、闭源黑盒较多、缺少系统级指引**的痛点。

不少开发者习惯了普通前端或手机端开发思维，极易陷入**“伪 Web 幻觉”**（误以为有 DOM 树、window 或 Node.js 环境），经常遭遇**内联样式崩塌、循环绑定空白、息屏无故丢数据、安装包体积超限、蓝牙侧载证书报错**等棘手难题。

`Xiaomi Band Dev Guide` 是一份基于**真实手环项目实战淬炼**（以自研 35KB 全量 3500 字频拼音输入法与独立记事本 [BandNotepad](https://github.com/KaiTeeDreamChai/band-notepad) 为实战依托）沉淀而成的开源开发手册与避坑指南。旨在系统化拆解 Vela 快应用底层原理、表盘制作体系、穿戴 UI 规范、性能红线与真机部署技巧，帮助开发者与 AI Agent 快速跨越嵌入式穿戴开发的深水区！

---

## ✨ 核心特性与指引体系

- 🏗️ **底层运行机制全景透视**：深入剖析 Vela QuickApp C++ Lite 引擎本质，明确其非浏览器特性的执行边界，从原理上消除排版与语法错误。
- 🎨 **表盘与快应用双轨联动**：详解 EasyFace / Mi-Create 表盘制作工具链、AOD 息屏 10% OPR 功耗红线，以及通过表盘点击热区（Hotspot）无缝拉起快应用的工程范式。
- 📐 **双屏形态工程适配**：针对手环 9 Pro（336×480 微曲面）与手环 9 标准版（192×490 细长跑道屏）分别确立安全边距、40px+ 防误触热区与 AMOLED 纯黑节能配色。
- 💾 **穿戴无后台与即时存盘定律**：深度拆解穿戴系统随时被系统挂起与 Watchdog 强杀的生命周期特性，确立“变动即持久化”的零丢失数据架构。
- 🚫 **十大实战避坑红线速查**：涵盖模板语法循环 `$item` 绑定、Emoji 代理对安全删除、安装包 50KB 内存预算、单页状态机架构等高频血泪踩坑点。
- 🚀 **蓝牙侧载与部署闭环**：详述利用 AstroBox / 表盘自定义工具进行无线分包推送的完整流程，阐明 AuthKey 云端提取、GATT 冲突预防与 Debug 签名的决定性作用。

---

## 🔧 工作原理与穿戴系统架构

```
┌────────────────────────────────────────────────────────┐
│            开发者编写的 UX 代码 (.ux / JS / CSS)         │
└────────────────────────────────────────────────────────┘
                           │  (aiot-toolkit Webpack 编译打包)
                           ▼
┌────────────────────────────────────────────────────────┐
│        RPK 穿戴包 (Bytecode / AST / 静态矢量与字库)      │
└────────────────────────────────────────────────────────┘
                           │  (AstroBox 蓝牙侧载 / 运行时装载)
                           ▼
┌────────────────────────────────────────────────────────┐
│             Xiaomi Vela QuickApp C++ Engine            │
│  ├── 嵌入式 JS 解释器 (JerryScript / QuickJS Lite)      │
│  ├── Flexbox 布局计算与原生 2D 绘图 (类似 LVGL 管线)   │
│  └── 系统桥接层 (@system.storage / vibrator / router)  │
└────────────────────────────────────────────────────────┘
       │                   │                   │
       ▼                   ▼                   ▼
[ 闪存持久化 (Flash) ]   [ 触觉震动马达 ]   [ 336×480 AMOLED 屏 ]
```

---

## 📚 专题指南目录 (Documentation)

| 章节 | 文档名 | 核心内容简介 |
| :--- | :--- | :--- |
| **01** | [**平台架构与渲染底座**](docs/01-platform-and-engine.md) | 手环系统底座演进、C++ Lite 引擎运转机制、物理约束矩阵与无后台铁律 |
| **02** | [**快应用工程与核心 API**](docs/02-quickapp-development.md) | 标准工程解剖、`manifest.json` 关键配置、UX 组件编写边界与核心 API 实战 |
| **03** | [**UI 设计与交互规范**](docs/03-ui-and-interaction.md) | 9 Pro vs 9 标准版双屏适配、微曲面安全区、40px+ 触控热区与 AMOLED 节能 |
| **04** | [**核心避坑红线速查表**](docs/04-pitfalls-and-solutions.md) | 真实踩坑十大金科玉律（伪 Web 幻觉、模板绑定、息屏丢数据、Emoji 切片等） |
| **05** | [**蓝牙侧载与真机部署**](docs/05-sideload-and-deployment.md) | AstroBox 推包流程、AuthKey 获取、GATT 冲突预防与真机部署故障排查表 |
| **06** | [**表盘开发与制作指南**](docs/06-watchface-development.md) | EasyFace / Mi-Create 工具链、切图规范、AOD 息屏 10% OPR 红线与快应用唤醒联动 |

---

## 🚀 快速开始

如果你想在本地新建一个小米手环快应用项目：

1. **环境准备**：
   确保本地已安装 Node.js (推荐 v18+)，全局安装官方构建工具 `aiot-toolkit`：
   ```bash
   npm install -g aiot-toolkit
   ```

2. **初始化工程与安装依赖**：
   ```bash
   mkdir my-band-app && cd my-band-app
   npm init -y
   npm install aiot-toolkit --save-dev
   ```

3. **配置构建脚本 (`package.json`)**：
   ```json
   {
     "scripts": {
       "build": "aiot build",
       "release": "aiot release"
     }
   }
   ```

4. **编译生成安装包**：
   ```bash
   # 编译用于 AstroBox 蓝牙侧载安装的 Debug 包
   npm run build

   # 编译用于正式签名的 Release 包
   npm run release
   ```
   构建生成的 `.rpk` 文件位于 `dist/` 目录下。

---

## 💡 核心开发速查表 (Quick Reference)

### 1. 界面与交互基准数据
- **物理基准宽度**：
  - 手环 9 Pro / 8 Pro：`designWidth: 336`（推荐画布 `336px × 480px`，有效内容宽 `308px`）
  - 手环 9 标准版：`designWidth: 192`（推荐画布 `192px × 490px`，有效内容宽 `176px`）
- **触控按键最低高度**：`≥ 40px`（手势盲按推荐 `44px ~ 48px`）
- **纯黑背景能耗底色**：全屏一律使用 `background-color: #000000;`

### 2. 常用 API 代码片段

#### 本地即时存盘：
```javascript
import storage from '@system.storage';

storage.set({
  key: 'user_note_data',
  value: JSON.stringify(dataObj)
});
```

#### 机械短震反馈：
```javascript
import vibrator from '@system.vibrator';

try {
  vibrator.vibrate({ mode: 'short' });
} catch (e) {}
```

#### 返回手势拦截：
```javascript
export default {
  onBackPress() {
    if (this.isModalOpen) {
      this.isModalOpen = false;
      return true; // 拦截返回手势，仅关闭弹窗
    }
    return false; // 放行退出应用
  }
};
```

---

## ⚠️ 注意事项与系统边界

1. **内存与包体极度受限**：RPK 安装包严格控制在 **50KB 以内**，运行期内存开销应保持平稳常数，严禁引入未经压缩的几兆大词库或高清位图。
2. **单页状态机优于多页面栈**：尽量采用单页 `pages/index/index.ux` 结合数据状态流转（列表态、编辑态、弹窗态）实现全套业务，避免频繁使用 `router.push` 开辟新页面导致手环 RAM 耗尽。
3. **侧载兼容性**：零售版手环通过蓝牙工具（AstroBox 等）侧载时，**必须使用 `npm run build` 生成的 Debug 包**，手环固件通常禁止未授权的自签名 Release 包。

---

## 🏗️ 仓库结构

```
xiaomi-band-dev-guide/
├── docs/
│   ├── 01-platform-and-engine.md       # 平台架构与 Vela 渲染引擎解析
│   ├── 02-quickapp-development.md      # 快应用工程结构与核心 API 实践
│   ├── 03-ui-and-interaction.md        # 9 Pro vs 9 标准版双屏 UI 规范与 AMOLED 节能
│   ├── 04-pitfalls-and-solutions.md    # 核心避坑红线速查表 (Top 10)
│   ├── 05-sideload-and-deployment.md   # 蓝牙侧载、AuthKey 提取与部署故障排查 (AstroBox)
│   └── 06-watchface-development.md     # 表盘制作全景指南 (EasyFace/Mi-Create/AOD规范)
├── LICENSE                             # MIT 开源许可证
└── README.md                           # 全景开发指南与速查手册
```

---

## ☕ 关于本项目 (About)

本项目为 **Vibe Coding / 实战经验驱动** 产物，旨在为小米穿戴生态的开发者提供一份真正接地气、经过真机千锤百炼的实战指南。
- **项目定位**：开箱即用、无商业冗余、专注于穿戴式快应用与表盘工程开发。
- **配套实战项目**：[BandNotepad (小米手环 9 Pro 腕上记事本与 T9 输入法)](https://github.com/KaiTeeDreamChai/band-notepad)。
- **共建说明**：欢迎提交 Issue 补充你遇到的手环特色 Bug 或提交 PR 完善更多手环机型（手环 8 / 9 标准版、手表系列）的开发经验！

---

## 📄 License

本项目基于 [MIT License](LICENSE) 开源。
