# HDMI-ISP-UDP-FPGA

基于紫光同创 Logos2 FPGA 的 HDMI 视频采集、ISP 图像处理、DDR3 帧缓存、以太网 UDP 推流系统。

## 系统架构

```
HDMI 输入 (RGB888)
  -> Gamma 校正 (LUT x3)
  -> 高斯滤波 (3x3 矩阵)
  -> 色彩空间转换 CSC (RGB -> YUV)
  -> 边缘增强 / 锐化
  -> 色彩空间转换 OCC (YUV -> RGB)
  -> 3 倍下采样 (1920x1080 -> 640x360)
  -> RGB565 打包
  -> DDR3 帧缓存 (写入)
  -> DDR3 帧读取
  -> UDP/IP/MAC/RGMII (千兆以太网发送)
```

## 目标器件

- **FPGA**: 紫光同创 Logos2 PG2L100H-6 (FBG676)
- **工具链**: Pango Design Suite 2022.2-SP6.4
- **顶层模块**: `hdmi_ddr_eth_top` (`source/rtl/hdmi_ddr_eth_top.v`)

## 关键参数

| 参数 | 值 |
|---|---|
| 图像分辨率 | 640 x 360 |
| 像素格式 | RGB565 |
| DDR3 数据位宽 | 32-bit |
| 本地 IP | 192.168.1.11 |
| 本地端口 | 8080 |
| 目标 IP | 192.168.1.105 |
| 目标端口 | 8080 |

## 目录结构

```
source/
  rtl/            # 顶层模块、DDR 帧写入/读取控制器、帧缓冲
  eth/            # UDP/IP/MAC/ARP 完整协议栈、RGMII 接口
  hdmi/           # MS7200/MS7210 HDMI 收发芯片 I2C 配置
  img_process/    # Gamma 查找表
  tools/          # PC 端 UDP 接收脚本 (Python)
  *.v             # ISP 模块 (高斯、CSC、EE、OCC、下采样、LCD 驱动)
ipcore/           # IP 核 (DDR3 控制器、PLL、FIFO)
sim_prj/          # 仿真工程
matlab/           # MATLAB 辅助脚本
*.pds             # Pango Design Suite 工程文件
*.fdc             # 引脚与约束文件
```

## ISP 处理链

所有 ISP 模块运行在 `pixclk_in` 时钟域（1080p 输入时约 148.5 MHz）：

1. **Gamma** - 逐通道 Gamma 校正（查找表实现）
2. **高斯滤波** - 3x3 空间低通滤波（matrix_3x3 + 行缓冲 FIFO 实现）
3. **CSC** - 色彩空间转换 RGB -> YUV
4. **EE** - YUV 域边缘增强 / 锐化
5. **OCC** - 色彩空间转换 YUV -> RGB
6. **下采样** - 3 倍双线性下采样（1920x1080 -> 640x360）

## 时钟域

| 时钟 | 频率 | 用途 |
|---|---|---|
| sys_clk | 27 MHz | 系统配置、I2C |
| pixclk_in | ~148.5 MHz | HDMI 像素时钟、ISP 全链路 |
| ddrphy_sysclk | 200 MHz | DDR3 控制器 |
| rgmii_clk_90p | 125 MHz | 千兆以太网 RGMII |

## UDP 数据包格式

每个 UDP 包承载一行图像数据（1286 字节）：

```
Byte [0..3]  : frame_id，大端序
Byte [4..5]  : line_id (0..359)
Byte [6..1285]: 640 像素 x RGB565，高字节在前
```

## PC 端接收

```bash
python source/tools/udp_frame_dump.py --bind-ip 0.0.0.0 --port 8080 --width 640 --height 360 --output-dir ./captures
```

接收 UDP 数据包，按 frame_id + line_id 重组整帧，RGB565 转 RGB888，保存为 .ppm 图像。

## 时序状态

所有约束均已满足（All Constraints Met），关键时钟域 WNS 余量：

- pixclk_in: +2.1 ns
- ddrphy_sysclk: +3.3 ns
- rgmii_clk_90p: +3.1 ns

## 文档

- [上板联调指南](source/HDMI_DDR_ETH_bringup.md) - 板级调试与接线说明
- [整合流程说明](source/HDMI_DDR_ETH_merge_flow.md) - 工程整合过程记录
