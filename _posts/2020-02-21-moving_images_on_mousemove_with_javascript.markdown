---
layout: post
title:      "Dragging images with mousemove in JavaScript"
date:       2020-02-21 20:13:20 -0500
permalink:  moving_images_on_mousemove_with_javascript
---

After reaching a certain point in learning JavaScript, I knew immediately what kind of web app I wanted to build with it when it came time to do a project. How much control JavaScript gives you over your front end seemed like a perfect fit for making another Pokémon website (no one can say I don't have hobbies!), this one with Pokémon sprites sized to how tall they would actually be relative to their trainers. After getting this set up, the thing I wanted most to do next was be able to drag them around so a user could create images of their trainer "posing" with their Pokémon. 

![Example of positioned Pokémon: some pokésnakes](https://lh3.googleusercontent.com/0lZec60qbwGyMHA4PFz7yG8eAhpwMHn2Aoihmz28yIYYAi-wm6MTg_Zn2B4M2SpXEFL4AeqUl3U40hSawmWcYuGGh3qY_8eteJ0ALixCPfVHvTiZxzSKB0iG_b3RPdlY1bz_nWCuZw=w2400)

This is where I hit a snag. My initial thought was to do movement through arrow keys, but this would be too many events and therefore too many PATCH fetch requests for my server. When it hit me to check google to see if there was a "drag" event listener I was elated to find one, but after hours perusing this documentation, it became clear to me that though the behaviour of this event looked similar to the behaviour I wanted, how it worked on the backend was quite different. The drag event listeners do involve moving elements, but are mostly concerned with transferring element data from one node to another (e.g. dragging an item from being a child of your "to do" list to be a child of your "done" list instead), not  with the page position the item was being dragged to. 

The event listeners I actually wanted were related to mouse movement, and since there was a lot of trial and error involved in getting this to work, even despite my attempts to follow some other tutorials, I'm going to get into what worked for me here. 

The first step was setting up event listeners for all of the events in question. My project was made with Object Oriented JavaScript, so I did this in a function on my `Pokemons` class that initialized all of my bindings and event listeners. 

```
initBindingsAndEventListeners() {
    this.view = document.getElementById("view-box")
    this.view.addEventListener("mousedown", this.onMouseDown.bind(this))
    this.view.addEventListener("mousemove", this.onMouseMove.bind(this))
    this.view.addEventListener("mouseup", this.onMouseUp.bind(this))
    this.view.addEventListener("dragstart", this.onDragStart.bind(this))
}
```

(The `.bind(this)` here is related to how my class is set up—it gives the function I'm calling the context of the instance of the class so it can access other methods and variables I have defined on this instance.)

Next, I had to define all of these class methods—and yes, all of these methods are necessary for moving images by dragging them to work! 

```
  onDragStart(e) {
    e.preventDefault()
  }
```

Images are `draggable` by default, an attribute necessary for dragging events to work, so when you click and start to drag one, the event that happens is `dragstart`. This would be fine if I wanted to use dragging event listeners to deal with movement, but since I didn't, I had to define a method to prevent the behaviour of the default event from firing.

```
  onMouseDown(e) {
    e.preventDefault()
    let movingSprite = e.target
    if (movingSprite.id.includes("pokesprite")) {
      movingSprite.style.position = "absolute"
      movingSprite.style.zIndex = parseInt(movingSprite.style.zIndex, 10) + 7
      function moveAt(pageX, pageY) {
        movingSprite.style.left = Math.round(pageX - movingSprite.offsetWidth / 2) + 'px';
        movingSprite.style.top = Math.round(pageY - movingSprite.offsetHeight / 2) + 'px';
      }
      moveAt(event.pageX, event.pageY)
      this.isMoving = true
    }
  }
	```
	
The first part of all of the rest of these methods was preventing the default action so I could set my own actions up. From `onMouseDown` I needed to access the target being clicked, in this case the image that was being dragged, and if it was the target I wanted to move (if its id included `pokesprite` in this case), I had to make adjustments to it so it could be moved. 

This was where I ran into my first stumbling block: images automatically have their position set to `static`, which means they will render in the order they appear in the document flow. This needs to be changed to `absolute`, where the image is positioned relative to its first positioned ancestor element instead. If the image's position is `static`, changing what the top and left styles are set to doesn't have any effect on where the image renders. I also incremented the `zIndex` property in this function so the object being moved would be above the other objects that could be moved on the page. 

I also set a `this.isMoving` boolean to true in the `onMouseDown` method so I could check for it in the next two functions. I only wanted the code in `onMouseMove` and then in `onMouseUp` to fire if an image had been clicked on—otherwise I would have run into errors like starting a target image moving simply by hovering over it. 
	
```
onMouseMove(e) {
	e.preventDefault()
	let movingSprite = e.target
	if (this.isMoving === true && movingSprite.id.includes("pokesprite")) {
		function moveAt(pageX, pageY) {
			movingSprite.style.left = Math.round(pageX - movingSprite.offsetWidth / 2) + 'px';
			movingSprite.style.top = Math.round(pageY - movingSprite.offsetHeight / 2) + 'px';
		}
		moveAt(event.pageX, event.pageY)
	}
}

onMouseUp(e) {
	e.preventDefault()
	if (this.isMoving === true && movingSprite.id.includes("pokesprite")) {
		this.isMoving = false
		this.updatePokemonPosition(e)
	}
}
```

The code seems a bit repetitive through these other methods, but in order for the movement to work properly, `preventDefault()` needs to be called on every action so the only things happening are what is defined in the methods. The `moveAt()` function needs to fire on `mousedown` and `mousemove` so the image will move properly in both. In `onMouseUp`, I set the `this.isMoving` boolean to false so the `onMouseMove` method would no longer fire once the user had stopped dragging the image, and then I could call the method to `updatePokemonPosition`. 

The position has already been updated in the DOM by these methods, but the `updatePokemonPosition` method that gets called here sends the `e.target.style.left`, `e.target.style.top` and `e.target.style.zIndex` attributes to a method that uses fetch to send a PATCH request to the API and update the sprite's position there. This means that the next time the page loads, it will still be in the same position it was left in! 

I hope this is helpful to anyone else struggling with similar issues I was! 

![Example of positioned Pokémon: some pokébirds](https://lh3.googleusercontent.com/fkOSEYBfiEwoR3T2fUgCe7MT6dyR2Q_zjPKZQWptz-VKkeiZzC85TNZsghqXjwAFfuw6gXKw9Rj3ijzTLlUbFsJlJFLxvOQ9ftNA1Mm2eKSTlgByeBJYft2WDKDASlJMzs_jQrXtuw=w2400)






