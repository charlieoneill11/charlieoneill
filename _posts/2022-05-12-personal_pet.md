---
title: "Digital pet project with ARM Assembly"
description: "Using machine code to create a living pet"
layout: post
toc: true
comments: true
hide: false
search_exclude: true
categories: [arm, assembly, machine]
---

[Digital pets](https://en.wikipedia.org/wiki/Digital_pet) like the [Tamagotchi](https://en.wikipedia.org/wiki/Tamagotchi) or [Nintendogs](https://en.wikipedia.org/wiki/Nintendogs) are computer simulations of a friendly animal companion created as a toy or video game. These toys feature an animated depiction of the animal and inputs through button that allow the pet’s human owner to care for it or interct with it. Usually the owner gets to play with it, but also needs to provide food or entertainment in order for the pet to be happy.

Here, we're going to use ARM Assembly and a microbit to create a digital pet. The program will **display the pet** on the LED display, and allow a human to **interact** with the pet using the microbit’s buttons. The digital pet should provide companionship (that is, it’s **engaging** to look at) but also responsibility (that is, it **requires** interaction).

Your pet should have some kind of _state_ (e.g., it’s levels of health and happiness) that is stored in memory, and we will need to use _interrupts_ to receive input from the microbit’s buttons.

Here are our specifications for the program: 
-   **must** be written in **ARMv7 assembly**
-   **must** use the LED display to show a digital pet
-   **must** use (at least one) data structure in memory to store the “state” of the digital pet
-   **must** use interrupts to detect interactions with the microbit’s buttons
-   **must** work when the microbit is powered over USB but not connected to a computer (that is, it works after you upload it and plug into a USB charger)
-   **must** be _engaging_ and _require_ interaction for a fun experience over 1-3 minutes
-   _can_ use the speaker to create sound
-   _can_ use other inputs (e.g., microphone, IMU)
-   _can_ use **any** peripheral available on the microbit

Once we've accomplished this, we'll try and add a few extensions to make the pet more sophisticated:
- Improve the LED display to make it really stand out, e.g., by using pulse-width modulation (PWM) and smooth animations, catchy visuals and interesting effects, etc.
- Write a program that creates pets randomly and provides variation every time you play (e.g., a _generative_ program)
- Implement a random number generator and use it in your program
- Use the speaker to create an interesting synthesised sound to go with the digital pet.
- Use some kind of network connectivity (LEDs to light sensor, bluetooth, UART, etc) to send the pet to another microbit.
- Have the pet react to changes in environment (use the motion sensor or light sensors).
- Use non-volatile memory to store the digital pet on the microbit even when the power is disconnected.

# Using records: storing attributes in memory
```ARM
.data
attributes:
.word 5, 4, 3
```
# Home screen: displaying attributes
We need some way to show the user how their pet is going, in terms of both hunger and contentment, and also overall health. My initial thought was to use a single vertical bar for health, like so:
![[health_bar.jpg|300]]
Each LED in this bar represented a health point. I would then show the pet to the right of this as its own entity i.e. a head and a body. However, after some experimentation, I decided that this system would be too clunky. I simply didn't have enough screen space (that is, enough LEDs) for this system to be effective. Another problem with this approach is that it left me no room to display energy points. I would have had to implement a different peripheral (separate from the two buttons) if the user wanted to check how many energy points they had. 

Because of this, I decided to incorporate different attributes into the entity display of the pet itself. 

This still left the issue of how to display energy points. Because it was a bit of a challenge, I decided to see if I could use the touch sensor to trigger a screen which would display the energy points left. 

## Rendering function
So, now that we have the design for the pet egg, we need a way to display all components of the egg at once. We can't just have one function for rendering the egg. This is because different components of the egg rely on different attributes for how many LEDs to display. (I mean, I guess we could have one function, but it would get awfully messy.) Because of this, I'm going to write a function to display the health, hunger and happiness attributes separately. We'll then write a `render` wrapper function over these three parts of the egg, looping through them so quickly that they appear to be show at the same time. 

### Initial display

Here's the outline for my render function:
```ARM
.type render, %function
render:
	push {lr, r0-r4}
	mov r2, 0xff0 @ for-loop counter
	renderLoop:
		cmp r2, 0
		ble endRender
		push {r2}
		@ do the actual display
		bl displayHunger
		bl displayHappiness
		bl displayHealth 
		@ decrement, reenter loop
		pop {r2}
		sub r2, 1
		b renderLoop
	endRender:
		pop {lr, r0-r4}
		bx lr
```

All we're doing is using a simple for-loop to repeatedly 'blip' each attribute (represented as a different part of the egg) over and over, creating the illusion of a persistent image. Now, all we have to do is write the `display` functions. Health is the easiest, since that's just a vertical bar down the middle. Let's do that first:
```ARM
.type displayHealth, %function
displayHealth:
	push {lr}
	mov r2, 0x5
	healthLoop:
		cmp r2, 0
		ble endHealth
		push {r2}
		@ health bar
		mov r0, 0b11111
		mov r1, 0b11011
		bl frameBlip
		@ decrement, reenter loop
		pop {r2}
		sub r2, 1
		b healthLoop
	endHealth:
		mov r0, 0b00000
		mov r1, 0b11111 @ set all columns to high
		bl led_on @ clear LED screen
		pop {lr}
		bx lr
```
This is essentially the same structure as our `render` function above; we've just got another for-loop (an *inner* for-loop). The thing to note here is that we only go through this loop 5 times, in comparison to `0xff0` times for the render function. This is because we want to create persistence of vision; if we show each part of the egg for too long, the image won't be persistent, because our eyes will be able to tell that we're only displaying one component at a time. The `0x5` value was chosen through trial and error.

We now write essentially the same functions for the hunger and happiness attributes. And that's it! We've done it. (It's funny - it's quite difficult to actually take a photo to show you because the photo obviously only captures one aspect of the egg. At least this is proof that it's doing what we want it to do!)
![[egg.jpg|300]]

### Reflecting updated attributes in the display
However, we haven't finished our rendering function yet. We don't just want a static egg - we want our egg to reflect the current state of the pet's attributes in memory. This basically means that we're going to need to wrap the render loop itself in an outer main loop. This loop will display the pet for a short period of time, check its attributes, and display it again based on the updated attributes. This way, if an event occurs (e.g. we've fed the pet, played with the pet, the timer has decremented the pet's hunger, etc.) the state of the egg will be updated on the screen.

As we discussed above, if the pet's attributes decrement, we're going to turn off LEDs relating to that attribute in the display. This choice of display is probably a good one; it's easy to imagine the egg decaying or breaking due to neglect or overstimulation. The order for removing LEDs will be:
* *Hunger and happiness:* first, we'll take out the middle LED. Then, the top and bottom LEDs (in that order). Finally, the 2nd and 4th LEDs.
* *Health:* since health is a vertical bar, it's a bit easier. We'll take out the middle LED, then the 2nd and 4th LEDs, and finally the top and bottom LEDs. The bottom will be last because it represents the base of the egg. 

>[!error] Warning
>Remember, once the final health point attribute goes (represented by the bottom LED in the health bar), the egg dies. 

# Peripherals and interrupts
## Setting up the two buttons
We need to configure the two buttons, A and B, to trigger a feeding event and a playing event respectively. My ideas for the two events are this:
* *Feeding:* the user presses Button A to feed the egg. This uses up an energy point. A banana shows up on screen (representing that we are now feeding). The user then has to play a mini-game where they have to press Button B to open the cage, and Button A to drop in the food. The cage will only be open for a short period of time. If the player is successful in feeding, the hunger attribute goes up a level. If the egg is already full, the health will drop a point (as the egg is now overfed). Regardless of if the player is successful in feeding the egg or not, after the cage is shut, we return to the main screen where the egg is now displayed with the new attributes.
* *Playing:* the user presses Button B to play with the egg. This uses up an energy point. A smiling face shows up on screen (representing that we are now playing with the egg). The user then has to play a mini-game where they have to press Buttons A and B to match the LEDs on either the left or right of the screen, respectively. If the user presses the wrong button, the minigame ends, and we return to the main display. If the user gets through the minigame, the happiness attribute goes up a level, and we also return to the main screen. 

### Extension: using both buttons together
What could this do? Display how long we've been playing? 

## SysTick timer for hunger and happiness
If the user doesn't interact with the pet, we want it to get hungry and lonely. To measure when to do this in absolute time (rather than CPU cycles) we'll use the SysTick timer. 

## Configuring the touch sensor
We're going to use the touch tensor so that the user can see how many energy points they have left. Engaging with the touch sensor will briefly take them to a separate screen where the letters "E P" will flash up (standing for energy points), followed by a number from 0 to 5. Obviously, we're also going to store the energy points in memory. 

## Adding sounds
To make this even more interesting, we want to add sounds that accompany various events in the game. 

## Putting the pet to sleep: using the light sensor
We want this to be even more engaging. What if the egg could go into hibernation/a form of sleep? If the egg's environment grows too dark, it could sleep. All attributes would be frozen and a sleeping "ZZZ" could move around on the screen to represent the hibernation.

# Endgame
No game would be complete without an objective. Since this project is only meant to be engaging over 2-3 minutes, I thought it would be fun to make the outcome of the game binary over this time period: either you keep the pet alive, or it dies. If it stays alive, we can display some sort of evolution animation of the pet breaking out of its shell.

# Improving the code
## Refactoring display functions using memory
In my previous ARM programs, I've been criticised for not utilising memory to its full extent. And this is understandable; storing things in memory makes it so much easier to deal.

## Adding some simple animations
Because it's quite boring to stare at an egg for 2-3 minutes, especially if you're not using the buttons or peripherals, we want to add some animations that will spice things up a bit. 
* *Rotating egg:* when a random event is triggered, we can rotate the egg before going to a new screen.
* *Egg filling up:* when the player has "topped up" the egg (i.e. the egg was previously suboptimal in either happiness or hunger, and is now Level 5 in both), the egg will fill up using an animation. Once it is full, it returns to the normal display for a perfect egg. 

## Random number generator for events
There is a random number generator (RNG) peripheral in the microbit (see section 6.19 in the [nRF52833 manual](https://infocenter.nordicsemi.com/pdf/nRF52833_PS_v1.5.pdf) ). The RNG generates true non-deterministic random numbers based on internal thermal noise. In fact, these numbers are so random that they are suitable for cryptographic purposes. In contrast to a linear congruential generator (LCG), the RNG peripheral does not require a seed value.
![[Screen Shot 2022-05-13 at 12.16.43 pm.png]]
The RNG is started by triggering the `START` task and stopped by triggering the `STOP` task. When started, new random numbers are generated continuously and written to the value register when ready. A `VALRDY` event is generated for every new random number that is written to the `VALUE` register. This means that after a `VALRDY` event is generated, the CPU has the time until the next `VALRDY` event to read out the random number from the `VALUE` register before it is overwritten by a new random number.

Using the RNG peripheral is much like using the LEDs. We set a certain bit in the `TASK_START` register (`0x400D000`, the RNG base address) to start generating random numbers, we read in a random number into a register using the `VALUE` register (offset from RNG by `0x508`), and then we set a bit in the `TASK_STOP` register (offset `0x004`) to stop generating random numbers.

This approach initially worked when I stepped through it in the debugger. However, when I just let the program run, no random number would be loaded into `r0`. I eventually figured out that this was because the RNG peripheral takes time to generate a random number. If I attempted to load the random number from the `VALUE` register immediately after setting `TASK_START`, the value wouldn't be ready. After reading through the documentation more carefully, I realised that this was the reason for the `VALRDY` register: it's set to 1 if the random number has been generated, and 0 otherwise. Loading and checking this value allowed me to reenter a simple loop if the random number wasn't ready.
```ARM
@ CONSTANTS
.set RNG, 0x4000D000
.set RNG_OFFS_TASKS_START, 0x0
.set RNG_OFFS_TASKS_STOP, 0x004
.set RNG_OFFS_VALUE, 0x508 @ output random number

@ RANDOM NUMBER GENERATOR FOR EVENTS
rng:
push {lr}
	@ Puts the random number in r0.
	ldr r0, =RNG @ load base address of RNG in r0
	
	@ TASK START
	ldr r1, [r0] @ load TASK_START into register
	mov r2, 1
	orr r1, r1, r2 @ set bit
	str r1, [r0]
	
	@ LOAD RANDOM NUMBER
	@ first we check if a random number is ready
	rngLoop:
		ldr r0, =RNG
		ldr r1, [r0, 0x100] @ VALRDY register
		cmp r1, 1
		beq loadRNG
		b rngLoop
	
	loadRNG:
		ldr r1, [r0, 0x508] @ load random number into r1
		mov r4, r1
	
	@ TASK STOP
	ldr r1, [r0, 4] @ load TASK_STOP into register
	mov r2, 1
	orr r1, r1, r2 @ set bit
	str r1, [r0, 4]
	
	@ epilogue
	mov r0, r4
	pop {lr}
	bx lr
```

Now, we just need to tell the program (1) *when* to trigger a random event i.e. the conditions the random number has to meet for the event to be triggered, and (2) *what* to do for the event.
