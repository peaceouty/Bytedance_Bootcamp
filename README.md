# 主程序

其它插件的入口、设置页、悬浮球

## 主程序约定使用
- 渲染进程：Vue 3 Composition + vue-router
- UI 组件：Element Plus（利用它的i18n与深浅色模式支持）
- 窗口配置：`contextIsolation: false`，`sandbox: false`

## 目录结构
```shell
demo-plugin/          # 示例插件
src/                  # 源码目录
  main/               # 主进程相关代码
    plugins/          # 插件管理与内置插件
  preload/            # 预加载脚本
  renderer/           # 渲染进程
    src/              # 前端源码
      windows/        # 各窗口页面
        entrance/     # 入口窗口
        settings/     # 设置窗口
  share/              # 主/渲染进程共享类型与工具
    plugins/          # 插件相关类型定义
```

## 添加新窗口

当前把应用内的窗口也视为插件，方便管理和加载。

1. 在 `src/renderer/windows/<窗口名>/index.html` 创建 `index.html`（和`main.ts` `App.vue` 等文件，如果需要使用vue）
2. 在 `src/main/plugins/builtin.ts`中添加窗口（也是一个插件）的元数据：（其它跟普通插件共享的字段见文末）
   ```ts
   {
     id: 'builtin.mywindow',
     name: 'MyWindow',
     dist: '<窗口名>',  // 和 windws/<窗口名> 对应
     internal: {       // 标记这是一个内置窗口
        hidden: true  // （可选）在插件列表中隐藏这个窗口
     }
   }
   ```
3. 需要时通过插件机制打开窗口：
   ```ts
   await pluginManager.open('builtin.mywindow')
   ```

## 窗口跳转
使用 `pluginManager.open(id)` 在主进程中加载并显示对应窗口，实现不同页面间的切换。

## 应用设置的方案
- 持久化方案？
- 主进程与渲染进程如何状态同步？

# 插件开发

当前定义：插件是**任意**提供了 `plugin.json` `preload.js` `index.html` 的目录，相当于于electron中一个renderer。
目录中
- `index.html`是插件被激活时的窗口显示的内容。
- `preload.js`是插件的预加载脚本，等同于electron的`preload.js`，要求为CommonJS模块。
- `plugin.json`是插件的元数据，包含插件的ID、名称、版本等信息。

## 插件生命周期
![](https://mermaid.ink/img/pako:eNqFVMtO20AU_RVrELsQ2XHsJG6FBMRmValS6aZOF248ebSOHTl2gSKkCAHBpCK0VCBBqaAPlXYBVR_QJDx-xjMOf9HxIwFRqnph-dw599x7z5VnDuQNFQIBFDRjOl9STIuayuZ0ijzDwxQ63HbbDt47QaetMJjXlFotCwuUXYOm-BzqFlUoa5owJPJSSpJiNcs0nkFhiEln6AEcmS6rVklIVGfu3FCpKGV9gjQQiUiSlBL5gYg0lp5g-P-KmFBXoXldhpckMTuQ4RITSWbsNpnBoL3VBW-hHY4bBh_KuPXa7R6j9g_U-uYdNFGnhdaP0OrBY0EQBtOH5DFGRkdtt7OGnTp-66CT74TqvTnwthdDldtyEjI6raPWq159CTtNItw7O0NLn8IEvNnwi39s35I5LmPnc2__JVU1oWYoavxpzWf1vQxJWZnUxysn_liNbjTLzjnea9x9Yo7ilc3w2O2u9c43_LL7y5d1Bze__LNhUTb0-5pdLOuibkGTcjvNoMVFb3357_pS5J_3dQu1PqCVHdTt-KyrZYW8SRmt7pHJqTKJz8RLVkW7Sbva0sU7r3MR7YcaGRklxvcXEMDxvrXX0TgBJLXhdpbd33V8-AtvHV9u_fRtCD3AzgbZBDkOO77cfY93LwKF6E_IBkCMbAiAFE0ZgMmcDmKgAk1igUr-pTn_MAesEqzAHBDIpwoLiq1ZOZDT5wlVsS3jwayeB4Jl2jAGTMMulvrArqqKBbNlpWgqFSAUFK1GolVFf2QYlT6paPqVouzIK1u3gJDm2IAMhDkwAwSWp-MZlkvQGY5hU6lkho-BWSDwbJxnk2maSdBskubSzHwMvAjk6TjHc4kUneFpliVvlokBqJYtw7wXXhTBfdHvUgxOoibn_wD6vbpc?type=png)

注：
- onPluginEnter是在preload.js中利用平台API注册的事件，**当前未实现平台API**


## 约束
1. 插件即是一个独立的electron renderer，但是相对于Electron默认的安全策略，有如下调整
    - `contextIsolation: false`
      关闭上下文隔离，preload脚本可以直接通过window向index.html挂载所需要的Electron API或其它对象。
      虽然牺牲了安全上的最佳实践，但是相比开启隔离，关闭后向`mainWorld`暴露的API就不限制于可被完美序列化的对象，不用担心序列化导致外部模块的API出问题。
    - `sandbox: false`
      关闭沙盒模式，插件可以直接使用Node.js API。

    这么做有个另外的理由：<del>utools也干了</del>
    所以它把安全压力转移到“插件商店的审核”上，要求preload.js不加密、不混淆。    

2. 插件的`preload.js`必须是`CommonJS`模块，不能使用ESM。
    可通过`tsconfig.json`配置为`"moduleResolution": "node"`来确保。    
    `electron`的`preload.js`虽然支持ESM，但是有额外的[注意事项](https://www.electronjs.org/docs/latest/tutorial/esm)。所以还是让在Node.js中运行的`preload.js`使用`CommonJS`风味吧。

## 插件的`plugin.json`定义
（将会保持更新，随时根据需要调整）
```json
{
  "id": "your.plugin.id",                // 插件唯一标识符
  "name": "插件名称",                     // 插件名称
  "version": "1.0.0",                    // 插件版本
  "window": {                            // （可选）窗口配置
    "width": 800,                        // （可选）窗口宽度
    "height": 600,                       // （可选）窗口高度
    "disableTransition": false,           // （可选）禁用窗口动画
    "frame": true,                        // （可选）是否显示窗口边框，默认true
    "transparent": false                  // （可选）是否透明窗口，默认false
  }
}
```