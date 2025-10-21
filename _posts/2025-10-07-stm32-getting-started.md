---
layout: post
title: "Getting started with Embedded Systems"
description: "Dipping toes in the bare metal programming world"
comments: false
---

Being a backend engineer by day, mostly involved in web development, I tend to
explore different areas and domains at night. For many years I've been
attracted by low level systems engineering, and invariably found myself
periodically coming back to it; specifically C for the better part.

Curiously, as the time passes, I found myself wanting to dive deeper, going
closer and closer to the hardware, dipping my toes with networked applications
such as MQTT brokers, virtual machines, programming languages design, CPU
simulations and recently even game engine development. Something about removing
the magic, the burden of endless external dependencies that permeates the
higher level stacks, and being able to govern the behavior of the system to a
fine-grained detail, makes it all the more addicting. I'm far from good at it,
but nonetheless, I feel like I learn something new and exciting each and every
time I explore the depth of computer science and engineering at the foundation
levels.

The natural progression of this process led me to try my hands on bare metal,
developing on something without any real abstraction, not even an OS doing the
heavy work, only addresses, a bus to provide the clock to registers and
peripherals. Ideally, working my way up from there to an RTOS (Real Time
Operating System e.g. [FreeRTOS](https://www.freertos.org)) or something but
only after having clear how things work at the deepest level.

So what is an embedded system? Apparently the community is not really aligned
on the proper definition of it (so strange, usually everyone is so in sync
in this field..), but I like the definition provided by Elecia White in her
`Making Embedded Systems` book:

*An embedded system is a computerized system that is purpose-built for
its application.*

As the vast majority of this blog contents, this will largely be a learning
experience, gradually documenting the progress and discoveries along the
journey; so by definition, probably full of imprecisions and mistakes.

## Getting started with MCUs

There is a plethora of embedded systems, literally everywhere around us, from
the coffee machine, to the car pneumatics; the category I decided to focus on
pertains real-time systems (large chunk if the sector). I considered a couple
of microcontroller boards to begin with and read extensive literature on the
topic (including the aforementioned excellent `Making Embedded Systems` by
Elecia White, still in progress but so far, one of the best engineering books I
ever read).

There are a huge number of boards to start playing around with LEDs,
peripherals and other beginner-friendly projects, my shortlist reduced to:

- `Arduino`
- `ESP32`
- `STM32`

Eventually my choice fell on the last one, `STM32`, among those, without taking
anything from the other two, seems to be the most industry adopted and "real
world" board, comprised with extensive documentation and plenty of resources to
build a wide range of systems. I started on it by following an excellent
[Udemy
course](https://www.udemy.com/share/101XK43@QQJZUOvpUeB3jP2aBRXSCXpKEEiZwxo387JWk8jBSYngQN-2P5zYSLs6NNVifTy-/)
as introductory material, quickly adapting the workflow and dev environment
to better fit my liking.

The ST Microelectronics website provides an enormous amount of resources, docs,
data-sheets and manuals, including a full development environment and an IDE
(based on Eclipse, so not really my jam, but pretty powerful handy if you don't
know where to start).

## Dev setup

Most guides and tutorials suggest to begin with the ST Microelectronics
provided `STM32CubeIDE`, there are although, other ways to build and upload the
software to the board. I opted for `PlatformIO IDE`, which provides the entire
tool-chain to start developing from the more common editors, primarily targeting
`VSCode` but supporting also the likes of `Emacs` and `Vim`.

## Blinking a LED

The mandatory hello world of the embedded systems world, on bare-metal, a
synchronous infinite loop activating a LED at every clock cycle with a small
delay to allow seeing it blinking rather than just being perpetually on.

```c
// 1. Where is the LED connected?
//
// >>> From Nucleo user guide:
//     User LD2: the green LED is a user LED connected to ARDUINO® signal D13 corresponding
//               to STM32 I/O PA5 (pin 21) or PB13 (pin 34) depending on the STM32 target
//
// PA5 or PB13, 90% of the time is PA5
//
// Port: A
// Pin: 5

#include <stdint.h>

// 2. From the datasheet, locate the address space for peripherals

#define PERIPH_BASE       (0x40000000UL)
#define AHB1PERIPH_OFFSET (0x00020000UL)  // AHB1 is the bus, provide access to the clock to the peripheral
#define AHB1PERIPH_BASE   (PERIPH_BASE + AHB1PERIPH_OFFSET)
#define GPIOA_OFFSET      (0x0000UL)

// 2.a check for memory mapping

#define GPIOA_BASE        (AHB1PERIPH_BASE + GPIOA_OFFSET)

#define RCC_OFFSET        (0x3800UL)
#define RCC_BASE          (AHB1PERIPH_BASE + RCC_OFFSET)

// 3. From the reference manual, look for AHB1 bus registers

// ENR => (EN)able(R)egister
#define AHB1EN_R_OFFSET   (0x30UL)
#define RCC_AHB1EN_R      (*(volatile unsigned int *)(RCC_BASE + AHB1EN_R_OFFSET))

// Mode R (mode register)
#define MODE_R_OFFSET     (0x00UL)
#define GPIOA_MODE_R      (*(volatile unsigned int *)(GPIOA_BASE + MODE_R_OFFSET))

// Output data register
#define OD_R_OFFSET       (0x14UL)
#define GPIOA_OD_R        (*(volatile unsigned int *)(GPIOA_BASE + OD_R_OFFSET))

#define  GPIOAEN          (1U<<0) // 0b 0000 0000 0000 0000 0000 0000 0000 0001

#define PIN5              (1U<<5)
#define LED_PIN           PIN5

typedef struct {
    volatile uint32_t DUMMY[12];
    volatile uint32_t AHB1ENR;
} RCC_TypeDef;

typedef struct {
    volatile uint32_t MODER;
    volatile uint32_t DUMMY[4];
    volatile uint32_t ODR;
} GPIO_TypeDef;

/*
 * (1UL<<10) Set bit 10 to 1
 * &=~ (1UL<<11) Set bit 11 to 0
 */

#define RCC   ((RCC_TypeDef *)(RCC_BASE))
#define GPIOA ((GPIO_TypeDef *)(GPIOA_BASE))

int main(void)
{
    // 1. Enable clock access to GPIOA (using OR to prevent overwriting other possible set bits)
    RCC->AHB1ENR |= GPIOAEN;

    // 2. Set PA5 as output pin
    GPIOA->MODER |= (1U<<10);
    GPIOA->MODER &=~ (1U<<11);

    while (1) {
        // 3. Set PA5 high (turn on the LED)
        GPIOA->ODR |= LED_PIN;
        for (int i = 0; i < 100000; i++){}
    }

}
```

The snippet is part of my notes from the course, it is really basic and is set
in order to show how things work under the hood before jumping into real
development using the extremely vast array of headers and libraries mapping
peripherals and registers already available.

<span class="to-be-continued">`つづく`</span>
