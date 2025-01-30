---
title: "Zephyr on ESP32: Hello World and blink LED"
description: Learn how to run Zephyr on ESP32, print a message and blink an LED.
author: samuelh
date: 2025-01-30 11:24:00 -0300
categories: [Zephyr, ESP32, LED]
tags: [esp32, zephyr, rtos]
math: true
media_subpath: "/assets/img/posts/2024-01-30-hello-world-and-blink-led/"
image:
  path: cover.png
  lqip: data:image/png;base64,
# permalink: /post/hello-zephyr/
# toc: true
---

<!-- Getting started with Zephyr on ESP32 -->
<!-- ==================================== -->

In this article, you will learn how to run Zephyr on an ESP32, print a message, and blink an LED. This is the first tutorial in a series about using Zephyr with the ESP32. In this tutorial, we’ll cover the basics of working with ESP32 peripherals in Zephyr.

For all tutorials in this series, I’ll use the **ESP-WROOM-32** (38-pin version). However, you can use any ESP32 board you have.

## What is Zephyr? {#what-is-zephyr}

Zephyr is an open-source real-time operating system (RTOS) designed for resource-limited devices. It’s a Linux Foundation project built on a small kernel, suitable for devices like sensors, wearables, smartwatches, and IoT gateways. Zephyr supports multiple architectures and is ideal for embedded systems.

## Environment Setup {#environment-setup}

To run Zephyr on an ESP32, you need to install the Zephyr SDK. Follow the steps in the [official documentation](https://docs.zephyrproject.org/latest/develop/getting_started/index.html) to set it up.

> Use Python virtual environments to install the Zephyr SDK. This helps avoid conflicts with other Python packages.
{: .prompt-tip}

<details> <summary> My environment setup</summary>

  <table>
    <thead>
      <tr>
        <th>Component</th>
        <th>Version</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td>OS</td>
        <td>Ubuntu 22.04</td>
      </tr>
      <tr>
        <td>CMake version</td>
        <td>3.22.1</td>
      </tr>
      <tr>
        <td>Python version</td>
        <td>3.10.12</td>
      </tr>
      <tr>
        <td>Zephyr SDK version</td>
        <td>0.17.0</td>
      </tr>
      <tr>
        <td>Zephyr version</td>
        <td>4.0.99</td>
      </tr>
      <tr>
        <td>West version</td>
        <td>1.3.0</td>
      </tr>
      <tr>
        <td>DTC version</td>
        <td>1.6.0</td>
      </tr>
      <tr>
        <td>Python Environment</td>
        <td>Using <code>venv</code></td>
      </tr>
    </tbody>
  </table>

</details>

## Components Used and Circuit Diagram {#components}

For this tutorial, you need the following components:

- **ESP32**
- **LED**
- **Resistor ($220\Omega$)**
- **Jumper wires**
- **Breadboard** (optional)

### Circuit Diagram

Connect the components as shown in the circuit diagram:

![Circuit Diagram](led_esp32.png){: .light }
![Circuit Diagram](led_esp32_dark.png){: .dark }

The LED is connected to **GPIO pin 25** of the ESP32.

## Create a New Zephyr Project {#create-new-project}

Let's create a Zephyr project to blink an LED. Follow these steps:

### 1. Create a Project Folder

First, create a folder for the project. In this example, we'll name it `blink_led_pt1`:

```bash
mkdir -p blink_led_pt1/src
cd blink_led_pt1
```
{: .nolineno}

### 2. Create Required Files

Inside the `blink_led_pt1` folder, create the necessary files:

```bash
touch CMakeLists.txt prj.conf src/main.c
```
{: .nolineno}

- `CMakeLists.txt`: Configures the build system.
- `prj.conf`: Defines project settings.
- `src/main.c`: The main source file.

### 3. Configure `CMakeLists.txt`

Edit the `CMakeLists.txt` file with the following content:

```cmake
cmake_minimum_required(VERSION 3.20.0)

set(BOARD esp32_devkitc_wroom)

find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})

project(blink_led_pt1)

target_sources(app PRIVATE src/main.c)
```

### 4. Configure `prj.conf`

The `prj.conf` file is used to enable features like GPIO and logging. Add the following configuration:

```conf
CONFIG_ESP32_USE_UNSUPPORTED_REVISION=y # Required for ESP32 Rev. 1.0
CONFIG_PRINTK=y # Required for printk
CONFIG_GPIO=y # Required for GPIO
```

> If your ESP32 is an older version, enable `CONFIG_ESP32_USE_UNSUPPORTED_REVISION=y` to prevent constant reboots.
{: .prompt-tip}

### 5. Write Code in `src/main.c`

Edit the `src/main.c` file to add the code for blinking an LED:

```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/gpio.h>

#define LED_PIN 25
#define LOW 0
#define HIGH 1

static const struct device* gpio_ct_dev = DEVICE_DT_GET(DT_NODELABEL(gpio0));

int main(void) {
  if (!device_is_ready(gpio_ct_dev)) {
    printk("GPIO device is not ready\r\n");
    return 0;
  }

  int ret = gpio_pin_configure(gpio_ct_dev, LED_PIN, GPIO_OUTPUT_ACTIVE);
  if (ret != 0) {
    printk("Failed to configure GPIO pin\r\n");
    return ret;
  }

  while (true) {
    printk("Hello, World!\r\n");
    ret = gpio_pin_set_raw(gpio_ct_dev, LED_PIN, HIGH);
    if (ret != 0) {
      printk("Failed to set GPIO pin\r\n");
      return ret;
    }
    k_msleep(1000);

    ret = gpio_pin_set_raw(gpio_ct_dev, LED_PIN, LOW);
    if (ret != 0) {
      printk("Failed to set GPIO pin\r\n");
      return ret;
    }
    k_msleep(1000);
  }
}
```

#### 5.1 Get the GPIO Device

To blink the LED, we’ll use GPIO pin 25, controlled by the `gpio0` device. 

```c
static const struct device* gpio_ct_dev = DEVICE_DT_GET(DT_NODELABEL(gpio0));
```
{: .nolineno}

- `gpio_ct_dev`: Holds a reference to the `gpio0` device from the device tree.
- `DT_NODELABEL`: Gets the identifier for a node labeled in the device tree.
- `DEVICE_DT_GET`: Retrieves the device reference based on the identifier.

> You can check the ESP32 devicetree in the `zephyr/dts/xtensa/espressif/esp32/esp32_common.dtsi`{: .filepath} file.
{: .prompt-tip }

Before using the GPIO device, verify that it’s ready:

```c
if (!device_is_ready(gpio_ct_dev)) {
  printk("GPIO device is not ready\r\n");
  return 0;
}
```
{: .nolineno}

#### 5.2 Configure the GPIO Pin

To configure the GPIO pin, use the `gpio_pin_configure` function. This sets the pin mode and behavior. Here, we configure it as `GPIO_OUTPUT_ACTIVE`, meaning the pin is an output with active-high logic.

```c
int ret = gpio_pin_configure(gpio_ct_dev, LED_PIN, GPIO_OUTPUT_ACTIVE);
if (ret != 0) {
  printk("Failed to configure GPIO pin\r\n");
  return ret;
}
```
{: .nolineno}

Now the GPIO is ready to control the LED.

#### 5.3 Blink the LED

Finally, we will blink the LED with the following code:

```c
while (true) {
  printk("Hello, World!\r\n");
  ret = gpio_pin_set_raw(gpio_ct_dev, LED_PIN, HIGH);
  if (ret != 0) {
    printk("Failed to set GPIO pin\r\n");
    return ret;
  }
  k_msleep(1000);

  ret = gpio_pin_set_raw(gpio_ct_dev, LED_PIN, LOW);
  if (ret != 0) {
    printk("Failed to set GPIO pin\r\n");
    return ret;
  }
  k_msleep(1000);
}
```
{: .nolineno}

- `printk`: Prints a message to the console.
- `k_msleep`: Pauses the thread for a specified number of milliseconds. In this case, we are pausing for 1 second.
- `gpio_pin_set_raw`: Sets the GPIO pin to a specific value. We toggle the pin between `HIGH` and `LOW` to make the LED blink.

With this loop, the LED will turn on, stay on for 1 second, turn off, and repeat indefinitely.

The entire code of this project can be found [here](https://github.com/samuelhrqe/zephyr-on-esp32/tree/main/blink_led_pt1).

### 6. Build the project

When you installed the Zephyr SDK, the `west` tool was also installed. This tool is used to build, flash, and debug Zephyr projects. To build the project, run the following command:

```bash
west build -b esp32_devkitc_wroom/esp32/procpu -p always
```
{: .nolineno}

- `-b esp32_devkitc_wroom/esp32/procpu`: Specifies the board and processor (procpu) for the build.
- `-p always`: Ensures a clean `build`{:.filepath} folder is generated every time.

> The ESP32 has two processors: `procpu` and `appcpu`. The Zephyr differentiates the two processors in the build process. So, you need to set the processor that you want to build the project. In this case, we are building the project for the `procpu`.
{: .prompt-tip}

### 7. Flash the project

After building the project, you can flash the project to the ESP32. To do this, run the following command:

```bash
west flash
```
{: .nolineno}

This command will flash the project to the ESP32. After flashing, the LED will start blinking.

### 8. Check the output

To check the output of the project, you can use the Espressif Monitor. Run the following command:

```bash
west espressif monitor
```
{: .nolineno}

This command shows the output logs, including any messages printed with `printk`.

> To exit the Espressif Monitor, press <kbd>Ctrl + ]</kbd>.
{: .prompt-tip}

{%
  include embed/video.html
  src='led.mp4'
  types='mp4'
  title='Result video'
  autoplay=false
  loop=true
  muted=true
  
%}

## References {#references}

- [Getting Started Guide](https://docs.zephyrproject.org/latest/develop/getting_started/index.html)
- [ESP32-DevKitC-WROOM - Zephyr Documentation](https://docs.zephyrproject.org/latest/boards/espressif/esp32_devkitc_wroom/doc/index.html)
