# CI1303 声音能量上报 — 实现、实测与坑

本文档记录 CI1303（CI130X SDK）侧「声音能量值」这条链路的真实实现、实测数据与踩过的坑。
能量值经串口协议 `fill_data` 最高字节上报给 WiFi 端，用于**声音能量律动**（点阵 / 灯条）。

> 整理日期：2026-09-17
> 工程路径：`D:\src\other\CI13XX_SDK_LLM_AIOT_2.1.2`
> Active sample：`projects/offline_asr_llm_aiot_uart_sample`
> 对端：`D:\src\jieli\ac791n_Voice_robot`（Jieli AC791N WiFi 端，消费方）

---

## 0. TL;DR

三条结论，按「踩坑概率」排序：

1. **录音缓存是双通道交织 `[ch0, ch1]`，不是单声道。** 就算 `HOST_MIC_USE_NUMBER = 1`，
   只要开了 `USE_AEC_MODULE`，`HOST_CODEC_CHA_NUM` 就被强制成 2。**任何按 `data_size/2`
   遍历的统计都会把两路混在一起。**
2. **片上 ALC 默认开启，会实时改 PGA 增益。** 实测回采通道在播放音乐时增益被从 +27 dB 压到 0 dB。
   从 ADC 数据算出来的任何绝对值都必须按当前 PGA 增益补偿，否则动态范围会被压掉一大半。
3. **通道语义**：`ch0` = MIC（差分），`ch1` = 喇叭回踩（单端，兼作 AEC 参考）。
   做「喇叭声音律动」要用 **ch1**，不是 ch0。

---

## 1. 数据链路

```
CI1302 片内 Codec ADC（差分 ch0 = MIC / 单端 ch1 = 回踩）
    ↓  cm_read_codec() 拿到一块交织 PCM
audio_in_manage_inner.c  按通道取均方 → ALC 补偿
    ↓  is_open_voice_energy() 注册的回调 ven_call_function()
asr_process_callback_decoder.c
    ↓  写 g_denergy_buffer[12]（环形）
energy_sort_print_task  每 100ms：排序取最大 4 帧求平均 → 标定 → g_mic_db
    ↓
cias_network_msg_send_task.c  energy_fill_data()：能量放 fill_data 最高字节
    ↓  心跳包（100ms）+ 音频包（每个 PCM_MIDDLE）
串口 ──▶ WiFi 端 ci1303_update_mic_energy() → ci1303_get_mic_energy()
```

关键点：能量取自 **`cm_read_codec` 出来的原始 ADC 数据**，位置在 `send_voice_info_to_audio_in_manage()`
**之前**——即算法侧（VAD/AEC/ASR）拿到的数据一个字都没被改动，能量统计是旁路。

---

## 2. 坑一：录音缓存是双通道交织

### 现象

`main.c` 的编译命令行里实际是：

```
-DHOST_MIC_USE_NUMBER=1 -DHOST_CODEC_CHA_NUM=2 -DUSE_AEC_MODULE=1 -DUSE_PWK=0
```

注意 `HOST_MIC_USE_NUMBER = 1`（单 mic），但 **`HOST_CODEC_CHA_NUM = 2`**。

### 原因链

| 位置 | 内容 |
|---|---|
| `app/app_main/user_config.h:456` | `#if USE_AEC_MODULE` → `#define HOST_CODEC_CHA_NUM 2` |
| `app/app_main/user_config.h:428-431` | `USE_DUAL_MIC_ANY` 只在 `HOST_MIC_USE_NUMBER == 2` 时才定义 |
| `driver/boards/CI-D02GS01J.c:480-484` | `#if (2 == HOST_CODEC_CHA_NUM)` → `.channel_flag = 3`（左右都录） |
| `driver/boards/CI-D02GS01J.c:505-512` | `block_size = 256 * 2 * resample_double * (cha==3 ? 2 : 1)` |

即：**开了 AEC 就必须录两路（mic + 参考），与 mic 数量无关。**

### 通道语义（`CI-D02GS01J.c:421-430`）

```c
#if (MIC_DIFF_SINGLE == 0)
    .codec_gain.codec_adc_input_mode_l = INNER_CODEC_INPUT_MODE_DIFF,          // ch0 = MIC 差分
#else
    .codec_gain.codec_adc_input_mode_l = INNER_CODEC_INPUT_MODE_SINGGLE_ENDED,
#endif
#if USE_AEC_MODULE
    .codec_gain.codec_adc_input_mode_r = INNER_CODEC_INPUT_MODE_SINGGLE_ENDED, // ch1 = 回踩 单端
#else
    .codec_gain.codec_adc_input_mode_r = INNER_CODEC_INPUT_MODE_DIFF,
#endif
```

本工程是 **单 mic 硬件 + 喇叭回踩电路**，所以：

| 通道 | 输入方式 | 实际信号 |
|---|---|---|
| **ch0** | 差分 | MIC（人说话 / 环境声） |
| **ch1** | 单端 | **喇叭回踩**（喇叭声经回踩分压电路进芯片），AEC 拿它当参考 |

### 原始实现的 bug

`components/audio_in_manage/audio_in_manage_inner.c` 原代码：

```c
int16_t *voice_data = (int16_t *)data_addr;
float voice_energy = 0.0f;
for (int i = 0; i < data_size / 2; i++)      // ← 整个缓存挨个平方，不分通道
{
    voice_energy += (float)(voice_data[i] * voice_data[i]);
}
..._callback(voice_energy / ((float)(data_size / 2)));
```

**两路样本被一起平方求和**，回采通道的贡献混进了 MIC 的读数里。

### 修复

新增 `energy_channel_mean_square()`（`audio_in_manage_inner.c:195`），按交织步长只取一路：

```c
frames = data_size / (sizeof(int16_t) * cha);
for (uint32_t i = 0; i < frames; i++)
{
    float s = (float)data[i * cha + cha_idx];   // cha_idx 指定取哪一路
    sum += s * s;
}
return sum / (float)frames;
```

通道选择由宏控制（`audio_in_manage_inner.c:165-175`）：

```c
#define ENERGY_CHA_MIC      0
#define ENERGY_CHA_REF      1
#define ENERGY_REPORT_CHA   ENERGY_CHA_REF   // 做喇叭律动用回踩通道
```

---

## 3. 坑二：片上 ALC 会实时改增益

### 事实

CI1302 片内 Codec 的模拟前端自带 ALC（自动电平控制 / AGC），**默认开启，且本工程从头到尾没人关过它**：

| 参数 | 值 | 代码位置 |
|---|---|---|
| 开关 | **ENABLE** | `ci130x_codec.c:517` `ALC_AGC_FUNC_SEL` → reg49 bit6 |
| 目标电平 | −6 ~ −7.5 dBFS | `ci130x_codec.c:513-516` |
| 调节范围 | **−18 ~ +28.5 dB**（共 46.5 dB） | `ci130x_codec.c:518-519` |
| 起音 / 释放 / 保持 | 均 **2 ms** | `ci130x_codec.c:503-505` |
| 噪声门 | **关闭**（静音时照样往上抬） | `ci130x_codec.c:509` |
| 是否控制 PGA | **是** | `ci130x_codec.c:575-576` `ALC_CONTROL_PGA_L/R` |

App 侧本来有三个地方可以关掉它，**但都没编译进来**：

- `app/app_main/main.c:366-369` 的 `cm_set_codec_alc(..., DISABLE)` 在 `#if USE_DUAL_MIC_ANY` 里，
  而该宏只在 `HOST_MIC_USE_NUMBER == 2` 时定义 → **未编译**（编译命令行无 `-DUSE_DUAL_MIC_ANY`）
- `app/app_main/main.c:407-408` 在 `#if USE_PWK` 里，本工程 `-DUSE_PWK=0` → **未编译**
- `components/audio_in_manage/audio_in_manage_inner.c:530-533` → **被注释掉**

### 实测证据

回采通道（ch1）在两种状态下的 PGA 增益：

| 状态 | `pgaR_reg` | 增益 |
|---|---|---|
| 静音 | 30 | **+27.0 dB** |
| 播放音乐 | 12 | **0 dB** |

`pga_gain_reg[]` 在软件侧只有两个写入点（`ci130x_codec.c:233/239` 初始化、`ci130x_codec.c:746` ioctl），
而本工程没有任何周期性调用第二处（那几处 `cm_set_codec_adc_gain` 同样没编译进来）。
**所以这个 27 dB 的变化是硬件 ALC 自己写的寄存器。**

对照：MIC 通道（ch0）的 `pgaL_x10` 全程恒为 **+19.5 dB 不动**——环境变化对它影响小，ALC 没什么好调的。

### 后果

不补偿的话，真实电平变化了约 37 dB，ALC 只让读数变化约 10 dB —— **动态范围被压掉一大半**。
这正是最初「静音也有 50 dB」「放音乐只在 73~76 之间浮动」的根因。

---

## 4. ALC 增益补偿

### 驱动侧 getter

`driver/ci130x_chip_driver/src/ci130x_codec.c:492` 新增（声明在 `ci130x_codec.h:604`）：

```c
float inner_codec_get_mic_pga_gain_db(inner_codec_cha_sel_t cha)
{
    uint32_t reg_data = CODEC->pga_gain_reg[cha] & 0x1f;
    return (float)reg_data * 1.5f - 18.0f;
}
```

它是 `get_pga_gain_reg_data_from_db()`（`ci130x_codec.c:34-48`，`reg = (gain_db + 18) / 1.5`）
的逆运算，范围 −18 ~ +28.5 dB。

### 补偿公式

`audio_in_manage_inner.c:238`：

```c
static float energy_alc_uncompensate(float measured_ms, int cha_idx)
{
    inner_codec_cha_sel_t cha = (cha_idx == ENERGY_CHA_REF) ? INNER_CODEC_RIGHT_CHA : INNER_CODEC_LEFT_CHA;
    float gain_db = inner_codec_get_mic_pga_gain_db(cha);
    // 10^(-gain_db/10) = exp(-gain_db * ln10 / 10)
    return measured_ms * MASK_ROM_LIB_FUNC->newlibcfunc.expf_p(-gain_db * 0.23025851f);
}
```

`10^(gain_db/10)` 是该通道 PGA 的**功率**倍数，除掉它就抵消了 ALC 的调节。

### 验证（实测数据，精度 0.1 刻度单位）

刻度换算：`1 刻度单位 = 0.7933 dB`（见 §6）

```
静音:  src 22.3 刻度 → ms = 4440 → 真实 = 4440 / 10^2.7 = 8.86 → -11.7 刻度
       实测 rpt = -11.6 刻度                                ✓ 误差 0.1
音乐:  src 43.6 刻度 → ms = 217400 → gain 0dB，不变 → 43.6 刻度
       实测 rpt = 43.6 刻度                                 ✓ 精确
```

**真实动态范围：静音 8.86 → 音乐 217400，即 43.9 dB。**
补偿前可见范围仅 21.3 刻度单位（223 → 436），补偿后 55.3 单位（−11.7 → 43.6），**放大 2.6 倍**。

### 开关

```c
#define ENERGY_ALC_COMPENSATE   1   // audio_in_manage_inner.c:183，置 0 退回原始值
```

### ⚠️ 不要用「关掉 ALC」来绕过这个问题

ch1 同时是 **AEC 的参考信号**。关掉 ALC 会让参考电平突变 27 dB，可能破坏回声消除。
**软件补偿是非侵入的**：算法侧（`voice_info_str`）拿到的数据一个字没变。

---

## 5. 统计窗口与上报

### 窗口

`components/asr/asr_process_callback_decoder.c:83-84`：

```c
#define ENERGY_BUFFER_CNT (12)     // 12 帧 × 16ms = 192ms
#define ENERAY_EFFECTIVE_CNT (4)   // 取最大的 4 帧求平均
```

**原为 120/40（1.92s 峰值保持）**，会把起伏抹平。改为 12/4 后快速跟随瞬时电平，
窗口仍大于 100ms 上报周期，保证每次上报都有新数据。取最大 1/3 帧的比例保持不变（抑制窗口内静音段）。

顺带：冒泡排序比较次数从 14400 降到 132。

### 统计线程

`asr_process_callback_decoder.c:90` `energy_sort_print_task`，每 100 ms 执行一次
`sortDescending()` → 写入全局 `g_mic_db`。

线程在 `ven_call_function()` 收到第一帧能量时惰性创建（`asr_process_callback_decoder.c:110-115`），
此时调度器已运行，`xTaskCreate` 合法。

`sortDescending()` 用**独立工作缓冲**先 `memcpy` 再排序，不破坏 `g_denergy_buffer` 的时间顺序；
并用 `s_sort_mutex` 保护（该函数也被 `asr_result_callback` 使用）。

### 上报协议

`app/app_audio_handle/cias_network_msg_send_task.c:115`：

```c
static uint32_t energy_fill_data(void)
{
    extern uint16_t g_mic_db;
    return (((uint32_t)g_mic_db & 0xFF) << 24) | (DEF_FILL & 0x00FFFFFF);
}
```

| 项 | 值 |
|---|---|
| 载体字段 | 协议帧头 `fill_data`（`cias_standard_head_t` 偏移 12，4 字节） |
| 位置 | **最高字节** `fill_data >> 24` |
| 低 24 位 | `0x345678`（原 `DEF_FILL = 0x12345678` 的低 24 位，保留作特征校验） |
| 携带的包 | `CIAS_CMD_HEARTBEAT (0x4003)` 心跳 + `PCM_MIDDLE (0x0105)` 音频包 |
| 心跳周期 | `NETWORK_UART_HEARTBEAT_INTERVAL_MS = 100`（`user_config.h:140`，**原 1000**） |

WiFi 端解析（`apps/common/asr/Ci1303/ci1303_uart.c`）：按 `type` 过滤（只认心跳与 PCM_MIDDLE），
并按低 24 位 == `0x345678` 做特征校验，排除解析误同步产生的垃圾帧
（该文件的 `ciasHeadCheck` magic 校验是注释掉的）。

### 顺带修的两个 UB

`sortDescending()` 里 `(uint16_t)mic_db` 有两处浮点转无符号的未定义行为：

1. **开机**：`g_denergy_buffer` 全 0 → `log10(0) = -inf` → 加 `if (sum <= 0.0f) return 0;`
2. **刻度下限以下**：标定后 `mic_db` 可能为负（补偿后静音必然如此）→ 加 `if (mic_db < 0.0f) return 0;`

第 2 条是补偿引入的**必然路径**，不修会回绕成 65526 之类的大数。

---

## 6. 标定刻度与实测值

`components/asr/asr_process_callback_decoder.c` 的 `sortDescending()`：

```c
mic_db = 10.0f * log10f_p(sum / deffectivecnt);   // 均方 → dB
mic_db = (mic_db - 18.783f) / 0.7933f;            // 标定：dB → 上报刻度
```

| 常数 | 作用 |
|---|---|
| `18.783` | **零点**：该 dB 值对应上报 0 |
| `0.7933` | **斜率**：0.7933 dB / 刻度单位 |

> 这两个数是早年照 **MIC 通道**「测试结果校准」出来的，对回采通道没有物理意义。
> 若要让值域更贴合律动，可重标定（见 §8「值域调优」）。

### 实测值（2026-09，中等音量）

| 场景 | 原始均方 | dB | 上报刻度 |
|---|---|---|---|
| 静音（ch1） | 8.86 | 9.58 | **−11.6 → 夹到 0** |
| 中等音量放音乐（ch1） | 217400 | 53.37 | **43.6 → 44** |
| 静音（ch0，对照） | — | — | 523 → 52.3 |
| 音乐（ch0，对照） | — | — | 779 → 77.9 |

**注意 ch0（MIC）读数明显被 ALC 压得更狠**（静音 52 → 音乐 78 只差 26 单位，
而 ch1 从 −11.6 → 43.6 差 55 单位），这从数据上验证了「选回踩通道是对的」。

### 上报 0 的歧义

WiFi 端 `ci1303_get_mic_energy()` 返回 0 可能是「静音」，也可能是「还没收到过数据」。
判断前应先调用 `ci1303_mic_energy_is_fresh(1000)` 区分。

---

## 7. 诊断开关与判读

`components/audio_in_manage/audio_in_manage_inner.c:254`：

```c
#define MIC_ENERGY_ALC_DIAG     1     // 排查完置 0 关闭
```

每约 1 秒打印一行：

```
[micdiag] ch=2 src=1 src_x10=466 rpt_x10=125 mic_x10=554 ref_x10=466 pgaL_x10=195 pgaR_x10=270
```

| 字段 | 含义 |
|---|---|
| `ch` | 实测通道数（应为 2） |
| `src` | 当前上报通道（0=MIC，1=回踩） |
| `src_x10` | 上报通道**实测值**（未补偿）×10，相当于改动前的读数 |
| `rpt_x10` | **实际上报值**（已补偿）×10 ← 与 WiFi 端收到的值对照 |
| `mic_x10` / `ref_x10` | 两路各自的实测值，供对照 |
| `pgaL_x10` / `pgaR_x10` | 两路 ALC 当前增益 ×10，单位 0.1 dB |

**判读 ALC 是否在工作**：看 `pgaR_x10` 在静音与放音乐之间是否变化。
若完全不动，说明 ALC 没触发（可能是音量太轻），换个更响的场景再测。

WiFi 端对应日志：`[rhy_led] e=<能量> lv=<电平>/<当前>`、`[rhythm] ... db= disp= on=`。

---

## 8. 配置项速查

| 宏 | 位置 | 当前值 | 说明 |
|---|---|---|---|
| `ENERGY_REPORT_CHA` | `audio_in_manage_inner.c:174` | `ENERGY_CHA_REF` | 上报哪一路（0=MIC / 1=回踩） |
| `ENERGY_ALC_COMPENSATE` | `audio_in_manage_inner.c:183` | `1` | ALC 增益补偿开关 |
| `MIC_ENERGY_ALC_DIAG` | `audio_in_manage_inner.c:254` | `1` | 诊断打印开关 |
| `ENERGY_BUFFER_CNT` | `asr_process_callback_decoder.c:83` | `12` | 统计窗口帧数（×16ms） |
| `ENERAY_EFFECTIVE_CNT` | `asr_process_callback_decoder.c:84` | `4` | 取最大的几帧求平均 |
| `NETWORK_UART_HEARTBEAT_INTERVAL_MS` | `app_main/user_config.h:140` | `100` | 心跳周期（能量上报周期） |

### 值域调优

当前刻度下静音 = 0、中等音量 = 44，一个字节（0~255）只用掉 17%。若需铺满，改标定常数：

| 方案 | 常数 | 静音 | 音乐 | 上报 255 对应 |
|---|---|---|---|---|
| 现状 | `18.783 / 0.7933` | 0 | 44 | 211 dB |
| A. 只挪零点 | `9.58 / 0.7933` | 0 | 55 | 221 dB |
| B. 挪零点+放大 | `9.58 / 0.3` | 0 | 146 | 86 dB |
| C. 激进放大 | `9.58 / 0.22` | 0 | 199 | 66 dB |

**动斜率前务必在「最大音量」下测一组真实峰值**——方案 C 只剩 12 dB 余量，音量拧大就会一直贴 255。

### 消费方需同步的参数

改标定后，WiFi 端这几个数要一起改（详见 Jieli 工程 `.cursor/skills/ac791n-7911bb-robot-v2/feature-rhythm.md` §8.2）：

| 位置 | 宏 | 当前值 |
|---|---|---|
| `apps/common/ui/lcd_driver/lcd_jk020ja001.c` | `RHY_ENTER_DB` / `RHY_EXIT_DB` / `RHY_MID_DB` / `RHY_PEAK_DB` | 12 / 9 / 32 / 55 |
| `apps/wifi_story_machine/board/wl82/board_7911B_develop_cfg.h` | `TCFG_WS2812_RHYTHM_LV_MIN` / `LV_MAX` | 12 / 55 |
| `apps/common/asr/Ci1303/ci1303_uart.h` | `CI1303_ENERGY_SILENT` / `MUSIC` / `FULL` | 0 / 44 / 55 |

---

## 9. 变更记录

| 日期 | 变更 |
|---|---|
| 2026-09-16 | 新增 `energy_sort_print_task`，每 100ms 统计并打印能量；能量值填入心跳包 `fill_data` 最高字节 |
| 2026-09-16 | 心跳周期 1000ms → 100ms；音频包 `fill_data` 也改用 `energy_fill_data()` |
| 2026-09-17 | **定位双通道问题**：能量统计改为按通道取值，不再混合两路 |
| 2026-09-17 | **定位 ALC 问题**：新增 `inner_codec_get_mic_pga_gain_db()` 与 `energy_alc_uncompensate()` 做增益补偿 |
| 2026-09-17 | 上报通道切到回踩（`ENERGY_REPORT_CHA = ENERGY_CHA_REF`），用于喇叭声音律动 |
| 2026-09-17 | 统计窗口 120/40 → 12/4（1.92s → 192ms） |
| 2026-09-17 | 修复 `log10(0)` 与负值转 `uint16_t` 两处 UB |
| 2026-09-17 | WiFi 端同步重标定：点阵门限 58/55/66/75 → 12/9/32/55；WS2812 去掉基线微分放大改直接 VU |

---

## 10. 编译环境备注

| 项 | 值 |
|---|---|
| 工具链 | `D:\src\other\riscv-nuclei-elf-gcc-9.2.0\gcc_fix_raissrc\bin` |
| 构建 | `make -C projects/offline_asr_llm_aiot_uart_sample/project_file -j12` |
| 产物 | `project_file/build/*.elf` + `firmware/user_code/[0]code.bin` |
| 依赖 | 编译前需修改过的 `.c` 文件时间戳更新，否则 make 不重编 |

---

*本文档由排查「静音也有 50dB / 音乐时 73-76 挤成一团」这一问题整理而成。*
