# offline_asr_llm_aiot_uart_sample 代码逻辑框架分析

本文档保存对当前 active 工程 `projects/offline_asr_llm_aiot_uart_sample` 的代码逻辑框架分析。

> 分析日期：2026-08-04  
> 工程路径：`D:\src\other\CI13XX_SDK_LLM_AIOT_2.1.2`  
> Active sample：`projects/offline_asr_llm_aiot_uart_sample`

---

## 1. 项目整体定位

该工程是 Chipintelli CI130X SDK 中的 **离线 ASR + LLM/AIoT + UART 音频交互** 示例工程。

整体功能可以概括为：

```text
本地麦克风采音
  ↓
音频前端处理 / VAD / 降噪 / ASR
  ↓
本地判断唤醒词、命令词
  ↓
唤醒后通过 UART 上传 PCM/压缩音频给网络侧 / WiFi / LLM 模块
  ↓
网络侧返回播放数据 / 控制命令
  ↓
本地播放 MP3/TTS/音频流
```

该工程主要承担：

1. 本地唤醒词识别。
2. 唤醒后进入命令词识别或对话状态。
3. 通过 UART 与外部网络/WiFi/LLM 模块通信。
4. 上传本地采集的语音数据。
5. 接收网络端返回的音频数据并播放。
6. 支持本地 ASR 命令处理、音量控制、退出唤醒、多轮对话、全双工上传播放等逻辑。

---

## 2. 关键目录结构

```text
projects/offline_asr_llm_aiot_uart_sample/
├── app/
│   ├── app_main/
│   │   ├── main.c                  # 系统入口、硬件初始化、任务创建
│   │   ├── user_config.h           # 项目核心宏配置
│   │   ├── system_msg_deal.c       # 系统消息、ASR 消息、唤醒状态机
│   │   ├── user_msg_deal.c         # 用户命令/语义/外设消息处理
│   │   ├── ci_nlp_user.c           # NLP 后处理
│   │   └── ci_ssp_config.c         # 音频前端/算法链配置
│   │
│   └── app_audio_handle/
│       ├── cias_voice_upload.c          # 唤醒、VAD、PCM/压缩上传任务
│       ├── cias_audio_data_handle.c     # 网络命令处理、云端播放控制
│       ├── cias_uart_protocol.c         # UART1 收发底层封装
│       ├── cias_network_msg_protocol.c  # 网络协议接收解析
│       ├── cias_network_msg_send_task.c # 网络协议打包发送
│       ├── cias_voice_plyer_handle.c    # 云端播放器状态/回调
│       └── cias_aiot_protocol.h         # AIoT 全局运行状态和功能参数
│
├── project_file/
│   ├── makefile                  # 工程构建、算法/库/链接脚本选择
│   ├── source_file.prj           # 源文件/头文件/库文件清单，新增源文件应改这里
│   └── build/source_file.mk      # 自动生成文件，不应手改
│
└── firmware/
    └── ...                       # ASR/DNN/voice/user_code 等固件资源
```

---

## 3. 构建和配置关系

### 3.1 Active 工程选择

VS Code 配置中 active project 指向：

```text
offline_asr_llm_aiot_uart_sample
```

相关文件：

```text
.vscode/settings.json
.vscode/tasks.json
```

构建任务会进入：

```text
projects/offline_asr_llm_aiot_uart_sample/project_file
```

执行 `make`。

### 3.2 构建入口

主 makefile：

```text
projects/offline_asr_llm_aiot_uart_sample/project_file/makefile
```

该 makefile 主要负责：

1. 引入 SDK 公共构建规则。
2. 选择算法组合。
3. 选择音频上传方式。
4. 选择压缩库。
5. 选择链接脚本。
6. 增加项目宏定义。
7. 输出 ELF、LST、BIN 等产物。

构建输出通常包括：

```text
projects/offline_asr_llm_aiot_uart_sample/project_file/build/
projects/offline_asr_llm_aiot_uart_sample/firmware/user_code/[0]code.bin
```

### 3.3 源文件清单

源文件清单是：

```text
projects/offline_asr_llm_aiot_uart_sample/project_file/source_file.prj
```

`build/source_file.mk` 由 `source_file.prj` 自动生成。

因此：

```text
新增/删除 .c 文件 → 修改 source_file.prj
不要手动修改 build/source_file.mk
```

### 3.4 核心配置文件

项目主要宏配置集中在：

```text
projects/offline_asr_llm_aiot_uart_sample/app/app_main/user_config.h
```

重点配置包括：

- 板型和芯片类型。
- log UART。
- 语音模块协议 UART。
- AIoT 音频交互 UART。
- 音频上传是否走 UART。
- 上传音频是否压缩。
- 云端播放格式。
- 多轮对话。
- 上传/播放全双工。
- VAD 参数。
- 播放 buffer 和上传 queue 大小。

---

## 4. 启动流程

入口文件：

```text
projects/offline_asr_llm_aiot_uart_sample/app/app_main/main.c
```

总体启动链路：

```text
main()
  ├── hardware_default_init()
  ├── platform_init()
  ├── welcome()
  ├── xTaskCreate(task_init)
  └── vTaskStartScheduler()
```

### 4.1 `hardware_default_init()`

主要负责硬件基础初始化：

```text
SystemInit()
ECLIC 中断优先级配置
全局中断打开
平台硬件初始化
maskrom lib 初始化
DMA 中断模式和 reset release
```

### 4.2 `platform_init()`

主要负责平台服务初始化：

```text
log 串口初始化
可选 CLI 初始化
检查 log UART 与协议 UART 是否冲突
SysView 初始化
IWDG 看门狗初始化
```

### 4.3 `task_init()`

`task_init()` 是系统真正的初始化核心。

大致流程：

```text
task_init()
  ├── DSU/算法内存边界初始化
  ├── codec manager 初始化
  ├── audio input codec 注册
  ├── nuclear_com 核间通信初始化
  ├── decoder / flash / dnn / asr / vad / codec 端口初始化
  ├── ciss 状态共享初始化
  ├── 设置 ASR decoder、intent、通道数、beam 等共享状态
  ├── 设置 VOX/VAD/NN 降噪/AEC 参数
  ├── 注册 ci_ssp 音频前端链
  ├── 注册 audio_capture 采音配置
  ├── decoder_task_init_port()
  ├── 创建 audio_in_manage_inner_task
  ├── audio_play_init() 或 sap_init()
  ├── alg_model_init()
  ├── 可选 DOA/SED/CWSL 等模块初始化
  ├── cias_online_func_init()
  ├── nlp_module_init()
  ├── userapp_initial()
  ├── sys_msg_task_initial()
  ├── 创建 UserTaskManageProcess
  ├── 配置 ASR 后处理参数
  ├── 发送 CIAS_AUDIO_SYS_READY
  └── 周期打印 task/heap 状态
```

---

## 5. 运行时任务框架

该工程是 FreeRTOS 多任务结构，主要通过 queue、timer、semaphore、stream buffer 进行模块间通信。

核心任务包括：

| 任务 | 主要来源 | 作用 |
|---|---|---|
| `init task` | `main.c` | 系统初始化任务 |
| `audio_in_manage_inner_task` | SDK audio 模块 | 音频采集和前端处理 |
| decoder task | ASR/decoder port | ASR decoder 处理 |
| `UserTaskManageProcess` | `system_msg_deal.c` | 系统消息中心、ASR 结果分发、唤醒状态机 |
| `network_recv_data_task` | `cias_network_msg_protocol.c` | 从 UART stream buffer 接收并解析网络侧数据 |
| `network_send_data_task` | `cias_network_msg_send_task.c` | 从发送队列取命令/音频包并通过 UART 发送 |
| `vad_state_handle_task` | `cias_voice_upload.c` | 轮询 VAD 状态，触发 VAD start/end |
| `record_data_upload_task` | `cias_voice_upload.c` | 从 queue/buffer 取上传音频并发送 |
| speex/opus/g722 encode task | `cias_voice_upload.c` | 可选音频编码任务 |
| `request_play_data_task` | `cias_audio_data_handle.c` | 播放 buffer 低水位时请求云端播放数据 |

总体任务关系：

```text
task_init()
  ├── audio_in_manage_inner_task
  ├── decoder task
  ├── UserTaskManageProcess
  └── cias_online_func_init()
       ├── network_recv_data_task
       ├── network_send_data_task
       ├── vad_state_handle_task
       ├── record_data_upload_task
       └── request_play_data_task
```

---

## 6. 系统消息中心

核心文件：

```text
projects/offline_asr_llm_aiot_uart_sample/app/app_main/system_msg_deal.c
projects/offline_asr_llm_aiot_uart_sample/app/app_main/system_msg_deal.h
```

系统消息队列初始化：

```text
sys_msg_task_initial()
  ├── sys_msg_queue = xQueueCreate(16, sizeof(sys_msg_t))
  └── WakeupMutex = xSemaphoreCreateMutex()
```

消息发送封装：

```text
send_msg_to_sys_task()
```

该函数会根据当前是否处于中断上下文，选择：

```text
xQueueSendFromISR()
或
xQueueSend()
```

系统消息主循环：

```text
UserTaskManageProcess()
  └── xQueueReceive(sys_msg_queue)
       ├── SYS_MSG_TYPE_ASR       → sys_deal_asr_msg()
       ├── SYS_MSG_TYPE_CMD_INFO  → sys_deal_cmd_info_msg()
       ├── SYS_MSG_TYPE_PLAY      → prompt_play_by_cmd_id()
       ├── SYS_MSG_TYPE_NLP       → NLP 处理
       ├── SYS_MSG_TYPE_NLP_RESULT→ NLP 结果处理
       └── 其他                   → deal_userdef_msg()
```

系统消息中心主要负责：

1. 处理 ASR 识别结果。
2. 判断唤醒词/命令词。
3. 管理唤醒状态。
4. 管理退出唤醒 timer。
5. 切换唤醒词模型和命令词模型。
6. 分发用户自定义消息。
7. 触发本地播报。

---

## 7. 唤醒状态机

唤醒状态主要由 `system_msg_deal.c` 管理。

### 7.1 唤醒词识别成功

链路：

```text
ASR GOOD_RESULT
  ↓
cmd_info_is_wakeup_word(cmd_handle) == true
  ↓
asr_wakeup_on_handle()
  ↓
ciss_set(CI_SS_CMD_STATE, CI_SS_CMD_IS_WAKEUP)
  ↓
enter_wakeup_deal(...)
  ↓
播放唤醒提示音
  ↓
切换到正常命令词模型
```

其中 `asr_wakeup_on_handle()` 位于：

```text
projects/offline_asr_llm_aiot_uart_sample/app/app_audio_handle/cias_voice_upload.c
```

主要动作：

```text
清除 VOX/VAD 状态
刷新 AIoT 参数
设置 is_wake_up_flag
清除云端播放/解析状态
向网络侧发送 WAKE_UP
执行用户唤醒回调
```

### 7.2 进入唤醒状态

`enter_wakeup_deal()` 主要做：

```text
获取 WakeupMutex
设置 wakeup_state = SYS_STATE_WAKEUP
设置 CI_SS_WAKING_UP_STATE = CI_SS_WAKEUPED
设置 CI_SS_WAKING_UP_STATE_FOR_SSP = CI_SS_WAKEUPED
重启 exit_wakeup_timer
播放唤醒提示音或直接触发回调
必要时启动声纹识别
```

### 7.3 唤醒后切换命令词模型

唤醒提示音播放完成后，会投递：

```text
MSG_CMD_INFO_STATUS_POST_CHANGE_ASR_NORMAL_WORD
```

之后调用：

```text
change_asr_normal_word()
```

其核心动作：

```text
cmd_info_change_cur_model_group(0)
ignore_asr_msg++
```

即从唤醒词模型切换到命令词模型。

### 7.4 退出唤醒

退出唤醒来源包括：

1. `exit_wakeup_timer` 超时。
2. 网络端下发退出唤醒命令。
3. 本地命令词触发退出。
4. 云端/用户逻辑强制退出。

超时链路：

```text
exit_wakeup_timer_callback()
  ↓
MSG_CMD_INFO_STATUS_EXIT_WAKEUP
  ↓
sys_deal_cmd_info_msg()
  ↓
exit_wakeup_deal(1)
  ↓
播放退出唤醒提示音
  ↓
MSG_CMD_INFO_STATUS_POST_CHANGE_ASR_WAKEUP_WORD
  ↓
change_asr_wakeup_word()
  ↓
set_state_exit_wakeup()
  ↓
切回唤醒词模型组
```

`set_state_exit_wakeup()` 主要做：

```text
asr_wakeup_exit_handle()
停止 exit_wakeup_timer
wakeup_state = SYS_STATE_UNWAKEUP
CI_SS_WAKING_UP_STATE = CI_SS_NO_WAKEUP
CI_SS_CMD_STATE = CI_SS_CMD_IS_NULL
cmd_info_change_cur_model_group(1)
```

---

## 8. ASR 结果处理逻辑

核心函数：

```text
sys_deal_asr_msg()
```

主要处理：

```text
MSG_ASR_STATUS_GOOD_RESULT
MSG_ASR_STATUS_NO_RESULT
MSG_ASR_STATUS_VAD_START
MSG_ASR_STATUS_VAD_END
MSG_ASR_STATUS_AUDIO_PROCESS
```

### 8.1 GOOD_RESULT 是唤醒词

```text
ASR GOOD_RESULT
  ↓
cmd_info_is_wakeup_word(cmd_handle)
  ↓
asr_wakeup_on_handle()
  ↓
enter_wakeup_deal()
```

同时会：

```text
设置 CI_SS_CMD_STATE = CI_SS_CMD_IS_WAKEUP
记录 weakup_asr_hander
刷新 AIoT 状态
向网络侧发送 WAKE_UP
```

### 8.2 GOOD_RESULT 是普通命令词

前提是当前处于唤醒状态。

处理链路：

```text
普通命令词识别成功
  ↓
设置 CI_SS_CMD_STATE = CI_SS_CMD_IS_NORMAL
  ↓
刷新唤醒持续时间
  ↓
获取 semantic_id 和 cmd_id
  ↓
向网络侧发送 LOCAL_ASR_RESULT_NOTIFY
  ↓
设置 local_asr_finish_flag = true
  ↓
cias_aiot_vad_end_handle(1)
  ↓
deal_asr_msg_by_cmd_id()
  ↓
如果未处理，再 deal_asr_msg_by_semantic_id()
  ↓
如果仍未处理，按命令配置播放提示音
```

### 8.3 VAD 状态对 ASR busy/idle 的影响

当收到：

```text
MSG_ASR_STATUS_VAD_START
```

会设置：

```text
ASR busy
```

当收到：

```text
MSG_ASR_STATUS_NO_RESULT
MSG_ASR_STATUS_VAD_END
```

会设置：

```text
ASR idle
```

退出唤醒时会参考 ASR busy 状态，避免正在识别时立即退出。

---

## 9. 用户命令处理

核心文件：

```text
projects/offline_asr_llm_aiot_uart_sample/app/app_main/user_msg_deal.c
```

主要职责：

1. `userapp_initial()` 初始化用户侧协议/外设。
2. `deal_userdef_msg()` 处理 key/com/i2c 等消息。
3. `deal_asr_msg_by_cmd_id()` 按命令 ID 处理本地命令。
4. `deal_asr_msg_by_semantic_id()` 按语义 ID 处理通用语义。

典型命令包括：

```text
音量加
音量减
最大音量
最小音量
开启多轮对话
关闭多轮对话
退出对话/退出唤醒
```

处理优先级：

```text
cmd_id 优先
semantic_id 兜底
默认命令提示音最后处理
```

---

## 10. NLP 后处理

核心文件：

```text
projects/offline_asr_llm_aiot_uart_sample/app/app_main/ci_nlp_user.c
```

入口：

```text
nlp_module_init()
```

主要负责：

1. 配置 NLP 结果输出函数。
2. 配置多意图识别相关词表。
3. 配置误识消除词。
4. 在 `nlp_result()` 中输出 intent 结果。

NLP 结果会区分：

```text
唤醒词结果
普通命令词结果
```

---

## 11. 音频前端/算法链配置

核心文件：

```text
projects/offline_asr_llm_aiot_uart_sample/app/app_main/ci_ssp_config.c
```

该文件配置：

```text
STFT / ISTFT
传统降噪
AI NN 降噪
AEC
VOX/VAD
DOA / AI DOA
音频输出路径
采音通道数
参考通道数
```

其中 `ci_ssp` 数组/结构根据宏把不同算法模块挂到音频前端链路上。

`audio_capture` 根据以下宏决定采集通道：

```text
USE_AEC_MODULE
USE_AI_DOA
USE_DUAL_MIC_ANY
HOST_MIC_USE_NUMBER
HOST_CODEC_CHA_NUM
```

所以如果改算法组合，通常需要同时检查：

```text
project_file/makefile
app/app_main/user_config.h
app/app_main/ci_ssp_config.c
链接脚本 lds/
算法库 libs/
```

---

## 12. AIoT 全局状态结构

核心头文件：

```text
projects/offline_asr_llm_aiot_uart_sample/app/app_audio_handle/cias_aiot_protocol.h
```

### 12.1 运行状态 `gCiasAiotRunParam`

定义在：

```text
projects/offline_asr_llm_aiot_uart_sample/app/app_audio_handle/cias_voice_upload.c
```

主要包含：

```text
唤醒状态
VAD 状态
上传帧计数
PCM stream buffer
上传 queue
播放 stream buffer
云端播放状态
云端响应超时状态
用户回调函数
```

典型字段：

```text
is_wake_up_flag
is_vad_on_flag
stop_collect_pcm_flag
compress_pcm_to_wifi_flag
upload_pcm_to_wifi_flag
local_asr_finish_flag
request_play_data_flag
play_cloud_data_flag
cloud_play_state
```

### 12.2 功能参数 `gCiasAiotFuncParam`

同样定义在：

```text
projects/offline_asr_llm_aiot_uart_sample/app/app_audio_handle/cias_voice_upload.c
```

主要包含：

```text
VAD 过滤帧数
VAD end 静音参数
VAD start 最大时长
唤醒持续时间
是否多轮对话
是否上传/播放全双工
VAD start 是否停止播放
播放模式
是否上传降噪音频
是否播放进入/退出唤醒提示音
```

这些参数既可以来自 `user_config.h` 默认宏，也可能被网络端命令动态修改。

---

## 13. VAD 和语音上传链路

核心文件：

```text
projects/offline_asr_llm_aiot_uart_sample/app/app_audio_handle/cias_voice_upload.c
```

### 13.1 唤醒处理

```text
asr_wakeup_on_handle()
  ├── 清 VOX/VAD 状态
  ├── 刷新 AIoT 参数
  ├── is_wake_up_flag = true
  ├── 清云端播放/解析状态
  ├── 发送 WAKE_UP
  └── 执行 customer_wakeup_on_callback
```

退出唤醒：

```text
asr_wakeup_exit_handle()
  ├── is_wake_up_flag = false
  ├── 发送 EXIT_WAKE_UP
  └── 执行 customer_wakeup_exit_callback
```

### 13.2 VAD start

```text
vad_state_handle_task()
  ↓
检测 CI_SS_VAD_START
  ↓
cias_aiot_vad_start_handle()
```

`cias_aiot_vad_start_handle()` 主要做：

```text
回退唤醒前 PCM 帧
设置 is_vad_on_flag
设置 compress_pcm_to_wifi_flag
设置 upload_pcm_to_wifi_flag
发送 VAD_START
执行 customer_vad_start_callback
```

### 13.3 VAD end

```text
vad_state_handle_task()
  ↓
检测 CI_SS_VAD_END 或 CI_SS_VOX_VAD_FORCE_END
  ↓
cias_aiot_vad_end_handle()
```

`cias_aiot_vad_end_handle()` 主要做：

```text
stop_collect_pcm_flag = true
等待本地 ASR 或压缩/上传完成
清理 VAD/上传状态
刷新 AIoT 参数
发送 PCM_FINISH 或 VAD_END 相关命令
执行 customer_vad_end_callback
```

### 13.4 上传任务

上传任务初始化：

```text
voice_upload_task_init()
  ├── 创建云端响应超时 timer
  ├── 创建 pcm_play_data_stream_semaphore
  ├── 创建可选 speex/opus/g722 编码任务
  ├── 创建 pcm_upload_queue
  ├── 创建 pcm_compress_stream_buffer
  ├── 创建 pcm_play_data_stream_buffer
  ├── 创建 vad_state_handle_task
  └── 创建 record_data_upload_task
```

实际上传任务：

```text
record_data_upload_task()
  ├── 从 pcm_upload_queue 或 pcm_compress_stream_buffer 获取音频
  └── voice_data_packet_and_send(PCM_MIDDLE, ...)
```

---

## 14. UART 网络协议栈

AIoT 和外部 WiFi/网络模块通过 UART 通信，主要分三层：

```text
cias_uart_protocol.c              # UART 中断和底层收发
cias_network_msg_protocol.c       # 接收协议包、解析命令
cias_network_msg_send_task.c      # 组包、发送队列、发送任务
```

### 14.1 UART 底层

核心文件：

```text
projects/offline_asr_llm_aiot_uart_sample/app/app_audio_handle/cias_uart_protocol.c
```

初始化：

```text
network_uart_port_init()
```

接收：

```text
UARTx_IRQHandler()
  ├── 从 UART FIFO 读取字节
  └── 写入 network_msg_rcv_stream_buffer
```

发送：

```text
network_send()
  ├── 获取 network_send_sem
  ├── 调用 network_send_Byte() 逐字节发送
  └── 释放 network_send_sem
```

### 14.2 接收协议解析

核心文件：

```text
projects/offline_asr_llm_aiot_uart_sample/app/app_audio_handle/cias_network_msg_protocol.c
```

接收任务：

```text
network_recv_data_task()
  ├── 从 network_msg_rcv_stream_buffer 按字节读取
  ├── 查找 CIAS_STANDARD_MAGIC
  ├── 解析 cias_standard_head_t
  ├── 根据 len 收满完整包
  └── wifi_msg_deal()
       └── aiot_wifi_msg_callback()
```

### 14.3 发送协议组包

核心文件：

```text
projects/offline_asr_llm_aiot_uart_sample/app/app_audio_handle/cias_network_msg_send_task.c
```

发送队列：

```text
network_msg_queue
```

组包函数：

```text
broadlink_frame_create()
```

协议头字段：

```text
magic
type
len
version
checksum
fill_data
```

常用发送接口：

```text
cias_send_cmd()
cias_send_cmd_and_data()
voice_data_packet_and_send()
```

发送任务：

```text
network_send_data_task()
  ├── xQueueReceive(network_msg_queue)
  ├── 判断 CMD_AND_DATA / ONLY_CMD / AUDIO_MIDDLE
  └── network_send()
```

---

## 15. 网络端命令处理

核心函数：

```text
aiot_wifi_msg_callback()
```

位置：

```text
projects/offline_asr_llm_aiot_uart_sample/app/app_audio_handle/cias_audio_data_handle.c
```

该函数根据协议包 `type` 处理网络端命令。

主要命令类型包括：

### 15.1 参数设置类

```text
SET_VAD_SENSITIVITY                  设置 VAD 灵敏度
SET_AUDIO_EXIT_WAKE_UP               设置退出唤醒/触发退出唤醒
PCM_DENOISE_ENABLE                   设置上传音频是否使用降噪数据
SET_VAD_FILTER_FRAME                 设置 VAD 过滤帧
SET_VAD_SENSITIVITY_ACTIVATE_LENTH   设置 VAD end 静音长度
SET_VAD_START_MAX_TIMEOUT            设置 VAD 最大时长
SET_WAKE_UP_CONTINUE_TIME            设置唤醒持续时间
SET_ENTER_WAKE_UP                    网络端触发进入唤醒
SET_INTERACTION_NULTI_ROUND_ENABLE   设置多轮对话
UPLOAD_PLAY_FULL_DUPLEX_EANBLE       设置上传/播放全双工
SET_AUDIO_VOLUME                     设置音量
SET_VOLUME_MUTE_STATE                设置功放静音
SET_AUDIO_START_RECORD               强制开始录音
SET_AUDIO_STOP_RECORD                强制停止录音
SET_FORCE_VAD_END                    强制 VAD end
```

### 15.2 云端播放类

```text
NET_PLAY_START
NET_PLAY_PAUSE
NET_PLAY_RESUME
NET_PLAY_STOP
PLAY_DATA_GET
PLAY_DATA_RECV
PLAY_DATA_END
PLAY_TTS_END
```

### 15.3 网络状态类

```text
WIFI_CONNECTED
WIFI_DISCONNECTED
CLOUD_CONNECTED
CLOUD_DISCONNECTED
```

---

## 16. 云端播放链路

核心文件：

```text
projects/offline_asr_llm_aiot_uart_sample/app/app_audio_handle/cias_audio_data_handle.c
projects/offline_asr_llm_aiot_uart_sample/app/app_audio_handle/cias_voice_plyer_handle.c
```

### 16.1 播放开始

网络端发送：

```text
NET_PLAY_START
```

本地处理大致为：

```text
停止当前播放
清空 outside stream
预请求若干帧播放数据
request_play_data_flag = true
play_cloud_data_flag = true
半双工时关闭 VOX/VAD 和上传
启动/恢复播放器
```

### 16.2 播放数据请求

当播放 buffer 低于阈值时：

```text
request_play_data_task()
  ↓
request_play_data_func()
  ↓
发送 PLAY_DATA_GET
```

### 16.3 播放结束

播放器回调位于：

```text
cias_voice_plyer_handle.c
```

典型回调：

```text
mp3_player_end_callbk()
tts_player_end_callbk()
```

播放结束后通常会：

```text
设置播放器 idle
更新 offset
发送 NET_PLAY_END / PLAY_TTS_END
必要时请求重连或重启播放
恢复 VOX/VAD
根据多轮对话配置决定是否继续保持唤醒
```

---

## 17. UART 角色划分

当前工程中有三类 UART 概念：

| 用途 | 典型宏 | 说明 |
|---|---|---|
| log 串口 | `CONFIG_CI_LOG_UART` | SDK log 打印 |
| 语音模块协议 UART | `UART_PROTOCOL_NUMBER` | 用户协议/模块控制 |
| AIoT 网络音频 UART | `UART_NUM_SEND_PLAY_AUDIO_NUMBER` | 上传音频、接收云端播放数据和网络命令 |

AIoT 网络音频 UART 默认通常是：

```text
UART1, 1Mbps
```

实际当前代码请以 `user_config.h` 中宏定义为准。

---

## 18. 关键协议命令

协议命令定义在：

```text
projects/offline_asr_llm_aiot_uart_sample/app/app_audio_handle/cias_network_msg_protocol.h
```

关键命令包括：

```text
LOCAL_ASR_RESULT_NOTIFY   本地 ASR 结果通知
WAKE_UP                   唤醒通知
VAD_START                 VAD 开始
VAD_END                   VAD 结束
PCM_MIDDLE                PCM/音频中间帧
PCM_FINISH                PCM 上传结束
EXIT_WAKE_UP              退出唤醒通知
NET_PLAY_START            网络播放开始
PLAY_DATA_GET             请求播放数据
PLAY_DATA_RECV            接收播放数据
PLAY_DATA_END             播放数据结束
CIAS_AUDIO_SYS_READY      系统就绪通知
```

---

## 19. 推荐阅读顺序

如果后续要继续深入源码，建议按以下顺序：

1. 启动与初始化：

```text
projects/offline_asr_llm_aiot_uart_sample/app/app_main/main.c
```

重点看：

```text
main()
hardware_default_init()
platform_init()
task_init()
```

2. 系统消息和 ASR 分发：

```text
projects/offline_asr_llm_aiot_uart_sample/app/app_main/system_msg_deal.c
```

重点看：

```text
sys_msg_task_initial()
send_msg_to_sys_task()
UserTaskManageProcess()
sys_deal_asr_msg()
sys_deal_cmd_info_msg()
enter_wakeup_deal()
exit_wakeup_deal()
```

3. 用户命令处理：

```text
projects/offline_asr_llm_aiot_uart_sample/app/app_main/user_msg_deal.c
```

重点看：

```text
userapp_initial()
deal_userdef_msg()
deal_asr_msg_by_cmd_id()
deal_asr_msg_by_semantic_id()
```

4. AIoT 初始化和网络命令处理：

```text
projects/offline_asr_llm_aiot_uart_sample/app/app_audio_handle/cias_audio_data_handle.c
```

重点看：

```text
cias_online_func_init()
aiot_wifi_msg_callback()
request_play_data_task()
request_play_data_func()
```

5. VAD 和上传：

```text
projects/offline_asr_llm_aiot_uart_sample/app/app_audio_handle/cias_voice_upload.c
```

重点看：

```text
asr_wakeup_on_handle()
asr_wakeup_exit_handle()
cias_aiot_vad_start_handle()
cias_aiot_vad_end_handle()
vad_state_handle_task()
record_data_upload_task()
voice_upload_task_init()
```

6. UART 和协议：

```text
projects/offline_asr_llm_aiot_uart_sample/app/app_audio_handle/cias_uart_protocol.c
projects/offline_asr_llm_aiot_uart_sample/app/app_audio_handle/cias_network_msg_protocol.c
projects/offline_asr_llm_aiot_uart_sample/app/app_audio_handle/cias_network_msg_send_task.c
```

重点看：

```text
network_uart_port_init()
UARTx_IRQHandler()
network_recv_data_task()
wifi_msg_deal()
broadlink_frame_create()
network_send_data_task()
voice_data_packet_and_send()
```

7. 配置：

```text
projects/offline_asr_llm_aiot_uart_sample/app/app_main/user_config.h
projects/offline_asr_llm_aiot_uart_sample/app/app_main/ci_ssp_config.c
projects/offline_asr_llm_aiot_uart_sample/project_file/makefile
projects/offline_asr_llm_aiot_uart_sample/project_file/source_file.prj
```

---

## 20. 总体逻辑框架简图

```text
main()
  ├── hardware_default_init()
  ├── platform_init()
  ├── xTaskCreate(task_init)
  └── vTaskStartScheduler()

task_init()
  ├── codec / nuclear_com / ciss / ASR / VAD / decoder init
  ├── audio_in_manage_inner_task
  ├── audio_play_init() / sap_init()
  ├── alg_model_init()
  ├── cias_online_func_init()
  │    ├── network_uart_port_init()
  │    ├── audio_player_param_init()
  │    ├── request_play_data_task
  │    ├── network_recv_data_task
  │    ├── network_port_recv_queue_init()
  │    ├── voice_upload_task_init()
  │    │    ├── vad_state_handle_task
  │    │    ├── record_data_upload_task
  │    │    └── optional speex/opus/g722 encode task
  │    ├── network_send_task_init()
  │    └── network_send_data_task
  ├── nlp_module_init()
  ├── userapp_initial()
  ├── sys_msg_task_initial()
  └── UserTaskManageProcess

ASR good result
  └── sys_deal_asr_msg()
       ├── wakeup word
       │    ├── asr_wakeup_on_handle()
       │    ├── enter_wakeup_deal()
       │    └── change_asr_normal_word()
       │
       └── normal command in wakeup
            ├── LOCAL_ASR_RESULT_NOTIFY
            ├── cias_aiot_vad_end_handle(1)
            ├── deal_asr_msg_by_cmd_id()
            ├── deal_asr_msg_by_semantic_id()
            └── prompt play if needed

VAD start/end
  └── vad_state_handle_task()
       ├── cias_aiot_vad_start_handle()
       ├── record_data_upload_task()
       ├── voice_data_packet_and_send()
       ├── network_send_data_task()
       └── UART to network/WiFi side

Network/WiFi side response
  └── UART IRQ
       └── network_msg_rcv_stream_buffer
            └── network_recv_data_task()
                 └── wifi_msg_deal()
                      └── aiot_wifi_msg_callback()
                           ├── config command
                           ├── NET_PLAY_START
                           ├── PLAY_DATA_RECV
                           └── PLAY_DATA_END
```

---

## 21. 总结

该工程可以理解为一个“**本地语音入口 + UART 云端 AIoT/LLM 音频通道**”示例：

```text
本地负责：
  - 唤醒词识别
  - VAD
  - 本地 ASR 命令
  - 音频前端处理
  - 本地播放器
  - 系统状态机

外部网络侧负责：
  - 云端/LLM 语音处理
  - 音频响应生成
  - 播放数据下发
  - 动态参数配置

UART 负责连接两者：
  - 上传 PCM/压缩音频
  - 通知 WAKE_UP / VAD_START / PCM_FINISH / LOCAL_ASR_RESULT_NOTIFY
  - 接收 NET_PLAY_START / PLAY_DATA_RECV / 参数设置命令
```

后续如果要修改功能，通常优先看：

```text
user_config.h                 # 开关和参数
system_msg_deal.c             # 唤醒/ASR 状态机
user_msg_deal.c               # 本地命令逻辑
cias_voice_upload.c           # VAD 和上传
cias_audio_data_handle.c      # 网络命令和播放
cias_network_msg_send_task.c  # 协议发送
cias_network_msg_protocol.c   # 协议接收
cias_uart_protocol.c          # UART 底层
```
