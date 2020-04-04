---
layout: post
title: "Timer Interrupt: In the Blink of an Eye"
date: 2020-04-04 07:07:21 -0400
categories: Rust embedded
---

As my daily routine was interrupted by COVID-19, I stayed home and played around with embedded Rust’s interrupt this week.

Interrupt can be dangerous if you don’t do it right. I read [the concurrency section of the book](https://docs.rust-embedded.org/book/concurrency/index.html) and learned Rust's safe way to share resources between the main and ISR contexts.

## How to use timer interrupt
In this post, I will show how I did a blinking LED application using a timer interrupt. 

![]({{ site.url }}/assets/blinking_led.gif)

> "It's amazing in the blink of an eye, you finally see the light." - Steven Tyler

### 1. Wrap shared resources

The goal is to flip a desired state of a LED in an interrupt handler and turn on/off the LED accordingly in the main. So, it is necessary to share the state of LED and a timer peripheral between ISR and the main code.

In embedded Rust, we use statically allocated `Mutex` to share peripherals and variables between different contexts.

Here is how I wrap `bool` that represents the state of my LED. 

```rust
static LED_STATE: Mutex<Cell<bool>> = Mutex::new(Cell::new(false));
```

And a line below shows how I wrap my `TIM2`, timer 2 module. Note that `TIMER_TIM2`’s `RefCell` has an initial value of `None`. It will be replaced with a timer peripheral later.

```rust
static TIMER_TIM2: Mutex<RefCell<Option<Timer<stm32::TIM2>>>> = Mutex::new(RefCell::new(None));
```

Next, I set up an interrupt timer. I am setting it up as a 5Hz timer and wrapping it with `Mutex`. 

```rust
let mut timer = Timer::tim2(dp.TIM2, 5.hz(), clocks, &mut rcc.apb1);
timer.listen(Event::Update);
```

I then use [critical section](https://docs.rust-embedded.org/book/concurrency/#critical-sections) with `cortex_m::interrupt::free` and replace `TIMER_TIM2`’s content (`None`) with my instantiated `timer`. 

```rust
free(|cs| {
    TIMER_TIM2.borrow(cs).replace(Some(timer));
});
```

### 2. Handle interrupts
Enabling the `TIM2` interrupt is very easy with NVIC (Nested vector interrupt control).

```rust
stm32::NVIC::unpend(Interrupt::TIM2);
unsafe {
    stm32::NVIC::unmask(Interrupt::TIM2);
}
```

That’s it. The timer is now running. When the interrupt is triggered, it goes to the interrupt handler defined like this:

```rust
#[interrupt]
fn TIM2() {
    free(|cs| {
        if let Some(ref mut tim2) = TIMER_TIM2.borrow(cs).borrow_mut().deref_mut() {
            tim2.clear_update_interrupt_flag();
        }
        let led_state = LED_STATE.borrow(cs);
        led_state.replace(!led_state.get());
    });
}
```

The first thing to do in the ISR is to clear the interrupt flag. `stm32f3xx_hal` has `clear_update_interrupt_flag()`. This method clears the status register’s UIF bit. This bit is set by the hardware and must be cleared by the software. If this bit is not cleared, the program repeats the ISR forever.
After clearing the flag, I flip the state of `LED_STATE` and replace the old value with a new one. 

### 3. Reflect the change in the main
In the main, I evaluate `LED_STATE` like this:

```rust
loop {
    if free(|cs| LED_STATE.borrow(cs).get()) {
        led.set_high().unwrap();
    } else {
        led.set_low().unwrap();
    }
}
```

See how I use critical section by calling `cortex_m::interrupt::free` to access `LED_STATE`. This and `Mutex` ensures an exclusive access to the shared resource.

## Code: Blink Blink
Here are two codes for blinking LED. One for [STM32F3DISCOVERY board](https://www.st.com/en/evaluation-tools/stm32f3discovery.html) and the other for [Nucleo-F429ZI](https://www.st.com/en/evaluation-tools/nucleo-f429zi.html). I used `stm32f3xx_hal` for DISCOVERY and `stm32f4xx_hal` for Nucleo. The basic concept is the same between the two implementations. Only differences are the way to instantiate the timer peripheral and the call to clear the interrupt flag by using HAL specific methods.

### STM32F3DISCOVERY

Here is the code for [STM32F3DISCOVERY board](https://www.st.com/en/evaluation-tools/stm32f3discovery.html) board. This is built with `stm32f3xx_hal`.

- [GitHub repo](https://github.com/lonesometraveler/stm32f3xx-examples)

```rust
#![no_main]
#![no_std]

extern crate panic_halt;

use core::cell::{Cell, RefCell};
use core::ops::DerefMut;
use cortex_m;
use cortex_m::interrupt::{free, Mutex};
use cortex_m_rt::entry;
use stm32f3xx_hal::{
    prelude::*,
    stm32,
    stm32::{interrupt, Interrupt},
    timer::{Event, Timer},
};

static LED_STATE: Mutex<Cell<bool>> = Mutex::new(Cell::new(false));
static TIMER_TIM2: Mutex<RefCell<Option<Timer<stm32::TIM2>>>> = Mutex::new(RefCell::new(None));

#[interrupt]
fn TIM2() {
    free(|cs| {
        if let Some(ref mut tim2) = TIMER_TIM2.borrow(cs).borrow_mut().deref_mut() {
            tim2.clear_update_interrupt_flag();
        }
        let led_state = LED_STATE.borrow(cs);
        led_state.replace(!led_state.get());
    });
}

#[entry]
fn main() -> ! {
    let dp = stm32::Peripherals::take().unwrap();
    // Set up the system clock
    let mut flash = dp.FLASH.constrain();
    let mut rcc = dp.RCC.constrain();
    let clocks = rcc.cfgr.freeze(&mut flash.acr);

    // Set up the LED
    let mut gpioe = dp.GPIOE.split(&mut rcc.ahb);
    let mut led = gpioe
        .pe9
        .into_push_pull_output(&mut gpioe.moder, &mut gpioe.otyper);

    // Set up the interrupt timer
    let mut timer = Timer::tim2(dp.TIM2, 5.hz(), clocks, &mut rcc.apb1);
    timer.listen(Event::Update);

    // Move shared resources to Mutex
    free(|cs| {
        TIMER_TIM2.borrow(cs).replace(Some(timer));
    });

    // Enable interrupt
    stm32::NVIC::unpend(Interrupt::TIM2);
    unsafe {
        stm32::NVIC::unmask(Interrupt::TIM2);
    }

    loop {
        if free(|cs| LED_STATE.borrow(cs).get()) {
            led.set_high().unwrap();
        } else {
            led.set_low().unwrap();
        }
    }
}

```

### Nucleo-F429ZI

[Nucleo-F429ZI board](https://www.st.com/en/evaluation-tools/nucleo-f429zi.html) has STM32F429 microcontroller. I use `stm32f4xx_hal` for this.

- [GitHub repo](https://github.com/lonesometraveler/stm32f4xx-examples)

```rust
#![no_main]
#![no_std]

extern crate panic_halt;

use core::cell::{Cell, RefCell};
use core::ops::DerefMut;
use cortex_m;
use cortex_m::interrupt::{free, Mutex};
use cortex_m_rt::entry;
use stm32f4xx_hal::{
    prelude::*,
    stm32,
    stm32::interrupt,
    timer::{Event, Timer},
};

static LED_STATE: Mutex<Cell<bool>> = Mutex::new(Cell::new(false));
static TIMER_TIM2: Mutex<RefCell<Option<Timer<stm32::TIM2>>>> = Mutex::new(RefCell::new(None));

#[interrupt]
fn TIM2() {
    free(|cs| {
        if let Some(ref mut tim2) = TIMER_TIM2.borrow(cs).borrow_mut().deref_mut() {
            tim2.clear_interrupt(Event::TimeOut);
        }
        let led_state = LED_STATE.borrow(cs);
        led_state.replace(!led_state.get());
    });
}

#[entry]
fn main() -> ! {
    let dp = stm32::Peripherals::take().unwrap();
    // Set up the LED
    let gpiob = dp.GPIOB.split();
    let mut led = gpiob.pb7.into_push_pull_output();

    // Set up the system clock
    let rcc = dp.RCC.constrain();
    let clocks = rcc.cfgr.sysclk(48.mhz()).freeze();

    // Set up the interrupt timer
    let mut timer = Timer::tim2(dp.TIM2, 5.hz(), clocks);
    timer.listen(Event::TimeOut);

    free(|cs| {
        TIMER_TIM2.borrow(cs).replace(Some(timer));
    });

    // Enable interrupt
    stm32::NVIC::unpend(stm32::Interrupt::TIM2);
    unsafe {
        stm32::NVIC::unmask(stm32::Interrupt::TIM2);
    }

    loop {
        if free(|cs| LED_STATE.borrow(cs).get()) {
            led.set_high().unwrap();
        } else {
            led.set_low().unwrap();
        }
    }
}
```
