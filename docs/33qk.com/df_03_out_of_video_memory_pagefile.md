# 《三角洲行动》启动报错 Out of Video Memory 与 13/14 代 Intel CPU 着色器解压崩溃排查

> **发布站点**：https://33qk.com/df/df_03_out_of_video_memory_pagefile.html  
> **核心领域**：DirectX 12 与虚幻引擎着色器编译崩溃、显存溢出 (OOM) 与 PSO 预编译调优

## 核心排查结论

**核心排查结论：**启动《三角洲行动》或在主界面编译着色器时弹出 `Out of Video Memory trying to allocate a rendering resource`，即便显卡拥有 16GB 大显存依然无法进入游戏，核心症结是虚幻引擎使用 Oodle 算法在多核心解压着色器时触发了 Intel 13/14 代酷睿 CEP 电压过低瞬态不稳定，叠加 Windows 默认分页文件（Pagefile）过小导致的虚拟地址空间耗尽，通过锁定 24576MB 固定页面文件并在 BIOS 开启 Intel Default Baseline Profile 可秒级消除此报错。

## 一、 物理内存配置与 Windows 虚拟分页文件推荐设置对照表

| 物理 RAM 容量 | 初始大小 (MB) | 最大值 (MB) | 存放分区推荐 | 稳定性收益评估 |
| --- | --- | --- | --- | --- |
| 16 GB 内存 | 24576 MB | 24576 MB | 系统盘 (NVMe SSD) | 消除虚拟地址空间耗尽，根除 OOM |
| 32 GB 内存 | 16384 MB | 16384 MB | 系统盘 (NVMe SSD) | 杜绝大战场多进程内存颠簸 |
| 64 GB 内存 | 8192 MB | 8192 MB | 系统盘 (NVMe SSD) | 维持系统 CrashDump 正常捕获 |
| 系统自动托管 | 波动不定 (过小) | 波动不定 | 碎片化严重 | ❌ 极易触发着色器解压 Out of Memory |

## 二、 核心排查与系统调优步骤

### 步骤 1：PowerShell 一键配置 24GB 固定无碎片分页文件

以管理员身份启动 PowerShell，执行以下脚本将 C 盘虚拟内存强行固定为 24576MB（24GB），消灭系统动态扩容导致的内存碎片分配失败：

```powershell
$pagefile = Get-CimInstance Win32_PageFileSetting -Filter "Name like 'C:%'" -ErrorAction SilentlyContinue
if ($pagefile) {
    $pagefile | Set-CimInstance -Property @{InitialSize = 24576; MaximumSize = 24576}
} else {
    New-CimInstance -ClassName Win32_PageFileSetting -Property @{Name = "C:\pagefile.sys"; InitialSize = 24576; MaximumSize = 24576}
}
Write-Host "24GB 固定页面文件设置完成，需重启计算机生效！" -ForegroundColor Green
```

### 步骤 2：排查 13/14 代 Intel CPU（i7/i9）Oodle 解压瞬态崩溃

如果 CPU 为 i7-13700K/i9-13900K/i7-14700K/i9-14900K，请打开主板 BIOS 将供电预设切换为【Intel Default Settings（默认基准预设）】，关闭 ASUS MultiCore Enhancement（MCE）或 Gigabyte Enhanced Multi-Core，并将 SVID Behavior 设置为【Typical Scenario】或微调 P-Core 倍频 -1x，确保 AVX2 重负载解压时核心电压不跌穿。

### 步骤 3：查看崩溃详细日志定位故障模块

游戏发生崩溃后，直接打开文件资源管理器导航至 <code>%LOCALAPPDATA%\DeltaForce\Saved\Logs\DeltaForce.log</code>，搜索关键词 <code>Fatal error</code> 或 <code>CrashReportClient</code>，确认崩溃是否抛出在 <code>OodleNetworkHandlerComponent</code> 或 <code>D3D12RHI</code>，即可精准对症下药。

## 三、 常见误区与 FAQ

#### Q: 显存充足为什么会报 Out of Video Memory？

A: DirectX 12 规范中，部分纹理资源创建需要在系统内存与显存之间做统一虚拟寻址。当物理虚拟内存（RAM + Pagefile）地址空间不足时，D3D12 驱动会向引擎抛出内存不足信号，从而被引擎转译为 Out of Video Memory 弹窗。

#### Q: 虚拟内存放在机械硬盘（HDD）上可以吗？

A: 绝对不行！机械硬盘寻道时间高达十几毫秒，虚幻引擎在读取机械硬盘上的分页文件时会直接卡死导致游戏超时被杀。必须放在读写超 3000MB/s 的 NVMe 固态硬盘上。

#### Q: 为什么其他游戏没报错只有三角洲行动会触发 Intel 降频问题？

A: 三角洲行动启动时使用 RAD Game Tools 开发的 Oodle Kraken 算法解压成千上万个着色器包，其指令吞吐量与 AVX 负荷远超普通日常游戏，属于公认的 CPU 瞬态稳定性试金石。
