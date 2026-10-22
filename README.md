# 第一次嵌入式作业：基于STM32最小系统板的流水灯实现

- 姓名：罗智允
- 学号：260201425

## 一、实验目的

使用 STM32CubeMX + STM32CubeIDE，在 STM32F103C8T6 最小系统板上实现 PA0~PA6 共 7 个 LED 的往返流水灯效果，掌握 GPIO 输出控制与 HAL 库的基本用法。

## 二、硬件连接

| 项目 | 说明 |
| --- | --- |
| 主控 | STM32F103C8T6 最小系统板（8 MHz 晶振） |
| LED | 7 个，分别接 PA0 ~ PA6 |
| 限流电阻 | 每个 LED 串联 1 kΩ 电阻 |
| 接法 | PAx → 1 kΩ 电阻 → LED 阳极（长脚），LED 阴极（短脚）→ GND |

LED 为**高电平点亮**：引脚输出 `GPIO_PIN_SET`（高电平）时灯亮，`GPIO_PIN_RESET`（低电平）时灯灭。

## 三、CubeMX 配置

- 系统时钟：HSE 8 MHz 晶振，PLL ×9，SYSCLK = 72 MHz
- PA0 ~ PA6：`GPIO_Output`，推挽输出（Push Pull）、无上下拉、Low speed，初始电平为低
- SYS：SWD 调试接口（PA13/PA14）
- 工程工具链：STM32CubeIDE

## 四、控制 LED 的代码

### 1. 引脚与状态变量

```c
/* USER CODE BEGIN 1 */
uint16_t led_pin[] = {GPIO_PIN_0, GPIO_PIN_1, GPIO_PIN_2, GPIO_PIN_3,
                      GPIO_PIN_4, GPIO_PIN_5, GPIO_PIN_6};
uint8_t led_count = sizeof(led_pin) / sizeof(led_pin[0]);
int8_t led_index = 0;
uint32_t delay_time = 500; // 每步 500 ms
int8_t direction = 1; // 1 表示向右流动，-1 表示向左流动
/* USER CODE END 1 */
```

### 2. 流水灯主循环

```c
while (1)
{
  /* USER CODE BEGIN 3 */
  for (uint8_t i = 0; i < led_count; i++) {
    HAL_GPIO_WritePin(GPIOA, led_pin[i], GPIO_PIN_RESET);
  }
  HAL_GPIO_WritePin(GPIOA, led_pin[led_index], GPIO_PIN_SET);

  HAL_Delay(delay_time);

  if (led_index >= led_count - 1) {
    direction = -1;
  } else if (led_index <= 0) {
    direction = 1;
  }
  led_index += direction;
  /* USER CODE END 3 */
}
```

## 五、代码说明

1. **`led_pin[]`**：用数组保存 7 个 LED 的引脚号（`GPIO_PIN_0` ~ `GPIO_PIN_6`），后续通过下标访问，方便统一循环处理。
2. **`led_count`**：用 `sizeof(led_pin) / sizeof(led_pin[0])` 自动计算 LED 数量，以后增删灯只需要改数组，不用再改循环次数。
3. **`led_index` / `direction`**：`led_index` 记录当前点亮第几个灯；`direction` 为 `1` 时下标递增（PA0 → PA6），为 `-1` 时下标递减（PA6 → PA0）。两者都使用 `int8_t` 有符号类型，避免下标递减时发生无符号回绕。
4. **先全灭、再点亮一个**：循环开始时用 `for` 把 7 个引脚全部置低（全部熄灭），然后只把 `led_pin[led_index]` 置高，这样就只有当前这一个灯亮，形成"一个亮点在移动"的流水效果。
5. **`HAL_Delay(500)`**：每一步停留 500 ms，控制流水速度。
6. **往返方向切换（关键）**：先判断边界，再移动下标。
   - 当 `led_index` 到达 `6`（最后一个灯）时，把 `direction` 改为 `-1`，下一次下标变成 5，开始往回走；
   - 当 `led_index` 回到 `0`（第一个灯）时，把 `direction` 改回 `1`，重新向右流动。
   - 这种"先判断、后移动"的顺序保证 `led_index` 始终落在 0 ~ 6 之间，不会出现 `led_pin[7]` 或 `led_pin[-1]` 这样的数组越界访问。

## 六、运行效果

上电后 PA0 的 LED 先亮 0.5 s，随后依次点亮 PA1、PA2 …… PA6；到达 PA6 后方向反转，再依次返回 PA0，如此循环，形成 7 个 LED 的往返流水灯。

## 七、编译与下载

1. 用 STM32CubeIDE 打开本工程（`.project` / `.cproject`），点击 `Build` 编译；
2. 用 ST-Link 连接开发板（SWDIO、SWCLK、GND、3.3V），选择 `Debug`/`Run` 下载程序；
3. 下载完成后按一下复位键，即可看到流水灯效果。

> 注：工程中 `SystemClock_Config()` 在 HSE 起振失败时会自动切换到内部 HSI（8 MHz）运行，保证程序不会卡死在 `Error_Handler` 中；正常情况仍使用 72 MHz 的 HSE + PLL 时钟。
