# Real-Time Security System using FreeRTOS & STM32

A practical demonstration of **Priority-Based Preemptive Scheduling** using FreeRTOS on an STM32 Nucleo-F401RE microcontroller. The project simulates a smart-home priority automation layout where a high-priority safety sensor event instantly overrides a low-priority background lighting task.

---

## System Architecture & Concept

This project relies on the core principles of a Real-Time Operating System (RTOS). Instead of utilizing sequential execution or blocking delay routines (`HAL_Delay`), the application splits processing cycles into two independent, priority-ranked tasks:

### 1. `LightingTask` (Normal Priority)
* **Function:** Simulates standard household ambient lighting.
* **Action:** Toggles a Green LED every 500ms.
* **State:** Runs continuously under normal conditions, yielding CPU execution to other threads during its idle duration via non-blocking delay vectors.

### 2. `SecurityTask` (High Priority)
* **Function:** Simulates an emergency monitoring array (such as an intruder sensor or fire alarm).
* **Action:** Polls the state of the built-in user button.
* **Preemption Mechanism:** When the button is triggered, the FreeRTOS kernel instantly suspends (`preempts`) the lower-priority `LightingTask` to execute the security routine. The Green LED is forced low, and the Red LED remains active for 2 full seconds. 

---

##  Hardware Requirements & Pin Mapping

### Component List
* 1x STM32 Nucleo-F401RE Development Board
* 1x Solderless Breadboard
* 1x Green LED (Normal Operation indicator)
* 1x Red LED (Alarm indicator)
* 2x $280\Omega$ (or similar up to $330\Omega$) Resistors
* Jumper Wires

### Pin Connection Diagram
Connect the components according to the structural map below:

| Peripheral Device | STM32 Microcontroller Pin | Header Designation |
| :--- | :--- | :--- |
| **Green LED (Anode)** | `PA5` | Arduino Header **D13** |
| **Red LED (Anode)** | `PA6` | Arduino Header **D12** |
| **Built-in Blue Button** | `PC13` | On-board Hardware Connection |
| **Common Ground Rail** | `GND` | Any On-board **GND** Pin |

---

## Configuration Profile (STM32CubeMX)

The hardware peripheral initialization abstract layer was configured using the following core criteria:

* **System Core -> SYS:** Set **Timebase Source** to `TIM1`. *(Crucial step to release `SysTick` exclusively for FreeRTOS scheduler context switching).*
* **Middleware -> FREERTOS:** Interface set to **CMSIS_V2**.
* **GPIO Configurations:** * `PA5` and `PA6` configured as `GPIO_Output` (Push-Pull, No Pull-up/Pull-down).
  * `PC13` configured as `GPIO_Input`.
* **Task Definitions:**
  * `defaultTask`: Priority set to `osPriorityNormal`, Entry Function: `StartDefaultTask`.
  * `myTask02`: Priority set to `osPriorityHigh`, Entry Function: `StartTask02`.

---

##  Source Code

The multitasking loops located in `main.c` utilize the following architecture:

```c
/* Task 1: Low-Priority Lighting Automation Loop */
void StartDefaultTask(void *argument)
{
  for(;;)
  {
    HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5); // Toggle Green LED (PA5)
    osDelay(500);                          // Relinquish control for 500ms
  }
}

/* Task 2: High-Priority Emergency Preemption Loop */
void StartTask02(void *argument)
{
  for(;;)
  {
    // Evaluate if Built-in Blue Button (PC13) is depressed
    if(HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13) == GPIO_PIN_SET)
    {
      /* --- EMERGENCY MODE ACTIVATED --- */
      HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_RESET); // Terminate background lighting
      HAL_GPIO_WritePin(GPIOA, GPIO_PIN_6, GPIO_PIN_SET);   // Assert Alarm indicator (Red LED)
      
      osDelay(2000); // Maintain locked emergency state for 2000ms
    }
    else
    {
      /* --- SAFE MODE --- */
      HAL_GPIO_WritePin(GPIOA, GPIO_PIN_6, GPIO_PIN_RESET); // De-assert Alarm indicator
      
      osDelay(10); // Relinquish CPU cycles briefly to prevent low-priority starvation
    }
  }
}
