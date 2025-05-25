---
title: "Zephyr on ESP32: Blinking an LED using Overlay"
description: Learn how to configure and blink an LED using Zephyr RTOS on ESP32 with Device Tree and Overlay.
author: samuelh
date: 2025-02-01 11:24:00 -0300
categories: [Zephyr, ESP32, LED]
tags: [esp32, zephyr, rtos]
math: true
media_subpath: "/assets/img/posts/2025-02-01-blink-led-with-overlay-and-dts/"
image:
  path: cover.png
  lqip: data:image/png;base64,
# permalink: /post/hello-zephyr/
# toc: true
---

In this post, we will learn how to configure and blink an LED using Zephyr RTOS on ESP32 with Device Tree and overlay. Based on the [previous post]({{ site.baseurl }}/posts/hello-world-and-led-blink/), we will use the same setup and add the LED blinking functionality. So, if you haven't read the previous post, I recommend you do it before continuing.

## What is Overlay in Zephyr? {#what-is-overlay}

An overlay is a way to extend the default configuration of a Zephyr project. It allows you to add or modify configuration options without changing the original configuration files. This is useful when you want to add or change configurations without modifying the original files.

## What is Device Tree in Zephyr? {#what-is-dts}

The Device Tree is a data structure that describes the hardware components of a system. It provides a way to describe the hardware in a generic and reusable way. The Device Tree is used by the Zephyr kernel to configure the hardware at runtime.

## Configure the project {#configure-the-project}

You can simply copy the previous project and add the LED blinking functionality. So, let's start by copying the previous project to a new directory:

```bash
cp -r blink_led_pt1 blink_led_pt2
cd blink_led_pt2
```
{: .nolineno}

Now, open the `CMakeLists.txt` file and update it with the following content:

```cmake
cmake_minimum_required(VERSION 3.20.0)

set(BOARD esp32_devkitc_wroom)

# Define paths for board-specific configurations
set(BOARD_DIR ${CMAKE_CURRENT_SOURCE_DIR}/boards)
set(DTC_OVERLAY_FILE ${BOARD_DIR}/esp32_devkitc_wroom.overlay)
set(EXTRA_CONF_FILE ${BOARD_DIR}/esp32_devkitc_wroom.conf)

# Locate and configure Zephyr
find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})

project(blink_led_pt2)

# Add the main application source file
target_sources(app PRIVATE src/main.c)

# Display important configuration details during compilation
message(STATUS "Board: ${BOARD}")
message(STATUS "Devicetree Overlay: ${DTC_OVERLAY_FILE}")
message(STATUS "Extra Config File: ${EXTRA_CONF_FILE}")
```
{: file='blink_led_pt2/CMakeLists.txt'}

We added the followings compile variables:

- `project(blink_led_pt2)`: Changes the project name to `blink_led_pt2`.
- `BOARD_DIR`: Specifies the directory containing board-specific files.
- `DTC_OVERLAY_FILE`: Defines the path to the Device Tree overlay file, which configures the GPIO pin for the LED.
- `EXTRA_CONF_FILE`: Points to an additional configuration file, which enables necessary features for the ESP32 board.
- `message()`: Outputs the configured values to the console, helping with debugging and verification.

### Create the Device Tree Overlay {#create-the-device-tree-overlay}

Next, create the `boards` directory and add the `esp32_devkitc_wroom.overlay` file:

```bash
mkdir boards
touch boards/esp32_devkitc_wroom.overlay
```
{: .nolineno}

Now, open the `esp32_devkitc_wroom.overlay` file and add the following content:

```c
/ {
  leds {
    compatible = "gpio-leds"; // This is the driver that will be used to control the LED
    blinking_led: blinking_led {  // This is the name of the LED
      gpios = <&gpio0 25 GPIO_ACTIVE_HIGH>; // The GPIO pin that the LED is connected to
      label = "LED_0"; // This is the label that will be used to control the LED
    };
  };
};
``` 
{: file='blink_led_pt2/boards/esp32_devkitc_wroom.overlay'}

This file defines the LED configuration using the `zephyr/dts/bindings/led/gpio-leds.yaml`{: .filepath} binding. The `leds` node groups LED-related properties:

- `compatible`: Specifies the `gpio-leds` driver, which enables LED control via GPIO.
- `blinking_led`: A node representing the LED instance.
- `gpios`: Defines the GPIO controller and pin number (GPIO 25 in this case) along with its active state (`GPIO_ACTIVE_HIGH`).
- `label`: Provides a symbolic name (`LED_0`) to reference the LED in the application.

By defining this overlay, Zephyr will recognize the LED as a controllable GPIO device, allowing it to be manipulated in the firmware code.

### Create the Extra Configuration File {#create-the-extra-configuration-file}

Next, create the `esp32_devkitc_wroom.conf` file inside the `boards` directory:

```bash
touch boards/esp32_devkitc_wroom.conf
```
{: .nolineno}

Open the `esp32_devkitc_wroom.conf` file and add the following configuration:

```conf
CONFIG_ESP32_USE_UNSUPPORTED_REVISION=y # Required for ESP32-D0WDQ6 (revision v1.0)
```
{: file='blink_led_pt2/boards/esp32_devkitc_wroom.conf'}

This configuration file is specific to the `esp32_devkitc_wroom` board and allows additional settings to be applied during the Zephyr build process.

The file is included in the build process through the `EXTRA_CONF_FILE` variable, which is defined in the CMakeLists.txt:

```cmake
set(EXTRA_CONF_FILE ${BOARD_DIR}/esp32_devkitc_wroom.conf)
```
{: file='blink_led_pt2/CMakeLists.txt' .nolineno}

This ensures that Zephyr applies the configurations specified in `esp32_devkitc_wroom.conf`{: .filepath} during compilation.

- `CONFIG_ESP32_USE_UNSUPPORTED_REVISION=y:` This option is necessary for ESP32 chips of revision v1.0 (ESP32-D0WDQ6).
  - Some early ESP32 revisions, such as D0WDQ6 v1.0, have known hardware issues that cause frequent resets when running Zephyr.
  - Enabling this setting ensures that Zephyr properly handles these revisions, preventing unexpected behavior like constant reboots.
  - If you are using a newer revision of the ESP32, this setting may not be necessary.

By defining this extra configuration, Zephyr will apply the necessary workarounds for older ESP32 revisions, ensuring a stable runtime environment.

### Update the `prj.conf` File {#update-the-prj-conf-file}

Now, the `blink_led_pt2/prj.conf`{: .filepath} file will contain only general project configurations, such as enabling logging and GPIO support:

```conf
CONFIG_PRINTK=y # Required for printk
CONFIG_GPIO=y # Required for GPIO
```
{: file='blink_led_pt2/prj.conf'}

With this setup, the project has basic logging capabilities and GPIO support enabled, ensuring the application can interact with the hardware properly.

## Write the LED Blinking Code {#write-the-led-blinking-code}

Now, update the `src/main.c` file to include the code for blinking the LED:

```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/gpio.h>

static const struct gpio_dt_spec led = GPIO_DT_SPEC_GET(DT_NODELABEL(blinking_led), gpios);

int main(void) {
  if (!device_is_ready(led.port)) {
    printk("GPIO device is not ready\r\n");
    return 0;
  }

  // int ret = gpio_pin_configure(led.port, led.pin, led.dt_flags);
  int ret = gpio_pin_configure_dt(&led, GPIO_OUTPUT_ACTIVE);
  if (ret != 0) {
    printk("Failed to configure GPIO pin\r\n");
    return ret;
  }

  while (true) {
    printk("Hello, World!\r\n");
    ret = gpio_pin_toggle_dt(&led);
    if (ret != 0) {
      printk("Failed to set GPIO pin\r\n");
      return ret;
    }
    k_msleep(1000);
  }
}
```
{: file='blink_led_pt2/src/main.c'}

With **Device Tree Overlay**, the LED configuration is abstracted from the application logic, making the code more maintainable and hardware-independent. The key addition to `main.c` is:

```c
static const struct gpio_dt_spec led = GPIO_DT_SPEC_GET(DT_NODELABEL(blinking_led), gpios);
```
{: .nolineno}

- The `DT_NODELABEL(blinking_led)` macro retrieves the node labeled `blinking_led` from the Device Tree overlay (`esp32_devkitc_wroom.overlay`{: .filepath}).
- The `GPIO_DT_SPEC_GET()` macro retrieves the GPIO specification for the LED node, including the GPIO controller, pin number, and active state.

### Configure the LED pin {#configure-the-led-pin}

The LED pin is configured using the `gpio_pin_configure_dt()` function, which simplifies the configuration process by accepting the `gpio_dt_spec` structure:

```c
// int ret = gpio_pin_configure(led.port, led.pin, led.dt_flags);
int ret = gpio_pin_configure_dt(&led, GPIO_OUTPUT_ACTIVE);
if (ret != 0) {
  printk("Failed to configure GPIO pin\r\n");
  return ret;
}
```
{: .nolineno}

You can use the `gpio_pin_configure()` function (commented out), but you need to pass the atributtes of the `gpio_dt_spec` structure.

### The main loop {#the-main-loop}

The LED is toggled using the `gpio_pin_toggle_dt()` function, which simplifies the pin state change process:

```c
while (true) {
  printk("Hello, World!\r\n");
  ret = gpio_pin_toggle_dt(&led);
  if (ret != 0) {
    printk("Failed to set GPIO pin\r\n");
    return ret;
  }
  k_msleep(1000);
}
```
{: .nolineno}

The LED state is toggled every second, providing a blinking effect. The `gpio_pin_toggle_dt()` function abstracts the pin state change, simplifying the code and making it more readable.

## Build and flash the project {#build-and-flash-the-project}

If you copied the project from the previous post, you may need to clean the build directory before building the new project:

```bash
rm -rf build
```
{: .nolineno}

Now, build the project using the following command:

```bash
west build -b esp32_devkitc_wroom/esp32/procpu -p always
```
{: .nolineno}

- `-b esp32_devkitc_wroom/esp32/procpu`: Specifies the board and processor (procpu) for the build.
- `-p always`: Ensures a clean `build`{:.filepath} folder is generated every time.

After building the project, flash it to the ESP32 board:

```bash
west flash
```
{: .nolineno}

The LED should start blinking once the flashing process is complete. If you encounter any issues, check the console output for error messages and verify the configurations in the overlay and configuration files.

Verify the output using the Espressif Monitor:

```bash
west espressif monitor
```
{: .nolineno}

## Conclusion {#conclusion}

In this post, we learned how to configure and blink an LED using Zephyr RTOS on ESP32 with Device Tree and overlay. By using the Device Tree overlay, we abstracted the LED configuration from the application logic, making the code more maintainable and hardware-independent. We also used an extra configuration file to enable necessary features for the ESP32 board, ensuring a stable runtime environment.

The entire code of this project can be found [here](https://github.com/samuelhrqe/zephyr-on-esp32/tree/main/blink_led_pt2).

## References {#references}

- [Zephyr Project Documentation][zephyr_docs]{:target="_blank"}
- [Device Tree in Zephyr][dt_in_zephyr]{:target="_blank"}
- [Blinky][blinky]{:target="_blank"}
- [ESP32 on Zephyr OS: "Hello, world!" (Blinking LED)][esp32_on_zephyr_yt]{:target="_blank"}

[zephyr_docs]: https://docs.zephyrproject.org/latest/
[dt_in_zephyr]: https://docs.zephyrproject.org/latest/guides/dts/index.html
[blinky]: https://docs.zephyrproject.org/latest/samples/basic/blinky/README.html#blinky
[esp32_on_zephyr_yt]: https://www.youtube.com/watch?v=Rk8p3OyW5A8&list=PLEQVp_6G_y4iFfemAbFsKw6tsGABarTwp&index=2
