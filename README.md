# HDMI-ISP-UDP-FPGA + RK 车牌识别系统

FPGA 端 HDMI 视频采集、ISP 图像处理、DDR3 帧缓存、千兆以太网 UDP 推流，配合 RK3588 端 NPU 车牌检测与识别的完整链路。

## 系统总览

```
┌─────────────────────── FPGA 端 (Logos2 PG2L100H) ───────────────────────┐
│                                                                          │
│  HDMI 输入 (RGB888, 1920x1080)                                          │
│    → Gamma 校正 (LUT ×3)                                                 │
│    → 高斯滤波 (3×3 matrix + 行缓冲 FIFO)                                │
│    → CSC (RGB → YUV)                                                     │
│    → 边缘增强 / 锐化 (EE)                                                │
│    → OCC (YUV → RGB)                                                     │
│    → 3× 下采样 (1920×1080 → 640×360)                                    │
│    → RGB565 打包                                                         │
│    → DDR3 帧缓存 (写入/读出)                                             │
│    → UDP/IP/MAC/ARP (千兆以太网 RGMII)                                   │
│                                                                          │
└──────────────────────────── UDP (1286 B/包) ─────────────────────────────┘
                              │
                              ▼
┌─────────────────────── RK 端 (RK3588 NPU) ──────────────────────────────┐
│                                                                          │
│  UDP 接收 + 帧重组 + 补线                                                │
│    → RGB565 → RGB 预处理                                                 │
│    → FastestDet 车牌检测 (352×352, RKNN NPU)                             │
│    → NMS 后处理 + ROI 裁剪                                               │
│    → 双层车牌拼接处理                                                     │
│    → CRNN 车牌识别 (168×48, RKNN NPU)                                    │
│    → CTC 解码                                                            │
│    → X11 实时显示 (检测框 + 识别结果)                                    │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

## 目标器件

| 组件 | 型号 |
|---|---|
| FPGA | 紫光同创 Logos2 PG2L100H-6 (FBG676) |
| 工具链 | Pango Design Suite 2022.2-SP6.4 |
| NPU 平台 | Rockchip RK3588 (RKNN SDK 2.x) |
| 顶层模块 | `hdmi_ddr_eth_top` (`source/rtl/hdmi_ddr_eth_top.v`) |

## FPGA 端

### ISP 处理链

所有 ISP 模块运行在 `pixclk_in` 时钟域（1080p 输入时约 148.5 MHz）：

1. **Gamma** — 逐通道 Gamma 校正（查找表实现，`source/img_process/gamma_table.v`）
2. **高斯滤波** — 3×3 空间低通滤波（`matrix_3x3` + `fifo_line_buf` 行缓冲）
3. **CSC** — 色彩空间转换 RGB → YUV（`source/isp_csc.v`）
4. **EE** — YUV 域边缘增强 / 锐化（`source/isp_ee.v`）
5. **OCC** — 色彩空间转换 YUV → RGB（`source/isp_occ.v`）
6. **下采样** — 3 倍双线性下采样 1920×1080 → 640×360（`source/scale3x_downsampler.v`）

### 时钟域

| 时钟 | 频率 | 用途 |
|---|---|---|
| sys_clk | 27 MHz | 系统配置、I2C |
| pixclk_in | ~148.5 MHz | HDMI 像素时钟、ISP 全链路 |
| ddrphy_sysclk | 200 MHz | DDR3 控制器 |
| rgmii_clk_90p | 125 MHz | 千兆以太网 RGMII |

### 时序状态

所有约束均已满足（All Constraints Met），关键时钟域 WNS 余量：

| 时钟域 | WNS |
|---|---|
| pixclk_in | +2.1 ns |
| ddrphy_sysclk | +3.3 ns |
| rgmii_clk_90p | +3.1 ns |

### 关键参数

| 参数 | 值 |
|---|---|
| 图像分辨率 | 640 × 360 |
| 像素格式 | RGB565 |
| DDR3 数据位宽 | 32-bit |
| 本地 IP | 192.168.1.11 |
| 本地端口 | 8080 |
| 目标 IP | 192.168.1.105 |
| 目标端口 | 8080 |

## RK 端

### 多线程架构

```
read_thread:     UDP 接收 + 帧重组 + RGB565→RGB 预处理
detect_thread:   FastestDet NPU 推理 (车牌检测)
rec_thread:      检测后处理 + 裁剪 + CRNN 推理 + CTC 解码
main_thread:     X11 显示 + 事件处理
```

线程间通过 `LatestQueue`（最新帧优先队列）连接，旧帧自动丢弃。

### 模型

| 模型 | 文件 | 输入尺寸 | 用途 |
|---|---|---|---|
| FastestDet | `FastestDet.rknn` | 352×352 RGB | 车牌检测（2 类：单层/双层） |
| CRNN | `best.rknn` | 168×48 BGR | 车牌文字识别（80 类字符） |

### 支持的字符

- 中国省份简称：京沪津渝冀晋蒙辽吉黑苏浙皖闽赣鲁豫鄂湘粤桂琼川贵云藏陕甘青宁新
- 特殊车牌：学警港澳挂使领民航危险品
- 数字：0-9，字母：A-Z（不含 I、O）

### 依赖

- RKNN SDK 2.x（需设置 `RKNN_API_ROOT`）
- OpenCV 4.x
- X11 + XShm
- C++17 编译器

### 编译与运行

```bash
cd rk_receiver
mkdir build && cd build
cmake .. -DRKNN_API_ROOT=/path/to/rknn-sdk
make -j$(nproc)
./main_v5_cpp -p 8080 -S 2
```

| 参数 | 说明 | 默认值 |
|---|---|---|
| `-b, --bind-ip IP` | 绑定 IP | 0.0.0.0 |
| `-p, --port PORT` | UDP 端口 | 8080 |
| `-s, --source-ip IP` | 仅接受指定源 IP 的包 | 无 |
| `-B, --rcvbuf-mb N` | Socket 接收缓冲区 (MB) | 32 |
| `-S, --scale N` | X11 窗口整数放大倍数 | 1 |

## UDP 数据包格式

每个 UDP 包承载一行图像数据（1286 字节）：

```
Byte [0..3]   : frame_id，大端序
Byte [4..5]   : line_id (0..359)
Byte [6..1285] : 640 像素 × RGB565，高字节在前
```

## 网络对接

FPGA 默认目标 IP 为 `192.168.1.105`，端口 `8080`。确保：

1. RK 板网口 IP 设为 `192.168.1.105`
2. 运行时加 `-s 192.168.1.11` 可只接受 FPGA 来源的包

## PC 端接收

```bash
python source/tools/udp_frame_dump.py --bind-ip 0.0.0.0 --port 8080 --width 640 --height 360 --output-dir ./captures
```

接收 UDP 数据包，按 frame_id + line_id 重组整帧，RGB565 转 RGB888，保存为 .ppm 图像。

## 目录结构

```
source/                  # FPGA 端 RTL 源码
  rtl/                   # 顶层模块、DDR 帧写入/读取控制器、帧缓冲
  eth/                   # UDP/IP/MAC/ARP 完整协议栈、RGMII 接口
  hdmi/                  # MS7200/MS7210 HDMI 收发芯片 I2C 配置
  img_process/           # Gamma 查找表
  tools/                 # PC 端 UDP 接收脚本 (Python)
  *.v                    # ISP 模块 (高斯、CSC、EE、OCC、下采样、LCD 驱动)
ipcore/                  # IP 核 (DDR3 控制器、PLL、FIFO)
sim_prj/                 # 仿真工程
matlab/                  # MATLAB 辅助脚本 (Gamma 曲线生成)
rk_receiver/             # RK 端车牌识别接收程序 (C++17 / RKNN)
*.pds                    # Pango Design Suite 工程文件
*.fdc                    # 引脚与约束文件
```

## 文档

- [上板联调指南](source/HDMI_DDR_ETH_bringup.md) — 板级调试与接线说明
- [整合流程说明](source/HDMI_DDR_ETH_merge_flow.md) — 工程整合过程记录
- [RK 端接收器说明](rk_receiver/README.md) — RK 端编译、运行与模型详情
