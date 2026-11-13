# 遥控器串口接收模块 · 使用说明

只依赖 STM32 HAL。包含两个文件：**`remote.h`、`remote.c`**。

模块内部完成帧同步、CRC 校验、数据解析，使用者只需初始化一次，然后周期读取结构体。

---

## 1. 串口配置要求

| 项目 | 要求 |
|---|---|
| 波特率 | **115200** |
| 数据格式 | **8N1**（8 数据位、无校验、1 停止位） |
| 中断 | **必须开启该 UART 的全局中断（NVIC）** |

CubeMX 里勾选该 UART 的 NVIC 使能，否则收不到数据。

---

## 2. 接入步骤

### 2.1 拷贝文件
把 `remote.h`、`remote.c` 放进工程，加入编译。

### 2.2 在 `main.c` 中初始化

```c
/* USER CODE BEGIN Includes */
#include "remote.h"

/* 指定遥控器使用的串口句柄：换串口只改这一行 */
#define REMOTE_CONTROL_UART_HANDLE   (&huart6)
/* USER CODE END Includes */
```

```c
/* USER CODE BEGIN 2 */
  RemoteControl_Start(REMOTE_CONTROL_UART_HANDLE);   // 必须传全局句柄
/* USER CODE END 2 */
```

### 2.3 串口回调（二选一）

`remote.c` 末尾已定义：

```c
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart);
void HAL_UART_ErrorCallback(UART_HandleTypeDef *huart);
```

- **工程里没有这两个函数** → 直接用，无需改动。
- **工程里已有这两个函数** → 会链接冲突。把 `remote.c` 末尾这两个删掉，
  在自己已有的回调里转发（模块内部会自动过滤非遥控串口）：

```c
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
  RemoteControl_RxCpltCallback(huart);
}

void HAL_UART_ErrorCallback(UART_HandleTypeDef *huart)
{
  RemoteControl_UART_ErrorCallback(huart);
}
```

---
## 3. 读取数据

直接读全局变量 `g_remote_control` 的字段即可（Cortex-M 上单字节字段读写是原子的，中断写、主循环读不会撕裂）。

推荐用法——把"已上线"和"未上线/已掉线"分开处理：

```c
if (g_remote_control.power_on &&
    RemoteControl_IsOnline(300))        // 最近 300ms 内收到过有效帧
{
    // 正常控制
    // g_remote_control.left_x / left_y / right_x / right_y ： -100 ~ 100
    // g_remote_control.left_gear / right_gear               ： 1 或 2
}
else
{
    // 未上线或已掉线：停车 / 归零
}
```

`RemoteControl_IsOnline(timeout_ms)` 用来识别掉线：遥控器断连后 `power_on` 会保持最后一帧的值不变，
所以单看 `power_on` 不够，要配合超时判断。`300ms` 是模块默认超时（`REMOTE_CONTROL_TIMEOUT_MS`）。

> 只有在需要"把整份数据一次性拷出来传给别人"时，才用 `RemoteControl_GetSnapshot(&rc)`；
> 日常控制循环读一两个字段，直接读 `g_remote_control.xxx` 即可。

## 4. 数据结构

```c
typedef struct
{
    uint8_t  sequence;           // 帧序号，0~255 循环
    bool     power_on;           // false=未上线，true=已上线
    int8_t   left_x;             // 左摇杆 X：-100 ~ 100
    int8_t   left_y;             // 左摇杆 Y：-100 ~ 100
    int8_t   right_x;            // 右摇杆 X：-100 ~ 100
    int8_t   right_y;            // 右摇杆 Y：-100 ~ 100
    uint8_t  left_gear;          // 左档位：未上线=0，上线后=1 或 2
    uint8_t  right_gear;         // 右档位：未上线=0，上线后=1 或 2
    uint32_t last_update_ms;     // 最近一次有效帧的时间戳（HAL_GetTick）
    uint32_t valid_frame_count;  // 累计有效帧数
    uint32_t crc_error_count;    // 累计 CRC 错误次数（调试用）
} RemoteControlData_t;
```

**协议层对有效帧的校验规则（代码实际执行的）：**

| 条件 | 要求 |
|---|---|
| `power_on == true`（上线） | `left_gear`、`right_gear` 必须为 **1 或 2** |
| `power_on == false`（未上线） | 四个摇杆 **全为 0**，`left_gear`、`right_gear` **全为 0** |

不满足上述规则的帧会被直接丢弃（不更新结构体，也不计入 `crc_error_count`）。
因此，**使用方永远不会读到"未上线但 gear 非 0"或"上线但 gear 为 3"的数据**：
- 从未收到有效帧 → `GetSnapshot` 返回 `false`，结构体内容全为 0（`Start` 时已清零）。
- 收到过有效帧后掉线 → `GetSnapshot` 仍返回 `true`（会拿到最后一帧数据），
  用 `IsOnline()` 判断是否超时，超时后请自行按"未上线"处理。

