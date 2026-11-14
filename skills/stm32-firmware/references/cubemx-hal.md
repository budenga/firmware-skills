# CubeMX + HAL patterns

## Contents
1. Project layout and where code goes
2. C++ in a CubeMX project
3. UART (interrupt, DMA, receive-to-idle) and printf
4. I2C / SPI
5. ADC with DMA
6. Timers, PWM, input capture, EXTI
7. HAL vs LL
8. Low power
9. Flash writes and EEPROM emulation
10. Watchdog and reset cause
11. FreeRTOS in CubeMX

---

## 1. Project layout and where code goes

```
Core/Inc, Core/Src        generated (main.c, stm32xx_it.c, stm32xx_hal_msp.c, usart.c, ...)
Drivers/                  HAL/LL/CMSIS — never edit
Middlewares/              FreeRTOS, USB, FatFS — configured via .ioc
App/                      ← YOUR code: app.c/.h, drivers for sensors, protocol, state machines
project.ioc               ← hardware configuration, committed
```

`main.c`:
```c
/* USER CODE BEGIN Includes */
#include "app.h"
/* USER CODE END Includes */
...
  /* USER CODE BEGIN 2 */
  app_init();
  /* USER CODE END 2 */

  while (1)
  {
    /* USER CODE END WHILE */      // (generated marker)
    /* USER CODE BEGIN 3 */
    app_loop();
  }
  /* USER CODE END 3 */
```
Add `App/` to the include paths and source folders (CubeIDE: Project → Properties → C/C++ General → Paths and Symbols; CMake: `target_sources`/`target_include_directories` in the user section of `CMakeLists.txt`, outside `cmake/stm32cubemx/`).

To change the HardFault handler or other IRQ handlers yourself, untick "Generate IRQ handler" for that interrupt in the `.ioc` (NVIC → Code generation) and implement it in your own file.

## 2. C++ in a CubeMX project

- Keep `main.c` as C. Create `App/app.cpp` with:
  ```cpp
  extern "C" void app_init(void) { /* construct drivers, start tasks */ }
  extern "C" void app_loop(void) { /* superloop body */ }
  ```
  and `app.h` with `#ifdef __cplusplus extern "C" { #endif ... #ifdef __cplusplus } #endif`.
- CubeIDE: convert the project to C++ (right-click → Convert to C++) so `.cpp` files are compiled with g++.
- Flags: `-fno-exceptions -fno-rtti -fno-threadsafe-statics -fno-use-cxa-atexit`.
- HAL callbacks are C functions with weak linkage: define them in a `.c` file or as `extern "C"` in C++, otherwise name mangling means your callback is never called.

## 3. UART and printf

**Receive variable-length frames with DMA + idle line** (F0/F1 HAL versions from ~2021+ and all recent families have it):
```c
static uint8_t s_rx[128];

void uart_start_rx(void) {
    if (HAL_UARTEx_ReceiveToIdle_DMA(&huart2, s_rx, sizeof s_rx) == HAL_OK) {
        __HAL_DMA_DISABLE_IT(huart2.hdmarx, DMA_IT_HT);   /* no half-transfer callback */
    }
}

void HAL_UARTEx_RxEventCallback(UART_HandleTypeDef *huart, uint16_t size) {
    if (huart == &huart2) {
        rx_frame_push(s_rx, size);   /* copy into a queue/ring buffer: ISR context! */
        uart_start_rx();             /* re-arm */
    }
}

void HAL_UART_ErrorCallback(UART_HandleTypeDef *huart) {
    if (huart == &huart2) uart_start_rx();   /* overrun/noise/framing stop reception */
}
```
Enable the USART global interrupt and the DMA channel in the `.ioc`; DMA mode Normal for this pattern (Circular needs position tracking).

**printf over UART** (blocking — fine for debug, not for hot paths). CubeIDE's `syscalls.c` calls `__io_putchar`:
```c
int __io_putchar(int ch) {
    uint8_t c = (uint8_t)ch;
    HAL_UART_Transmit(&huart2, &c, 1, 10);
    return ch;
}
```
On Nucleo-64 boards the ST-Link virtual COM port is on PA2/PA3, but the peripheral varies: USART2 on many (F4, L4...), **LPUART1 on G4 (NUCLEO-G431RB/G474RE)** and others — check the board user manual and the `.ioc` before writing printf code. `%f` needs `-u _printf_float` with newlib-nano (costs flash). For production logging, use a ring buffer drained by DMA.

## 4. I2C / SPI

- I2C address is the 7-bit address shifted: `addr7 << 1`.
- Use `HAL_I2C_Mem_Read/Write` for register-based sensors.
- `HAL_I2C_IsDeviceReady(&hi2c1, addr7 << 1, 3, 10)` to probe a device during bring-up.
- Bus stuck (`HAL_BUSY` forever, SDA low): de-init I2C, set SCL as GPIO open-drain, toggle 9 clock pulses, generate STOP, re-init. Also check external pull-ups (internal ones are too weak for 400 kHz).
- SPI chip select: drive CS as a plain GPIO around the transfer unless the peripheral's hardware NSS behaviour is exactly what the device needs.
- After `HAL_SPI_TransmitReceive_DMA`, wait for the completion callback before touching the buffer.

## 5. ADC with DMA

- Continuous conversions + DMA circular into `uint16_t adc_buf[N_CH * N_SAMPLES]`; process halves in `HAL_ADC_ConvHalfCpltCallback` / `HAL_ADC_ConvCpltCallback` (signal a task; do not do heavy math in the callback).
- Run calibration before starting where the family supports it. The signature differs per family — check `stm32xx_hal_adc_ex.h`:
  - G4/L4/U5: `HAL_ADCEx_Calibration_Start(&hadc1, ADC_SINGLE_ENDED)`
  - H7: `HAL_ADCEx_Calibration_Start(&hadc1, ADC_CALIB_OFFSET, ADC_SINGLE_ENDED)` (or `ADC_CALIB_OFFSET_LINEARITY`)
  - F1/G0/C0: `HAL_ADCEx_Calibration_Start(&hadc1)`
  - F4: no calibration function
- Sampling time must suit the source impedance (RM table); high-impedance dividers need longer sampling or a buffer capacitor.
- Convert with `__HAL_ADC_CALC_DATA_TO_VOLTAGE` / VREFINT for accuracy instead of assuming 3.3 V.

## 6. Timers, PWM, input capture, EXTI

- PWM: set PSC/ARR in the `.ioc`, start with `HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_1)`, change duty with `__HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, ccr)`. f_pwm = f_timer / ((PSC+1)(ARR+1)); remember the APB timer clock is ×2 when the APB prescaler ≠ 1.
- Advanced timers (TIM1/TIM8) also need `HAL_TIMEx_PWMN_Start` for complementary outputs and the MOE bit (HAL sets it on start).
- Periodic tick: `HAL_TIM_Base_Start_IT` + `HAL_TIM_PeriodElapsedCallback` — note CubeMX already uses that callback for the HAL timebase timer when SYS timebase ≠ SysTick; check `htim->Instance`.
- EXTI: `HAL_GPIO_EXTI_Callback(uint16_t pin)` (G0/U5/H5 split into `_Rising_Callback`/`_Falling_Callback`). Debounce in software, not with delays in the callback.

## 7. HAL vs LL

| HAL | LL (low-layer) |
|---|---|
| Portable across families, handles state, timeouts, DMA plumbing | Thin inline register wrappers, small and fast |
| Bigger, some overhead in ISRs | You handle sequencing and errors |
| Default choice | Hot paths: fast ISRs, high-rate GPIO, tight loops |

Mixing is fine: CubeMX lets you choose HAL or LL per peripheral (Project Manager → Advanced Settings).

## 8. Low power

- Sleep: `HAL_PWR_EnterSLEEPMode(PWR_MAINREGULATOR_ON, PWR_SLEEPENTRY_WFI)`; SysTick wakes it every 1 ms — call `HAL_SuspendTick()` first if you want to sleep longer.
- Stop: clocks stop, RAM kept. **After waking, the system clock is HSI/MSI** — call `SystemClock_Config()` again before using peripherals.
- Standby/Shutdown: RAM lost (except backup domain), wake = reset; read `PWR` flags at boot to tell a wake-up from a cold start.
- Debugging in low power: the debugger disconnects in Stop unless `HAL_DBGMCU_EnableDBGStopMode()` — which raises consumption, so remove for current measurements.
- Set unused GPIOs to analog mode and disable unused peripheral clocks to reach datasheet currents.

## 9. Flash writes and EEPROM emulation

- Sequence: `HAL_FLASH_Unlock()` → erase page/sector (`HAL_FLASHEx_Erase`) → program in the family's unit (half-word F1, word F4, double-word G0/G4/L4, 256-bit flash word H7) → `HAL_FLASH_Lock()`.
- Erase granularity: pages (1–4 KB) on most; large sectors (16–128 KB) on F4/F7 — reserve a small sector at the start of the bank for data.
- CPU stalls while erasing the bank it executes from; time-critical interrupts can be delayed by milliseconds.
- Use ST's EEPROM emulation (X-CUBE-EEPROM) or a simple wear-levelled log with version + CRC; never rewrite the same page on every change.
- Reserve the data area in the linker script so code can't grow into it.

## 10. Watchdog and reset cause

```c
void log_reset_cause(void) {
    if      (__HAL_RCC_GET_FLAG(RCC_FLAG_IWDGRST)) log_w("reset: independent watchdog");
    else if (__HAL_RCC_GET_FLAG(RCC_FLAG_WWDGRST)) log_w("reset: window watchdog");
    else if (__HAL_RCC_GET_FLAG(RCC_FLAG_SFTRST))  log_i("reset: software");
    else if (__HAL_RCC_GET_FLAG(RCC_FLAG_LPWRRST)) log_w("reset: low-power");
    else if (__HAL_RCC_GET_FLAG(RCC_FLAG_PORRST))  log_i("reset: power-on");   /* name varies: PORRST/BORRST per family */
    else if (__HAL_RCC_GET_FLAG(RCC_FLAG_PINRST))  log_i("reset: NRST pin");
    __HAL_RCC_CLEAR_RESET_FLAGS();
}
```
Check flag names in the family's `stm32xx_hal_rcc.h`. The PIN flag is set along with most other causes (reset goes through NRST), so test it last.

- IWDG cannot be stopped once started; freeze it while halted in the debugger with `__HAL_DBGMCU_FREEZE_IWDG()` (debug builds only).
- Refresh (`HAL_IWDG_Refresh`) from the main loop or a supervisor task, never from a timer ISR.

## 11. FreeRTOS in CubeMX

- Middleware → FREERTOS (CMSIS-RTOS v2 wrapper). Set SYS timebase to TIM6/TIM7.
- Stack sizes in CubeMX are in **words** (×4 bytes). Check with `uxTaskGetStackHighWaterMark()` (words) and enable `configCHECK_FOR_STACK_OVERFLOW = 2` + `vApplicationStackOverflowHook` during development.
- Interrupts that call `...FromISR` APIs must have a priority numerically ≥ `configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY` (5 by default in CubeMX); otherwise an assert or random corruption.
- Enable `configASSERT` in development; define `USE_NEWLIB_REENTRANT` if tasks use printf/malloc.
- Heap: heap_4 is the usual choice; prefer static allocation (`xTaskCreateStatic`) for fixed tasks.
