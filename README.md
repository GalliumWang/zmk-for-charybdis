# Charybdis 4×6 ZMK 配置

本仓库用于构建采用两块 **nice!nano v2** 和 **PMW3610 轨迹球传感器**的
Charybdis 4×6 分体键盘固件。

## 硬件与主从关系

- 右半：ZMK 分体中央主控（central），连接电脑并处理键位、活动层和轨迹球事件。
- 左半：ZMK 分体从属端（peripheral），扫描左侧按键并通过 BLE 发送给右半。
- 轨迹球：位于右半，使用 PMW3610 驱动。
- ZMK Studio：编译在右半固件中，通过右半 USB 接口连接。

## 当前轨迹球层行为

| 层 | 轨迹球模式 | 说明 |
| --- | --- | --- |
| 0 | 正常光标 | 默认指针移动 |
| 1 | 正常光标 | 已取消原滚轮模式，与第 0 层一致 |
| 2 | 正常光标 | 不启用特殊处理，与第 0、1 层一致 |
| 3 | 狙击模式 | 使用较低 CPI，便于精确定位 |
| 4 | 滚轮模式 | 将轨迹球移动转换为水平/垂直滚动 |

当前 PMW3610 参数：

- 正常 CPI：`400`
- 狙击 CPI：`200`
- 滚轮步进阈值：`70`
- 软件轮询率：`125 Hz`
- 启用 Smart Algorithm
- 传感器旋转 90°，并反转 X 轴

## 变更日志

### 2026-07-24：轨迹球功能层重新分配

工作分支：`codex/trackball-layer-remap`

#### 功能修改

1. 将 PMW3610 驱动内置滚轮层从第 1 层移动到第 4 层：

   ```dts
   scroll-layers = <4>;
   ```

2. 正式启用第 3 层作为狙击层：

   ```dts
   snipe-layers = <3>;
   ```

3. 第 1 层不再触发滚轮模式，恢复为普通光标移动。
4. 第 2 层不启用狙击或滚轮处理，轨迹球行为与第 0、1 层一致。
5. 保留右半的 ZMK Studio 支持；正常刷写不会主动清除 Studio 中保存的键位。

#### 构建稳定性

- 将 `zmk-pmw3610-driver` 固定到提交
  `1d9c2c68ca76012e1b1e5f6ef02fa5eadc4ca399`，避免上游分支变化导致相同配置在不同时间产生不同结果。
- GitHub Actions 已成功构建并验证以下三个目标：
  - `charybdis_right-nice_nano_v2-zmk.uf2`
  - `charybdis_left-nice_nano_v2-zmk.uf2`
  - `settings_reset-nice_nano_v2-zmk.uf2`
- 验证使用的源码提交：`6c93d618e0d79f46b47d0d39222957ad043d52f2`
- 构建记录：[GitHub Actions run #30032483471](https://github.com/GalliumWang/zmk-for-charybdis/actions/runs/30032483471)

#### 刷写说明

本次轨迹球功能修改位于右半主控配置中，因此只刷写右半固件即可生效。
左右两边都刷写同一次构建产物也可以用于保持版本一致。

正常更新时不要刷写 `settings_reset`。该固件仅用于排查持久化设置或分体配对故障，
会清除蓝牙配对、左右半配对和 ZMK Studio 保存的运行时键位。

## TODO：改用 ZMK 通用滚轮输入处理器

当前滚轮功能继续使用 PMW3610 驱动内置的：

```dts
scroll-layers = <4>;
```

现有方案已经可用，暂不修改。后续计划将滚轮层从传感器驱动的专用实现迁移到
ZMK 通用输入处理器 `&zip_xy_to_scroll_mapper`，以降低键盘配置和特定驱动之间的耦合。

计划步骤：

- [ ] 确认当前固定的 ZMK `v0.3` 与 PMW3610 模块对
      `zmk,input-listener`、层级 override 和 `&zip_xy_to_scroll_mapper` 的兼容性。
- [ ] 在轨迹球输入监听器中为第 4 层增加 layer-specific override。
- [ ] 在第 4 层使用 `&zip_xy_to_scroll_mapper`，将 X/Y 相对移动映射为
      horizontal wheel/wheel 事件。
- [ ] 根据实际手感组合 `&zip_scroll_scaler`，分别测试水平与垂直滚动速度。
- [ ] 根据当前传感器方向验证滚动方向；必要时使用
      `&zip_scroll_transform` 调整反向或轴交换。
- [ ] 保证第 0、1、2 层仍为普通光标，第 3 层狙击模式不受影响。
- [ ] 分别验证 USB 与蓝牙输出，并验证通过 ZMK Studio 激活第 4 层时行为一致。
- [ ] 完成上述验证后再移除驱动内置的 `scroll-layers = <4>;`，避免两个滚轮转换路径同时生效。
- [ ] 保留当前驱动内置方案作为回滚点，并记录最终滚动倍率和方向参数。

参考资料：

- [ZMK Input Processor Overview](https://zmk.dev/docs/keymaps/input-processors)
- [ZMK Input Processor Usage](https://zmk.dev/docs/keymaps/input-processors/usage)
- [ZMK Code Mapper Input Processor](https://zmk.dev/docs/keymaps/input-processors/code-mapper)
