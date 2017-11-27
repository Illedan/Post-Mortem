# Mean max


## Contest
The contest was funny and challenging! 
Good job to the creators. Nice clean Refree!
No problem with the creators participating! 

## Strategy
Some kind of mutating search with a few crossovers. (not sure if I can call it a GA since I added too many new random gnomes every round)

## Simulation

### Cache collisions
Created a list of every initial collision, found the first one (didn't bother to sort the list), performed it and only recalculated the collisions of the participating units.

### Grid collision detection



## Enemy prediction
Did a 5 ms search for each enemy given the other players were dummies during their search. 
Dummies were doing straight line 300 thrusts against:
* reaper => wreck (did use 0 thrust if closer than speed)
* Detroyer => tanker
* Doof => best reaper

## Scoring
The scoring defines what behaviours the units will have in a search. My units were trained together and all unit scores added together with the actual score gave the total score of each evaluation.

Scoring on points:
* My points * 39
* Target enemy * 10 
  * Target enemy if above me * 38
* Other enemy * 9.99
* 1000 * (depth - i + 1) for ending game sooner (i is round and depth is the total search depth)
* 100000 for winning (either by most points on round 200 or by the most points when someone reached 50)
* 10000 for second place
* -100000 for losing

### Doofer
At the start of each round I would pick which player to attack based on simple rules:
* score < 10 => Keep speeding and gain Rage to prepare for later.
* score > 10
  * Player who's score is closest to me, but take highest if equal. 

This made my AI focus on getting 2nd instead of losing which gained me alot of points.
Then I 

### Reaper



### Destroyer

#### Vornoi area
No idea if this is a term, but I'll use it. A vornoi area (in my world) is an area of the map which gains a certain score based on the gamestate. I used circles that are created around the map in a Radius of 4000 from the center and every 30 degrees (https://tech.io/snippet/Dtmv8sO)
Every step of my simulation I would find every Tanker that was within the radius(2000) of every circle and give those circles (each tank could be in multiple circles since they overlap) score based on:
* Water
* If it's a wreck => + 1.1
* If it's alive => Capacity - water*0.5
I gave my Destroyer negative score based on distance to the best circle.



## What didn't work

* Offline training.. Wasted 10+ hours on getting a simulation going with a whole lot of bugs. When it started running the output didn't help at all. Moving back to my initial trainingdata gained me 15 spots compared to the best from the learning.

* Using Gridparts in a list. There is not enough units here to use gridcollision detection in a list, insert, delete is just too slow.
