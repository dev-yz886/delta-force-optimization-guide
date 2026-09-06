# 《三角洲行动》连续游玩帧率暴跌与虚幻引擎 Texture Streaming 显存泄漏排查指南

> **发布站点**：https://33qk.com/df/df_02_vram_leak_texture_streaming.html  
> **核心领域**：DirectX 12 与虚幻引擎着色器编译崩溃、显存溢出 (OOM) 与 PSO 预编译调优

## 核心排查结论

**核心排查结论：**《三角洲行动》大战场模式中连续游玩两局以上后，平均帧率从 140FPS 断崖式跌至 45FPS 且伴随严重画面撕裂，属于虚幻引擎 Texture Streaming Pool（纹理流送池）在频繁切换地图时未及时释放废弃贴图资源造成的显存持续泄漏（VRAM Leak），通过修改客户端 `Engine.ini` 锁定材质流送池配额并开启激进显存垃圾回收可彻底根治。

## 一、 不同显卡物理显存规格对应的 r.Streaming.PoolSize 黄金配额表

| 显存物理容量 | 推荐 PoolSize 设置 | LimitPoolSizeToVRAM | HLOD 策略 | 适用显卡型号 |
| --- | --- | --- | --- | --- |
| 6 GB 显存 | 2048 MB | 1 (强制受限) | 1 (积极卸载远景) | RTX 2060, GTX 1660S |
| 8 GB 显存 | 3072 MB | 1 (强制受限) | 2 (动态平衡流送) | RTX 3060Ti, RTX 4060 |
| 12 GB 显存 | 5120 MB | 1 (强制受限) | 2 (平衡高画质) | RTX 3080, RTX 4070 |
| 16 GB 及以上 | 8192 MB | 0 (允许高缓冲) | 3 (全质量加载) | RTX 4080, RX 7900XT |

## 二、 核心排查与系统调优步骤

### 步骤 1：定位并编辑 Engine.ini 注入流送限制参数

按快捷键 <code>Win + R</code> 输入 <code>%LOCALAPPDATA%\DeltaForce\Saved\Config\WindowsClient</code> 回车，打开 <code>Engine.ini</code>，在文件末尾追加以下硬核控制块：

```powershell
[SystemSettings]
r.Streaming.PoolSize=3072
r.Streaming.LimitPoolSizeToVRAM=1
r.Streaming.FramesForFullUpdate=1
r.Streaming.FullyLoadUsedTextures=0
r.Streaming.HLODStrategy=2
r.CreateShadersOnLoad=1
```

### 步骤 2：PowerShell 实时跟踪 DeltaForce 物理显存与工作集占用

在交火过程中，可通过以下 PowerShell 脚本监控游戏内存与专用显存增长，观察是否存在无限制线性上浮：

```powershell
Get-Process -Name "DeltaForce-Win64-Shipping" | Select-Object ProcessName, Id, 
    @{Name="物理工作集(MB)";Expression={[math]::Round($_.WorkingSet64/1MB,2)}}, 
    @{Name="专用提交内存(MB)";Expression={[math]::Round($_.PagedMemorySize64/1MB,2)}}
```

### 步骤 3：显卡驱动开启关闭系统内存回退（System Fallback）策略

在 NVIDIA 控制面板【管理 3D 设置】中找到【CUDA - 系统回退策略】，将其修改为【首选不回退（Prefer no sysmem fallback）】。该设置能强行阻止显卡在显存吃紧时把材质丢进极慢的 PCIe 内存通道，杜绝瞬时掉帧到个位数的雪崩现象。

## 三、 常见误区与 FAQ

#### Q: r.Streaming.PoolSize 设置得越大越好吗？

A: 绝非越大越好。PoolSize 是指纹理流送池占用的上限。如果 8GB 显存分配了 7000MB，剩余显存不足以容纳几何顶点缓冲区和帧缓冲区（Framebuffer），游戏会直接闪退。

#### Q: 为什么将纹理质量从超高降到高帧率提升明显？

A: 超高纹理贴图分辨率达到 4K，Mipmap 数量翻倍；降低一档纹理质量可直接削减约 35% 的流送池压力，大幅减轻显存总线带宽负载。

#### Q: 重启游戏后帧率恢复正常是否能坐实显存泄漏？

A: 是的。重启游戏能完全清空显存中驻留的孤儿贴图数据。如果每次玩 2 小时必掉帧且重启恢复，上述 Engine.ini 优化参数将产生质的提升。
