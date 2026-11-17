# 笔记：首页双列图标 / 进度条黑线 排查记录（2026-11-17）

> 结论先行：这两个问题都**不是 DYYY 的代码 bug**，而是热更新覆写补丁里的旧键值
> 覆盖了服务器的新结构。本文档记录定位过程与结论，供以后排查参考。

---

## 一、首页「首页 ⇄」双列图标不显示

### 现象

- 不注入 DYYY：首页 Tab 右侧有 ⇄ 小图标
- 注入 DYYY：图标消失（同位置空的）

### 运行时证据

对同一个 `UIImageView` 做对比（`AWENormalModeTabBarFeedView` 内 8×8 的那个）：

| 状态 | `FeedView` 子视图数 | `UIImageView.image` |
|---|---|---|
| 不注入 | 3 | `{8, 8.33}`（正常） |
| 注入 + 旧补丁 | 2 | — （容器都没有） |
| 注入 + 修好后的补丁 | 3 | `OK` |

### 图标链路（新版头文件）

```
AWEHPChannelTabBarItemModel.rightIndicatorImage
  → -[AWENormalModeTabBarItemConfigAbilityImpl updateRightElementButtonViewModels:]
  → -[AWENormalModeTabBarFeedView updateRightIndicatorWithImage:size:animated:]
  → -[AWENormalModeTabBarFeedView addRightIndicatorViewWithImage:size:]
  → rightIndicatorContaienrView(16×16) → rightIndicatorImageView(8×8)
```

### 元凶（逐个键二分定位）

| 补丁键 | 结果 |
|---|---|
| `c2_recommend_feed_entry_style`（旧结构 `new_entry`/`entry_detail`） | 图标容器不创建（子视图 3→2） |
| `homepage_long_press_setting` | 同样让子视图 3→2 |
| 其余 45 个键 | 无影响 |

- 服务器现在下发的 `c2_recommend_feed_entry_style` 是**新结构**，图标地址在
  `button_detail.icon_mid` 里；补丁用旧结构整体替换后，`button_detail` 丢失，
  抖音拿不到图片 → `image = null`。
- `homepage_long_press_setting` 会让 `AWEFeedDoubleColumnManager` 直接不创建
  右侧指示器（`canShowC2IconAndEnterDoubleColumnChannel` 相关路径）。

### 修法（如果以后还要修）

把这两个键从补丁里删掉，交回服务器下发即可：

```json
"c2_recommend_feed_entry_style"   // 删除
"c2_feed_switch_preview_entry"    // 删除
"double_column_recommend_exp"     // 删除
"homepage_long_press_setting"     // 删除
```

### 排除项（都不是原因）

- 自定义底栏高度 40（`hp_tab_bar_custom_height_config`）
- DYYY 全屏、隐藏朋友/加号/商城按钮、底部背景隐藏
- 「禁止下发配置」开关状态
- `AWEDCFeedSwitchPreViewViewSettingManager` 的双列入口开关（实测 =1 一直是开）

---

## 二、滑动视频时进度条出现黑线

对应热更新键：

```json
"dplayer_feed_support_interaction": 0
```

README 里的说明是「播放进度条显示异常，设为 0 可解决」。

**注意**：这个键只有在「禁止下发配置」开启、补丁真正生效时才会应用
（`DYYYABTestBlockEnabled = 1`）。关掉补丁时服务器下发的是 `1`，黑线就会出现。

### 排查时踩的坑

- 手机上 `killall` 杀不掉抖音（PID 不变），看起来“重启了”其实没有，
  导致补丁一直没重新加载。要用 `kill -9 <pid>`。
- DYYY 的补丁只在 `DYYYABTestBlockEnabled = 1` 时应用，测试前后要确认这个值。

---

## 三、附：本次用到的探针方法

- 冷启动注入：`spawn` + `dev.attach(pid)` + `script.load()` + `dev.resume(pid)`
  （iSH 里用 `/opt/frida_client/spawn2.py`）
- 判断补丁是否生效：读 `AWEABTestManager.abTestData` 里的
  `enable_hdr_manual_settings`（只有补丁会注入这个键）
- 判断图标是否显示：遍历视图树找到 `AWENormalModeTabBarFeedView`，
  看子视图数量（2 = 无图标，3 = 有容器）和内部 `UIImageView.image` 是否为 nil
