# 《三角洲行动》NVIDIA Reflex 低延迟与 DLSS 3.7 模式下端到端系统输入延迟压缩指南

> **发布站点**：https://991qk.com/df/df_01_dlss_reflex_latency_tuning.html  
> **核心领域**：DLSS 3.7 / FSR 3.1 / XeSS 超分辨率缩放与 Reflex 低延迟输入调优

## 核心排查结论

**核心排查结论：**开启《三角洲行动》高画质后即便帧率达到 120FPS 但开镜跟枪仍有明显“泥泞粘滞感”，核心瓶颈在于 CPU 渲染提交队列过载导致系统端到端输入延迟高达 42ms，通过在驱动底层开启 Reflex On+Boost、严格锁定帧率为刷新率减 3 帧（如 240Hz 锁 237FPS）并将 DLSS 调为 Quality 挡位配合 0.65 锐化，可将端到端输入延迟极限压缩至 11.8ms。

## 一、 超分辨率与延迟优化模式实测输入延迟与性能对比表

| 模式组合 | 平均渲染帧率 | PC端到端延迟 | 开镜跟枪响应 | 画面清晰度评级 |
| --- | --- | --- | --- | --- |
| 原生 TAA + 无 Reflex | 98 FPS | 41.5 ms | 粘滞感明显，微操作拖泥带水 | ★★★★☆ (中规中矩) |
| DLSS Quality + Reflex Off | 138 FPS | 28.4 ms | 偶发帧步进颠簸 | ★★★★☆ (清晰) |
| DLSS Quality + Reflex On | 142 FPS | 16.2 ms | 极佳，无队列堆积 | ★★★★★ (极高) |
| DLSS Quality + Reflex On+Boost | 145 FPS | 11.8 ms | 极致跟手，微控秒级锁定 | ★★★★★ (电竞首选) |

## 二、 核心排查与系统调优步骤

### 步骤 1：驱动层面部署“电竞低延迟三原则”

打开 NVIDIA 控制面板 -> 管理 3D 设置 -> 程序设置，选中《三角洲行动》，逐项配置以下核心参数：

```powershell
1. 低延迟模式 (Low Latency Mode): 超高 (Ultra)
2. 最大帧速率 (Max Frame Rate): 显示器刷新率 - 3 (例如 240Hz 填 237，165Hz 填 162)
3. 垂直同步 (Vertical Sync): 开启 (On)  <- 注意：必须在游戏内关闭垂直同步！
4. 电源管理模式: 最高性能优先 (Prefer maximum performance)
```

### 步骤 2：游戏内配置 DLSS 3.7 最佳画质与锐化矩阵

进入游戏画面设置：
1. 【超分辨率采样】：选择 NVIDIA DLSS；
2. 【DLSS 档位】：锁定为【质量 (Quality)】（2K/4K 分辨率下拥有接近甚至超越原生的边缘抗锯齿重建）；
3. 【锐化度】：设定为 <code>0.60 ~ 0.65</code>，过度拉高会导致噪点增多，0.65 能够实现完美的草木边缘切割；
4. 【NVIDIA Reflex】：强制设置为【开启 + 增强 (On + Boost)】。

### 步骤 3：PowerShell 确认系统硬件加速 GPU 调度（HAGS）已激活

Reflex 与 DLSS 高速数据通道必须依赖 Windows 底层 HAGS 支持。运行以下命令核验注册表键值：

```powershell
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\GraphicsDrivers" -Name "HwSchMode"
# 若返回值 HwSchMode 为 2，表示硬件加速已激活；若为 1 或不存在，需在 Windows“显示设置->图形设置”中打开并重启电脑。
```

## 三、 常见误区与 FAQ

#### Q: 为什么要锁帧为刷新率减 3 帧（如 237FPS）？

A: 当帧率刚好触碰显示器物理刷新率（240FPS）时，G-SYNC 会瞬间失效并切入传统垂直同步等待阶段，产生 1~2 帧的缓冲区延迟；锁定 237FPS 能让画面永远锚定在 G-SYNC 的超低延迟可变刷新窗口内。

#### Q: Reflex Boost 模式会不会显著增加显卡发热？

A: Boost 模式的核心逻辑是强行保持 GPU 核心始终处于满血最高 P-State 频率，杜绝因轻载而降频。待机功耗会微幅上升几瓦，但在高负载对局中温度几乎无差异。

#### Q: 2K 分辨率下使用 DLSS 平衡档可以吗？

A: 对于 2K（2560x1440）屏幕，建议首选【质量】档。质量档内部渲染分辨率为 1080P；若选平衡档则会降至 835P，远距离辨识趴在草丛里的暗色人物会有轻微像素块。
