# Personal pet project
## Using records: storing attributes in memory
```ARM
.data
attributes:
.word 5, 4, 3
```
## Home screen: displaying attributes
We need some way to show the user how their pet is going, in terms of both hunger and contentment, and also overall health. My initial thought was to use a single vertical bar for health, like so:
![[health_bar.jpg|300]]
Each LED in this bar represented a health point. I would then show the pet to the right of this as its own entity i.e. a head and a body. However, after some experimentation, I decided that this system would be too clunky. I simply didn't have enough screen space (that is, enough LEDs) for this system to be effective. Another problem with this approach is that it left me no room to display energy points. I would have had to implement a different peripheral (separate from the two buttons) if the user wanted to check how many energy points they had. 

Because of this, I decided to incorporate different attributes into the entity display of the pet itself. 

This still left the issue of how to display energy points. Because it was a bit of a challenge, I decided to see if I could use the touch sensor to trigger a screen which would display the energy points left. 

### Rendering function
So, now that we have the design for the pet egg, we need a way to display all components of the egg at once. We can't just have one function for rendering the egg. This is because different components of the egg rely on different attributes for how many LEDs to display. (I mean, I guess we could have one function, but it would get awfully messy.) Because of this, I'm going to write a function to display the health, hunger and happiness attributes separately. We'll then write a `render` wrapper function over these three parts of the egg, looping through them so quickly that they appear to be show at the same time. 

#### Initial display

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

#### Reflecting updated attributes in the display
However, we haven't finished our rendering function yet. We don't just want a static egg - we want our egg to reflect the current state of the pet's attributes in memory. This basically means that we're going to need to wrap the render loop itself in an outer main loop. This loop will display the pet for a short period of time, check its attributes, and display it again based on the updated attributes. This way, if an event occurs (e.g. we've fed the pet, played with the pet, the timer has decremented the pet's hunger, etc.) the state of the egg will be updated on the screen.

As we discussed above, if the pet's attributes decrement, we're going to turn off LEDs relating to that attribute in the display. This choice of display is probably a good one; it's easy to imagine the egg decaying or breaking due to neglect or overstimulation. The order for removing LEDs will be:
* *Hunger and happiness:* first, we'll take out the middle LED. Then, the top and bottom LEDs (in that order). Finally, the 2nd and 4th LEDs.
* *Health:* since health is a vertical bar, it's a bit easier. We'll take out the middle LED, then the 2nd and 4th LEDs, and finally the top and bottom LEDs. The bottom will be last because it represents the base of the egg. 

>[!error] Warning
>Remember, once the final health point attribute goes (represented by the bottom LED in the health bar), the egg dies. 

## Setting up the two buttons
We need to configure the two buttons, A and B, to trigger a feeding event and a playing event respectively. 


## Configuring the touch sensor

## Adding sounds

## Endgame
No game would be complete without an objective. Since this project is only meant to be engaging over 2-3 minutes, I thought it would be fun to make the outcome of the game binary over such a time: either you keep the pet alive, or it dies. If it stays alive, we can display some sort of evolution animation of the pet breaking out of its shell.

## Improving the code
### Refactoring display functions using memory
