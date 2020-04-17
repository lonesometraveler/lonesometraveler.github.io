---
layout: post
title: "GPIO Interrupt: Let Me Know When You Need Me"
date: 2020-04-17 8:16:21 -0400
categories: Rust embedded
---

After [the successful timer interrupt experiments](https://lonesometraveler.github.io/2020/04/04/timer-interrupt.html), I worked on GPIO interrupts this week. 

For STM32 microcontrollers, we use EXTI (external interrupt controller) and do these to configure GPIO interrupts:

* Enable peripheral clocks.
* Configure an input pin and set the edge detection.
* Unmask the interrupt mask register.

Once properly configured, an interrupt request is generated when the selected edge occurs on the external interrupt line.

In this post, I will show 3 example programs that demonstrate GPIO interrupt. 

1. GPIO interrupt with one button
2. GPIO interrupt with two buttons and how to tell which one triggers an interrupt
3. Interrupt with two buttons in a different way 

The goal is to control LEDs when buttons are pressed. I used Nucleo-F429ZI and [stm32f4xx-hal](https://github.com/stm32-rs/stm32f4xx-hal) for all three of them. 

## Example 1: Interrupt with One Button

### Enable peripheral clocks

First, we need to turn on `SYSCFG` (System Configuration) peripheral.

```rust
let mut dp = stm32::Peripherals::take().unwrap();
dp.RCC.apb2enr.write(|w| w.syscfgen().enabled());
```

### Configure a LED and a button

I need to access my button and LED in both ISR and user contexts. So, I wrap them with `Mutex`. Note that the initial values are `None` here. I will replace them with initialized instances later.

```rust
static BUTTON: Mutex<RefCell<Option<PC13<Input<PullDown>>>>> = Mutex::new(RefCell::new(None));
static LED: Mutex<RefCell<Option<PB7<Output<PushPull>>>>> = Mutex::new(RefCell::new(None));
```

Nucleo-F429ZI's pin `pc13` is connected to the user button. After setting it up as a pull down input, I set the edge detection to `RISING`.  

```rust
let gpioc = dp.GPIOC.split();
let mut user_button = gpioc.pc13.into_pull_down_input();
user_button.make_interrupt_source(&mut dp.SYSCFG);
user_button.enable_interrupt(&mut dp.EXTI);
user_button.trigger_on_edge(&mut dp.EXTI, Edge::RISING);
```

I also set up my LED.

```rust
let gpiob = dp.GPIOB.split();
let led = gpiob.pb7.into_push_pull_output();
```

I now move my initialized button and LED to Mutex. Moving the shared resources to Mutex makes it possible to access the LED and the button in both ISR and user contexts. I use critical section with `cortex_m::interrupt::free` and replace the shared resources’ content (`None`) with `user_button` and `led`.

```rust
free(|cs| {
    BUTTON.borrow(cs).replace(Some(user_button));
    LED.borrow(cs).replace(Some(led));
});
```

### Unmask and enable interrupt

Interrupt can be enabled by unmasking NVIC's register. 

```rust
unsafe {
    stm32::NVIC::unmask(stm32::interrupt::EXTI15_10);
}
```

`EXTI15_10` means interrupt lines from 10 - 15. Whenever an interrupt is triggered on these lines (my button `pc13` is on line 13), EXTI calls this handler. 

```rust
#[interrupt]
fn EXTI15_10() {
    free(|cs| {
        if let Some(ref mut btn) = BUTTON.borrow(cs).borrow_mut().deref_mut() {
            btn.clear_interrupt_pending_bit();

            if let Some(ref mut led) = LED.borrow(cs).borrow_mut().deref_mut() {
                led.toggle().unwrap();
            }
        }
    });
}
```

Here, I clear the interrupt flag by calling `btn.clear_interrupt_pending_bit()` and toggle the LED. (If I don’t clear the flag, it keeps calling the interrupt handler.)

 - [Example 1 full code](https://github.com/lonesometraveler/stm32f4xx-examples/blob/master/examples/gpio_interrupt_1.rs)

## Example 2: Interrupt with Two Buttons

Ok, that was pretty easy. But, like I said, `EXTI15_10` means interrupt requests can be generated from any of six lines. In the example above, we know it is `BUTTON` that triggers an interrupt because there are no other buttons. But, what if we have two buttons and want to call different functions depending on which button is pressed? My second example handles interrupt requests from two buttons.

### Configure resources

This time, I declare two sets of button/LED.

```rust
static BUTTON: Mutex<RefCell<Option<PC13<Input<PullDown>>>>> = Mutex::new(RefCell::new(None));
static LED: Mutex<RefCell<Option<PB7<Output<PushPull>>>>> = Mutex::new(RefCell::new(None));
static ANOTHER_BUTTON: Mutex<RefCell<Option<PC10<Input<PullUp>>>>> = Mutex::new(RefCell::new(None));
static ANOTHER_LED: Mutex<RefCell<Option<PB14<Output<PushPull>>>>> = Mutex::new(RefCell::new(None));
```

I also declare a Mutex for the external interrupt controller.

```rust
static EXTI: Mutex<RefCell<Option<stm32::EXTI>>> = Mutex::new(RefCell::new(None));
```

Just like the first example, I configure buttons and LEDs. I then wrap my resources with `Mutex`. This time, I wrap an instance of the external interrupt controller as well. I need `EXTI` to determine where an interrupt request is generated in the ISR.

```rust
let exti = dp.EXTI; // External Interrupt Controller

free(|cs| {
    BUTTON.borrow(cs).replace(Some(user_button));
    LED.borrow(cs).replace(Some(led));
    ANOTHER_BUTTON.borrow(cs).replace(Some(another_button));
    ANOTHER_LED.borrow(cs).replace(Some(another_led));
    EXTI.borrow(cs).replace(Some(exti));
});
```

### Find out where the interrupt comes from

I now have two buttons: one on line 10 and the other on line 13. I want to find out which button is pressed and call a corresponding callback function. Here are my two callback functions.

```rust
fn user_button_cb() {
    free(|cs| {
        if let Some(ref mut btn) = BUTTON.borrow(cs).borrow_mut().deref_mut() {
            // Clear the interrupt flag
            btn.clear_interrupt_pending_bit();

            if let Some(ref mut led) = LED.borrow(cs).borrow_mut().deref_mut() {
                led.toggle().unwrap();
            }
        }
    });
}

fn another_button_cb() {
    free(|cs| {
        if let Some(ref mut btn) = ANOTHER_BUTTON.borrow(cs).borrow_mut().deref_mut() {
            // Clear the interrupt flag
            btn.clear_interrupt_pending_bit();

            if let Some(ref mut led) = ANOTHER_LED.borrow(cs).borrow_mut().deref_mut() {
                led.toggle().unwrap();
            }
        }
    });
}
```

And this is how my interrupt handler tells which button is pressed.

```rust
#[interrupt]
fn EXTI15_10() {
    free(|cs| {
        if let Some(ref mut exti) = EXTI.borrow(cs).borrow_mut().deref_mut() {
            let pr = exti.pr.read();
            // Interrupt on line 13?
            if pr.pr13().bit_is_set() {
                user_button_cb();
            }
            // Interrupt on line 10?
            if pr.pr10().bit_is_set() {
                another_button_cb();
            }
        }
    });
}
```

When an interrupt is triggered, the pending bit corresponding to the interrupt line is set. In the interrupt handler, I evaluate the external interrupt controller’s PR register and call a corresponding callback.

- [Example 2 full code](https://github.com/lonesometraveler/stm32f4xx-examples/blob/master/examples/gpio_interrupt_2.rs)

## Example 3: Another Way of Two Buttons

Example 2 works fine. But, if I touch `EXTI`’s PR register, I can easily clear interrupt flags by directly writing to it. As we see in example 2, when an interrupt is triggered, the pending bit corresponding to the interrupt line is set. This request can be reset by writing a `1` in the pending register. Actually, `stm32f4xx-hal`’s `clear_interrupt_pending_bit()` method does exactly that:

```rust
fn clear_interrupt_pending_bit(&mut self) {
    unsafe { (*EXTI::ptr()).pr.write(|w| w.bits(1 << self.i) ) };
}
```

I am accessing EXTI anyway. Why don’t I clear the flag by directly writing to the PR register?

```rust
#[interrupt]
fn EXTI15_10() {
    free(|cs| {
        if let Some(ref mut exti) = EXTI.borrow(cs).borrow_mut().deref_mut() {
            let pr = exti.pr.read();
            // Interrupt on line 13?
            if pr.pr13().bit_is_set() {
                // Clear the interrupt flag
                exti.pr.write(|w| w.pr13().set_bit());
                user_button_cb();
            }
            // Interrupt on line 10?
            if pr.pr10().bit_is_set() {
                // Clear the interrupt flag
                exti.pr.write(|w| w.pr10().set_bit());
                another_button_cb();
            }
        }
    });
}
```

Since `exti.pr.write(|w| w.pr13().set_bit());` clears the interrupt flag, I no longer need to call my button’s `clear_interrupt_pending_bit()` in my callbacks.

```rust
fn user_button_cb() {
    free(|cs| {
        if let Some(ref mut led) = LED.borrow(cs).borrow_mut().deref_mut() {
            led.toggle().unwrap();
        }
    });
}

fn another_button_cb() {
    free(|cs| {
        if let Some(ref mut led) = ANOTHER_LED.borrow(cs).borrow_mut().deref_mut() {
            led.toggle().unwrap();
        }
    });
}
```

If I do it this way, I don’t even need Mutex for my buttons any more. Once I configure the buttons and enable the interrupt, they generate interrupt requests and I can access them through EXTI. I don’t need my buttons in the interrupt handler. So, my shared resources are now just LEDs and EXTI.

```rust
static LED: Mutex<RefCell<Option<PB7<Output<PushPull>>>>> = Mutex::new(RefCell::new(None));
static ANOTHER_LED: Mutex<RefCell<Option<PB14<Output<PushPull>>>>> = Mutex::new(RefCell::new(None));
static EXTI: Mutex<RefCell<Option<stm32::EXTI>>> = Mutex::new(RefCell::new(None));
```

 - [Example 3 full code](https://github.com/lonesometraveler/stm32f4xx-examples/blob/master/examples/gpio_interrupt_3.rs)