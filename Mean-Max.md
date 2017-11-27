# Mean max


## Contest
The contest was funny and challenging! Good job to the creators. Nice clean Refree!


## Vornoi area
No idea if this is a term, but I'll use it. A vornoi area (in my world) is an area of the map which gains a certain score based on the gamestate. I used circles that are created around the map in a Radius of 4000 from the center and every 30 degrees (https://tech.io/snippet/Dtmv8sO)
Every step of my simulation I would find every Tanker that was within the radius(2000) of every circle and give those circles (each tank could be in multiple circles since they overlap) score based on:
* Water
* If it's a wreck => + 1.1
* If it's alive => Capacity - water*0.5


## What didn't work
