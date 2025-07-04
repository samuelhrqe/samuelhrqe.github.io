---
title: "Zephyr on ESP32: Push Button with Polling"
description: Learn how to configure and push button with polling using Zephyr RTOS on ESP32 with Device Tree and overlay.
author: samuelh
date: 2025-07-04 23:10:00 -0300
categories: [Zephyr, ESP32, Button]
tags: [esp32, zephyr, rtos]
math: true
media_subpath: "/assets/img/posts/2025-07-04-push-button-with-polling/"
image:
  path: cover.png
  lqip: data:image/png;base64,
# permalink: /post/hello-zephyr/
# toc: true
---

In this post, you will learn how to configure and use a push button with polling using Zephyr RTOS on ESP32. This is the third tutorial in a series about using Zephyr with the ESP32. In this tutorial, we’ll cover the basics of working with push buttons in Zephyr.

## Components Used and Circuit Diagram {#circuit-diagram}

For this tutorial, you need the following component:

- **ESP32**
- **Push Button**
- **LED**
- **Resistor ($220\Omega$)**
- **Breadboard and Jumper Wires**

Connect the components as shown in the circuit diagram below:

![ESP32 button circuit diagram (light mode)](esp32_button.png){: .light width="450" }
![ESP32 button circuit diagram (dark mode)](esp32_button_dark.png){: .dark width="450" }

The circuit diagram for this tutorial is simple. You will need a push button connected to `GPIO 27` of the ESP32. The other side of the button is connected to ground (GND). When the button is pressed, it will connect `GPIO 27` to ground, allowing us to detect the button press.

## Create a New Zephyr Project {#create-new-project}

Let's create a new Zephyr project for this tutorial. Follow these steps:

### 1. Create a Project Directory

First, create a new directory for your project. You can name it `button_polling` or any name you prefer. Open a terminal and run the following commands:

```bash
mkdir -p button_polling/src button_polling/boards
cd button_polling
```
{: .nolineno}

### 2. Create Required Files

Create the following files in the root of your project directory:

```bash
touch src/main.c proj.conf CMakeLists.txt \
boards/esp32_devkitc_wroom.conf boards/esp32_devkitc_wroom.overlay
```
{: .nolineno}

- `src/main.c`: The main source file.
- `proj.conf`: Defines project settings.
- `CMakeLists.txt`: Configures the build system.
- `boards/esp32_devkitc_wroom.conf`: Board configuration file.
- `boards/esp32_devkitc_wroom.overlay`: Device Tree overlay file.

### 3. Write the `esp32_devkitc_wroom.overlay` File

In this file, we will define the GPIO pins for the push button and LED. Open `boards/esp32_devkitc_wroom.overlay` and add the following content:

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

This overlay file has three main sections:

- **aliases**: Defines aliases for the LED and button.
- **leds**: Configures the LED.
- **buttons**: Configures the push button.

#### 3.1 LED configuration

The LED configuration can be found in the `zephyr/dts/bindings/led/gpio-leds.yaml`{: .filepath} binding file. The LED is defined with the following properties:

- `compatible`: Specifies the `gpio-leds` driver, which enables LED control via GPIO.
- `blinking_led`: A label for the LED node.
- `d25`: The node name for the LED, indicating it is connected to GPIO 25.
- `gpios`: Defines the GPIO controller, pin number (`GPIO25` in this case) and flags (`GPIO_ACTIVE_HIGH`).
- `label`: Provides a symbolic name (`LED_0`) to reference the LED in the application.
- `status`: Indicates that the LED is enabled (`okay`).

> `label` and `status` are optional properties, but they are useful for identification and status management.
{: .prompt-tip }

#### 3.2 Button configuration

The button configuration can be found in the `zephyr/dts/bindings/input/gpio-keys.yaml`{: .filepath} binding file. The button is defined with the following properties:

- `compatible`: Specifies the `gpio-keys` driver, which enables button control via GPIO.
- `polling-mode`: Enables polling mode for the button.
- `push_button`: A label for the button node.
- `d27`: The node name for the button, indicating it is connected to GPIO 27.
- `gpios`: Defines the GPIO controller, pin number (`GPIO27` in this case) and flags (`GPIO_ACTIVE_LOW | GPIO_PULL_UP`).
- `label`: Provides a symbolic name (`BUTTON_27`) to reference the button in the application.
- `status`: Indicates that the button is enabled (`okay`).

> The `polling-mode` property is important for enabling polling mode for the button, allowing us to check its state in the application.
{: .prompt-tip }

### 4. Write the `esp32_devkitc_wroom.conf` File

In this file, we will define the board configuration. In my case, I am using an old ESP-WROOM-32 board, so I will enable the `CONFIG_ESP32_USE_UNSUPPORTED_REVISION` option to prevent constant reboots. Open `boards/esp32_devkitc_wroom.conf` and add the following content:

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

To access the button and LED devices defined in the Device Tree, we use the `GPIO_DT_SPEC_GET` macro. This macro retrieves the GPIO device specification for the button and LED based on their aliases defined in the overlay file.

```c
static const struct gpio_dt_spec button = GPIO_DT_SPEC_GET(DT_ALIAS(my_button), gpios);
static const struct gpio_dt_spec led = GPIO_DT_SPEC_GET(DT_ALIAS(my_led), gpios);
```
{: .nolineno}

- `button`: Holds the GPIO specification for the push button.
- `led`: Holds the GPIO specification for the LED.

> The alias `my_button` and `my_led` are defined in the overlay file, allowing us to reference the button and LED easily in the code.
{: .prompt-tip }

Before using these devices, in the main function, we check if they are ready using the `gpio_is_ready_dt` function:

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

Next, we configure the GPIO pins for the button and LED using the `gpio_pin_configure_dt` function:

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

The button is configured as an input pin, and the LED is configured as an output pin. If the configuration fails, an error message is printed, and the program exits.

#### 5.3 The Main Loop

In the main loop, we continuously read the state of the button and set the LED state based on the button state:

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

The state of the button is read using `gpio_pin_get_dt`, which returns the current state of the button. If the button is pressed, it will return `1` (`GPIO_ACTIVE_LOW`), and if it is not pressed, it will return `0`. The LED state is set using `gpio_pin_set_dt`, which sets the LED to the same state as the button. If the button is pressed, the LED will turn on; if the button is not pressed, the LED will turn off.

The loop sleeps for a specified time (`sleep_time_ms`) to avoid excessive polling and printing.

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

## Build and Flash the Project {#build-and-flash}

In the root directory of your project, we will build the project using the following command:

```bash
west build -b esp32_devkitc_wroom/esp32/procpu -p always
```
{: .nolineno}

After the build is complete, you can flash the project to your ESP32 board using the following command:

```bash
west flash
```
{: .nolineno}

After flashing, you can monitor the output of your ESP32 board using the Espressif Monitor. This will allow you to see the printed messages from your application:

```bash
west espressif monitor
```
{: .nolineno}
> To exit the Espressif Monitor, press <kbd>Ctrl + ]</kbd>
{: .prompt-tip}

## Conclusion {#conclusion}

In this post, we learned how to configure and use a push button with polling using Zephyr RTOS on ESP32. We created a simple project that reads the state of a push button and controls an LED based on the button state. This is a fundamental example of how to work with GPIO devices in Zephyr.

The entire code for this tutorial is available [here](https://github.com/samuelhrqe/zephyr-on-esp32/tree/main/button_polling). You can use this as a starting point for more complex projects involving buttons and other GPIO devices.

## References {#references}

- [Zephyr Project Documentation][zephyr_docs]{:target="_blank"}
- [Device Tree in Zephyr][dt_in_zephyr]{:target="_blank"}
- [Blinky][blinky]{:target="_blank"}
- [ESP32 on Zephyr OS: "Hello, world!" (Blinking LED)][esp32_on_zephyr_yt]{:target="_blank"}

[zephyr_docs]: https://docs.zephyrproject.org/latest/
[dt_in_zephyr]: https://docs.zephyrproject.org/latest/guides/dts/index.html
[blinky]: https://docs.zephyrproject.org/latest/samples/basic/blinky/README.html#blinky
[esp32_on_zephyr_yt]: https://www.youtube.com/watch?v=Rk8p3OyW5A8&list=PLEQVp_6G_y4iFfemAbFsKw6tsGABarTwp&index=2
