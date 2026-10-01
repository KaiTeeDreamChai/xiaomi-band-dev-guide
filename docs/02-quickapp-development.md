# 02. 快应用工程搭建、生命周期与核心 API 实践

> 本章提供基于 `aiot-toolkit` 的小米手环快应用标准工程范式，详述 `manifest.json` 配置边界及核心系统模块的使用要领。

---

## 一、标准工程结构与文件规范

一个标准的 Vela 快应用工程目录如下：

```text
band-app/
├── src/
│   ├── manifest.json              # 核心应用清单配置 (权限、路由、物理基准)
│   ├── app.ux                     # 应用级生命周期入口与全局逻辑
│   ├── common/                    # 公共静态资源与通用工具库
│   │   ├── images/logo.png        # 应用高清图标 (通常为 PNG 格式)
│   │   └── js/                    # 纯原生 JS 业务模块
│   └── pages/                     # 业务页面目录
│       └── index/
│           └── index.ux           # 页面级组件 (模板 / 样式 / 逻辑)
├── sign/                          # 签名密钥对 (release 构建必备)
│   └── release/
│       ├── certificate.pem
│       └── private.pem
├── quickapp.config.js             # Webpack 编译扩展配置
├── package.json                   # npm 脚本与依赖清单
└── .gitignore
```

---

## 二、`manifest.json` 核心配置解析

`manifest.json` 是整个快应用的身份证，配置错误将直接导致手环解析包失败或权限缺失：

```json
{
  "package": "com.xiaomi.band.notepad",
  "name": "腕上记事本",
  "versionName": "1.6.0",
  "versionCode": 8,
  "minPlatformVersion": 1000,
  "icon": "/common/images/logo.png",
  "deviceTypeList": [
    "watch"
  ],
  "features": [
    { "name": "system.router" },
    { "name": "system.app" },
    { "name": "system.storage" },
    { "name": "system.vibrator" }
  ],
  "config": {
    "logLevel": "log",
    "designWidth": 336
  },
  "router": {
    "entry": "pages/index",
    "pages": {
      "pages/index": {
        "component": "index"
      }
    }
  },
  "display": {
    "backgroundColor": "#000000"
  }
}
```

### 关键字段要点说明：
1. **`deviceTypeList`**：必须显式声明包含 `"watch"`，否则手环系统包管理器可能拒绝安装。
2. **`designWidth`（设计宽度）**：
   - 对于小米手环 9 Pro / 8 Pro，必须设为 **`336`**。
   - 所有在 CSS 中编写的 `px` 单位将 1:1 映射至手环物理像素，杜绝任何缩放插值导致的字体模糊。
3. **`features`（系统模块授权）**：未在此处声明的系统模块，在代码中通过 `import ... from '@system.xxx'` 引入时会直接抛出 `undefined` 或调用静默失效。

---

## 三、单文件组件 (`.ux`) 的编写边界

快应用采用类似 Vue 选项式 API（Options API）的语法，但具有严格的子集限制。

### 1. 结构框架
```html
<template>
  <div class="root">
    <!-- 顶栏 -->
    <div class="top-bar">
      <text class="title">{{title}}</text>
    </div>
    <!-- 列表内容 -->
    <div for="{{list}}" class="item-card" onclick="tapItem($idx)">
      <text class="item-name">{{$item.name}}</text>
    </div>
  </div>
</template>

<style>
  .root {
    width: 336px;
    height: 480px;
    background-color: #000000;
    flex-direction: column;
  }
  .title {
    font-size: 16px;
    color: #ffffff;
    font-weight: bold;
  }
</style>

<script>
  import vibrator from '@system.vibrator';
  import storage from '@system.storage';

  export default {
    private: {
      title: '我的应用',
      list: []
    },
    onInit() {
      // 页面初始化
    },
    onShow() {
      // 页面显示或手环重新亮屏
    },
    onHide() {
      // 页面隐藏或息屏
    },
    onDestroy() {
      // 页面销毁
    },
    vib() {
      try {
        vibrator.vibrate({ mode: 'short' });
      } catch (e) {}
    }
  };
</script>
```

### 2. 核心语法约束：
- **容器布局基石**：只能使用 `<div>` 进行布局排版，默认采用 **Flexbox** 模型，推荐显式指定 `flex-direction: column` 或 `row`。
- **文字必须包裹于 `<text>`**：任何裸露在 `<div>` 内的纯文本字符串均不会被渲染，必须包裹在 `<text>` 标签内。
- **模板循环指令**：
  - 格式必须为 `<div for="{{list}}">`。
  - **当前循环项必须使用 `$item`，当前索引必须使用 `$idx`**。
- **条件渲染**：推荐使用 `if="{{condition}}"`。

---

## 四、核心系统模块实战要领

### 1. 本地存储 `@system.storage`
用于在手环闪存（Flash）上持久化存储键值对。

```javascript
import storage from '@system.storage';

// 存盘写入
function saveData(key, obj) {
  try {
    storage.set({
      key: key,
      value: JSON.stringify(obj),
      success: () => {},
      fail: (data, code) => {
        console.error('Storage set failed: ' + code);
      }
    });
  } catch (e) {}
}

// 读取数据
function loadData(key, defaultVal, callback) {
  try {
    storage.get({
      key: key,
      default: JSON.stringify(defaultVal),
      success: (val) => {
        try {
          callback(JSON.parse(val));
        } catch (e) {
          callback(defaultVal);
        }
      },
      fail: () => {
        callback(defaultVal);
      }
    });
  } catch (e) {
    callback(defaultVal);
  }
}
```

> **经验法则**：所有存储读写都是异步回调机制。在修改便签、切换状态等关键交互触发时，先同步修改内存数据（`this.xxx = ...`），随后立即调用 `saveStorage()`，绝不等待异步返回才刷新 UI。

---

### 2. 物理震动反馈 `@system.vibrator`
赋予小屏幕机械确认感的关键利器。

```javascript
import vibrator from '@system.vibrator';

// 常用短震动
function vibShort() {
  try {
    vibrator.vibrate({ mode: 'short' });
  } catch (e) {}
}

// 警告/长震动
function vibLong() {
  try {
    vibrator.vibrate({ mode: 'long' });
  } catch (e) {}
}
```

> **经验法则**：
> - 每次按键点击、选项切换，首选 `mode: 'short'`（轻微清脆短震）。
> - 涉及重要操作提示（如删除确认、达到极限翻页）可触发长震或短震两次。
> - 在按键处理函数的第一行直接触发震动，响应速度最佳。

---

### 3. 硬件按键拦截与手势返回 `onBackPress()`
手环右滑手势默认会触发“返回上一级”或“退出应用”。

```javascript
export default {
  // ...
  onBackPress() {
    // 场景 1：如果当前弹出了全屏模态窗口，优先关闭弹窗，不退出应用
    if (this.isCandExpanded) {
      this.isCandExpanded = false;
      return true; // 拦截返回手势
    }
    // 场景 2：如果软键盘正处于呼出状态，优先收起键盘
    if (this.showKeyboard) {
      this.showKeyboard = false;
      return true; // 拦截返回手势
    }
    // 场景 3：如果处于二级视图（如正在编辑），返回至一级列表并自动存盘
    if (this.viewMode === 'edit') {
      this.backToList();
      return true; // 拦截返回手势
    }
    // 场景 4：一级视图允许系统放行，正常退出手环应用
    return false;
  }
};
```

---

## 五、构建命令与打包脚本全景

在 `package.json` 中配置官方构建命令：

```json
{
  "scripts": {
    "build": "aiot build",
    "release": "aiot release"
  },
  "devDependencies": {
    "aiot-toolkit": "^2.0.5"
  }
}
```

| 命令 | 产物文件名规则 | 签名证书来源 | 适用场景 |
| :--- | :--- | :--- | :--- |
| **`npm run build`** | `dist/<pkg>.debug.<version>.rpk` | SDK 内置默认开发证书 | **蓝牙侧载首选**（AstroBox 等工具默认只认该签名的安装包） |
| **`npm run release`** | `dist/<pkg>.release.<version>.rpk` | `sign/release/` 下的自有证书对 | 正式生产分发、官方应用商店上架 |

> **提示**：每次版本升级前，务必同步递增 `manifest.json` 中的 `versionName` 与 `versionCode`。
