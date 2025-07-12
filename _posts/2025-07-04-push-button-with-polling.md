---
title: "Zephyr on ESP32: Push Button with Polling"
description: Learn how to use a push button with polling in Zephyr RTOS on ESP32 using Devicetree overlays.
author: samuelh
date: 2025-07-06 17:00:00 -0300
categories: [Zephyr, ESP32, Button, LED]
tags: [esp32, zephyr, rtos]
math: true
media_subpath: "/assets/img/posts/2025-07-04-push-button-with-polling/"
image:
  path: cover.png
  lqip: data:image/png;base64,
# permalink: /post/hello-zephyr/
# toc: true
---

This is the third tutorial in a series about Zephyr on the ESP32. Here, you'll learn how to use a push button with polling mode. We'll cover the basics of working with buttons in Zephyr RTOS.

## Components Used and Circuit Diagram {#circuit-diagram}

For this tutorial, you will need the following components:

- **ESP32**
- **Push Button**
- **LED**
- **Resistor ($220\Omega$)**
- **Breadboard**
- **Jumper Wires**

Connect the components as shown in the circuit diagram below:

![ESP32 button circuit diagram (light mode)](esp32_button.png){: .light width="450" }
![ESP32 button circuit diagram (dark mode)](esp32_button_dark.png){: .dark width="450" }

In the circuit diagram above:

- The **LED** is connected to `GPIO25` of the ESP32 through a $220\Omega$ resistor.
- The **push button** is connected to `GPIO27` and **GND**. A pull-up resistor is not used here because the internal pull-up will be enabled through the Device Tree overlay.

## Create a New Zephyr Project {#create-new-project}

Let's create a new Zephyr project for this tutorial. Follow the steps below:

### 1. Create a Project Directory

Start by creating a new directory for your project. You can name it `button_polling` or use any other name you prefer. Open a terminal and run:

```bash
mkdir -p button_polling/src button_polling/boards
cd button_polling
```
{: .nolineno}

### 2. Create Required Files

Now, create the essential files for your Zephyr application:

```bash
touch src/main.c proj.conf CMakeLists.txt \
boards/esp32_devkitc_wroom.conf boards/esp32_devkitc_wroom.overlay
```
{: .nolineno}

Here's a brief description of each file:

- `src/main.c`: The main source file where the application logic goes.
- `proj.conf`: Defines general project configurations (e.g., enabling GPIO or logging).
- `CMakeLists.txt`: Configures the build system for Zephyr.
- `boards/esp32_devkitc_wroom.conf`: Additional board-specific configurations.
- `boards/esp32_devkitc_wroom.overlay`: Device Tree overlay for hardware configuration (e.g., LED and button).

### 3. Write the `esp32_devkitc_wroom.overlay` File

In this file, we will define the GPIO pins used for the push button and LED. Open `boards/esp32_devkitc_wroom.overlay` and add the following content:

```c
/ {
  aliases {
    my-led = &blinking_led;
    my-button = &push_button;
  };

  leds {
    compatible = "gpio-leds";
    blinking_led: d25 {
      gpios = <&gpio0 25 GPIO_ACTIVE_HIGH>;
      label = "LED_0";
      status = "okay";
    };
  };

  buttons {
    compatible = "gpio-keys";
    polling-mode;

    push_button: d27 {
      gpios = <&gpio0 27 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>;
      label = "BUTTON_27";
      status = "okay";
    };
  };
};
```
{: file='button_polling/boards/esp32_devkitc_wroom.overlay'}

This overlay file defines three main sections:

- **aliases**: Creates alias names to simplify access to the LED and button nodes in the application.
- **leds**: Declares the LED hardware and how it is connected.
- **buttons**: Declares the push button and its configuration.

#### 3.1 LED configuration

TThe LED configuration follows the `zephyr/dts/bindings/led/gpio-leds.yaml`{: .filepath} binding. The LED node includes the following properties:

- `compatible`: Specifies the `gpio-leds` driver, which enables LED control via GPIO.
- `blinking_led`: Label used for referencing the LED node.
- `d25`: Node name, indicating GPIO pin 25.
- `gpios`: Defines the GPIO controller, pin number (`25`) and active logic level (`GPIO_ACTIVE_HIGH`).
- `label`: Logical name (`LED_0`) for use in the application code.
- `status`: Enables the node by setting it to `okay`.

> While `label` and `status` are optional, they help with identification and node activation.
{: .prompt-tip }

#### 3.2 Button configuration

The button is configured according to the `zephyr/dts/bindings/input/gpio-keys.yaml`{: .filepath} binding. The push button node includes:

- `compatible`: Uses the `gpio-keys` driver to handle buttons via GPIO.
- `polling-mode`: Instructs Zephyr to read the button state through polling instead of interrupts.
- `push_button`: Label used for referencing this node.
- `d27`: Node name, indicating GPIO pin 27.
- `gpios`: Specifies the GPIO controller, pin number (27), and flags:
  - `GPIO_ACTIVE_LOW`: The button is active when pressed (pulls the line low).
  - `GPIO_PULL_UP`: Enables internal pull-up resistor.
- `label`: Logical name (`BUTTON_27`) for use in the application code.
- `status`: Enables the node by setting it to `okay`.

> The `polling-mode` property is essential for enabling button state checking without interrupts.
{: .prompt-tip }

### 4. Write the `esp32_devkitc_wroom.conf` File

In this file, we will define board-specific configuration options.

In my case, I am using an old **ESP-WROOM-32** module, which may cause constant reboots when running Zephyr. To avoid this, we will enable the `CONFIG_ESP32_USE_UNSUPPORTED_REVISION` option.

Open `boards/esp32_devkitc_wroom.conf` and add the following content:

```conf
CONFIG_ESP32_USE_UNSUPPORTED_REVISION=y
```
{: file='button_polling/boards/esp32_devkitc_wroom.conf'}

If you are using a newer ESP32 board, you can skip this step.

### 5. Write the `src/main.c` File

Edit the `src/main.c` file to add the code for polling the push button and controlling the LED:

```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/gpio.h>

static const int32_t sleep_time_ms = 100;
static const struct gpio_dt_spec button = GPIO_DT_SPEC_GET(DT_ALIAS(my_button), gpios);
static const struct gpio_dt_spec led = GPIO_DT_SPEC_GET(DT_ALIAS(my_led), gpios);

int main(void) {

  int ret, state;

  if (!gpio_is_ready_dt(&button)) {
    printk("Button device is not ready\r\n");
    return 0;
  }

  if (!gpio_is_ready_dt(&led)) {
    printk("LED device is not ready\r\n");
    return 0;
  }

  ret = gpio_pin_configure_dt(&button, GPIO_INPUT);
  if (ret != 0) {
    printk("Failed to configure button GPIO pin\r\n");
    return 0;
  }

  ret = gpio_pin_configure_dt(&led, GPIO_OUTPUT);
  if (ret != 0) {
    printk("Failed to configure LED GPIO pin\r\n");
    return 0;
  }
  
  printk("Button spec flags: 0x%x\r\n", button.dt_flags);
  printk("LED spec flags: 0x%x\r\n", led.dt_flags);

  while (true) {

    state = gpio_pin_get_dt(&button);
    if (state < 0) {
      printk("Error %d: failed to read button state\r\n", state);
    } else {
      printk("Button state: %d\r\n", state);
    }

    ret = gpio_pin_set_dt(&led, state);
    if (ret < 0) {
      printk("Error %d: failed to set LED state\r\n", ret);
    }
    else {
      printk("LED state set to: %d\r\n", state);
    }

    k_msleep(sleep_time_ms);
  }

  return 0;
}
```
{: file='button_polling/src/main.c'}

#### 5.1 Get the Button and LED Devices

To access the button and LED devices defined in the Device Tree, we use the `GPIO_DT_SPEC_GET` macro. This macro retrieves the GPIO device specification based on the aliases defined in the overlay file:

```c
static const struct gpio_dt_spec button = GPIO_DT_SPEC_GET(DT_ALIAS(my_button), gpios);
static const struct gpio_dt_spec led = GPIO_DT_SPEC_GET(DT_ALIAS(my_led), gpios);
```
{: .nolineno}

- `button`: Holds the GPIO specification for the push button.
- `led`: Holds the GPIO specification for the LED.

> The alias `my_button` and `my_led` are defined in the overlay file, allowing us to reference the button and LED easily in the code.
{: .prompt-tip }

Before using these devices, we check if they are ready in the `main()` function using the `gpio_is_ready_dt` function:

```c
if (!gpio_is_ready_dt(&button)) {
  printk("Button device is not ready\r\n");
  return 0;
  }

if (!gpio_is_ready_dt(&led)) {
  printk("LED device is not ready\r\n");
  return 0;
}
```
{: .nolineno}

#### 5.2 Configure the Button and LED GPIO Pins

Next, configure the GPIO pins using the `gpio_pin_configure_dt` function:

```c
ret = gpio_pin_configure_dt(&button, GPIO_INPUT);
if (ret != 0) {
  printk("Failed to configure button GPIO pin\r\n");
  return 0;
}

ret = gpio_pin_configure_dt(&led, GPIO_OUTPUT);
if (ret != 0) {
  printk("Failed to configure LED GPIO pin\r\n");
  return 0;
}
```
{: .nolineno}

- The button is configured as an input.
- The LED is configured as an output.

If configuration fails, an error message is printed and the program exits.

#### 5.3 The Main Loop

In the main loop, we continuously read the button state update the LED accordingly:

```c
while (true) {

  state = gpio_pin_get_dt(&button);
  if (state < 0) {
    printk("Error %d: failed to read button state\r\n", state);
  } else {
    printk("Button state: %d\r\n", state);
  }

  ret = gpio_pin_set_dt(&led, state);
  if (ret < 0) {
    printk("Error %d: failed to set LED state\r\n", ret);
  }
  else {
    printk("LED state set to: %d\r\n", state);
  }

  k_msleep(sleep_time_ms);
}
```
{: .nolineno}

The button state is read using `gpio_pin_get_dt`, which retrieves the logical state of the button as interpreted by Zephyr.

Since the button is connected with one leg to **GND** and the other to **GPIO27**, and we enabled the `GPIO_PULL_UP` and `GPIO_ACTIVE_LOW` flags in the Device Tree, the electrical behavior is as follows:

- When the **button is not pressed**, the GPIO pin is pulled **HIGH** (logical 1) via the internal pull-up resistor.
- When the **button is pressed**, the circuit closes and connects the pin directly to **GND**, pulling it **LOW** (logical 0).

The `GPIO_ACTIVE_LOW` flag tells Zephyr to treat a **LOW voltage level (0)** as the **active state** of the button. As a result:

- `gpio_pin_get_dt()` returns **1** when the button is **pressed**.
- It returns **0** when the button is **released**.

The LED is updated using `gpio_pin_set_dt`, setting its state to match the button state:  

- When the button is pressed, the LED turns on.
- When the button is released, the LED turns off.

`k_msleep()` adds a short delay (e.g., 100 ms or 500 ms), preventing the loop from reading the button and updating the LED too frequently, which could flood the console with messages and use unnecessary CPU time.

### 6. Writing the `CMakeLists.txt` File

Finally, we need to create the `CMakeLists.txt` file to configure the build system. Open `CMakeLists.txt` and add the following content:

```cmake
cmake_minimum_required(VERSION 3.20.0)

set(BOARD_DIR ${CMAKE_CURRENT_SOURCE_DIR}/boards)
set(BOARD esp32_devkitc_wroom)
set(DTC_OVERLAY_FILE ${BOARD_DIR}/esp32_devkitc_wroom.overlay)
set(EXTRA_CONF_FILE ${BOARD_DIR}/esp32_devkitc_wroom.conf)

find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})

project(button_polling)

target_sources(app PRIVATE src/main.c)
```
{: file='button_polling/CMakeLists.txt'}

This configuration sets the board name, the location of the overlay and configuration files, and includes the main source file of your application.

## Build and Flash the Project {#build-and-flash}

From the root directory of your project, run the following command to build the firmware:

```bash
west build -b esp32_devkitc_wroom/esp32/procpu -p always
```
{: .nolineno}

After the build completes successfully, flash the firmware to your ESP32 board using:

```bash
west flash
```
{: .nolineno}

Once flashed, you can monitor the serial output using the Espressif Monitor:

```bash
west espressif monitor
```
{: .nolineno}
> To exit the Espressif Monitor, press <kbd>Ctrl + ]</kbd>
{: .prompt-tip}

## Conclusion {#conclusion}

In this post, we learned how to configure and use a push button in polling mode with Zephyr RTOS on the ESP32. We created a simple project that reads the state of a push button and controls an LED accordingly.

This example demonstrates the basics of working with GPIO devices in Zephyr, including Device Tree overlays, internal pull-ups, and GPIO abstraction through the API.

The entire code for this tutorial is available [here](https://github.com/samuelhrqe/zephyr-on-esp32/tree/main/button_polling). You can use this as a starting point for more complex projects involving buttons and other GPIO devices.

The video below provides a visual demonstration of the project:

{%
  include embed/video.html
  src='push_button.mp4'
  types='mp4'
  title='Demonstration of Push Button with Polling in Zephyr on ESP32'
  autoplay=false
  loop=true
  muted=true
%}

## References {#references}

- [Introduction to Zephyr Part 4: Devicetree Tutorial \| DigiKey][esp32_on_zephyr_yt]{:target="_blank"}
- [Zephyr Project Documentation][zephyr_docs]{:target="_blank"}
- [Button - Zephyr Docs][button]{:target="_blank"}
- [Device Tree in Zephyr][dt_in_zephyr]{:target="_blank"}
- [Practical Zephyr - Devicetree basics (Part 3)][practical_zephyr_dt]{:target="_blank"}

[zephyr_docs]: https://docs.zephyrproject.org/latest/
[dt_in_zephyr]: https://docs.zephyrproject.org/latest/guides/dts/index.html
[button]: https://docs.zephyrproject.org/latest/samples/basic/button/README.html
[esp32_on_zephyr_yt]: https://youtu.be/fOMJyjwowNk?si=rymuxnaIRnDH1FK-
[practical_zephyr_dt]: https://interrupt.memfault.com/blog/practical_zephyr_dt
