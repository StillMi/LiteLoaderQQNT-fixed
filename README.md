# LiteLoaderQQNT-fixed

基于官方 **LiteLoaderQQNT 1.4.1** 的最小修复版：只改一行代码，解决 **QQNT 9.9.36（Electron 40）下「插件目录里只要有插件，QQ 启动就弹主进程 JS 错误、无法启动」** 的问题。

## 症状

按官方方式装好 LiteLoaderQQNT 1.4.1、往 `plugins` 目录里放进任意插件后，启动 QQ 立即弹出：

```
A JavaScript error occurred in the main process
Uncaught Exception:
TypeError [ERR_INVALID_ARG_TYPE]: The path argument must be of type string. Received undefined
    at Object.scanPluginDirectory (...\QQNT\resources\app\LiteLoaderQQNT\src\main\store.js:104:28)
```

`plugins` 目录为空时一切正常，只要有一个插件就崩——这与具体是哪个插件无关。

## 根因

- QQNT 9.9.36 内置 Electron 40.0.0。它使用的 Node 运行时已经移除了 `fs.Dirent` 的 `path` 属性（`readdirSync` 返回的 `Dirent` 对象上 `dirent.path === undefined`）。
- LiteLoaderQQNT 1.4.1 在 `src/main/store.js:104` 用 `dirent.path` 拼插件 manifest 路径：

  ```js
  fs.readdirSync(LiteLoader.path.plugins, { withFileTypes: true })
      .map(dirent => path.join(dirent.path, dirent.name, "manifest.json"))
  ```

- `path.join(undefined, ...)` 抛出 `ERR_INVALID_ARG_TYPE`，且发生在主进程加载插件阶段、没有被捕获，于是 QQ 卡在错误弹窗上无法继续启动。

## 修复

`src/main/store.js` 单行改动：

```diff
-        .map(dirent => path.join(dirent.path, dirent.name, "manifest.json"))
+        .map(dirent => path.join(LiteLoader.path.plugins, dirent.name, "manifest.json"))
```

`LiteLoader.path.plugins` 正是刚被读取的父目录，与 `dirent.path` 语义等价，且不依赖已被移除的 Node API，跨版本稳定。

完整补丁见 [`patches/fix-dirent-path.patch`](patches/fix-dirent-path.patch)。

## 安装 / 使用

前提：已按官方流程安装 LiteLoaderQQNT 1.4.1（资源目录 `resources/app/LiteLoaderQQNT` + 版本目录 `package.json` 指向 `ml_install.js` 引导）。

任选一种：

1. **只替换文件（推荐）**：用本仓库的 `src/main/store.js` 覆盖
   `C:\Program Files\Tencent\QQNT\resources\app\LiteLoaderQQNT\src\main\store.js`，然后重启 QQ。
2. **整目录覆盖**：下载 [`dist/LiteLoaderQQNT-1.4.1-fixed.zip`](dist/LiteLoaderQQNT-1.4.1-fixed.zip)，解压覆盖
   `...\resources\app\LiteLoaderQQNT`（内含 `src/`、`package.json`、`LICENSE`，不含 `data/`、`plugins/`，不会覆盖你的插件和配置）。

   该 zip 的 SHA-256：`f6a36c813a4417557a58ce771d69035e3ddc1f044fb6cbb3963622cf4825f6f6`（31,606 bytes）。

重启 QQ 后，设置页能显示「LiteLoaderQQNT 1.4.1」即安装成功。

## 已验证环境（实测）

| 项目 | 值 |
| --- | --- |
| 系统 | Windows 10/11 x64 |
| QQNT | 9.9.36-53644 |
| LiteLoaderQQNT | 1.4.1（官方发布包） |
| Electron | 40.0.0 |
| 验证结果 | 插件正常加载、QQ 正常启动；恢复未修复的 `store.js` 后必现上述弹窗 |

验证方式：安装第三方插件（anti_recall）后启动 QQ，设置页显示加载器版本与插件列表，主进程日志正常输出插件日志。

## 说明

- 除 `src/main/store.js` 中上述一行外，本仓库所有文件与官方 **1.4.1 发布包（tag 1.4.1, commit `b1fd831`）** 逐字节一致。
- 这是**非官方**修改版，仅为兼容性补丁；上游仓库：<https://github.com/LiteLoaderQQNT/LiteLoaderQQNT>。
- QQ 更新或重装会覆盖加载器文件，届时重新替换一次 `store.js` 即可（本仓库可长期保留）。
- 该崩溃与插件内容无关，任何插件都会触发；如果你遇到的是 `SyntaxError: Unexpected token` / `is not valid JSON`，那是插件 `manifest.json` 带了 UTF-8 BOM，与本补丁无关（manifest 请保存为无 BOM 的 UTF-8）。

## License

MIT，版权归上游原作者所有，见 [LICENSE](LICENSE)。
