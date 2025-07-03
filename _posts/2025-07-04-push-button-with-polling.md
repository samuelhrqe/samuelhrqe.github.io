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

```bash
mkdir -p button_polling/src
cd button_polling/src
```
{: .nolineno}

## References {#references}

- [Zephyr Project Documentation][zephyr_docs]{:target="_blank"}
- [Device Tree in Zephyr][dt_in_zephyr]{:target="_blank"}
- [Blinky][blinky]{:target="_blank"}
- [ESP32 on Zephyr OS: "Hello, world!" (Blinking LED)][esp32_on_zephyr_yt]{:target="_blank"}

[zephyr_docs]: https://docs.zephyrproject.org/latest/
[dt_in_zephyr]: https://docs.zephyrproject.org/latest/guides/dts/index.html
[blinky]: https://docs.zephyrproject.org/latest/samples/basic/blinky/README.html#blinky
[esp32_on_zephyr_yt]: https://www.youtube.com/watch?v=Rk8p3OyW5A8&list=PLEQVp_6G_y4iFfemAbFsKw6tsGABarTwp&index=2
