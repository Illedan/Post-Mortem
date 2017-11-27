# Mean max
8th and C#!

## Strategy
Some kind of mutating search with a few crossovers. (not sure if I can call it a GA since I added too many new random gnomes every round)
My search had these properties:
Population: 30
Random genes: 10
Mutation: 10
Crossover: 10
Depth: 3

## Simulation
Started by getting my CSB code before I understood the simulation was different and ended copying from the refree instead. Could have been optimized more, but I spent my time on evaluation. Ended at 2k-5k simulations.

### Cache collisions
Created a list of every initial collision, found the first one (didn't bother to sort the list), performed it, subtracted the time of the first from every other collision on the list and removed those included in the first collision. Then added the new collisions of units in this first collision. This was repeated until the list was empty.
Gained 50% more simulations by using this.

### Grid collision detection
Only check collision between units contained in the same grid parts. To figure out which parts each unit is going to be in I create a 8 by 8 grid where each cell had it's unique ID. 
```
ulong currentId = 1;
for (int i = 0; i < Const.NUMCOLS; i++)
{
    for (int j = 0; j < Const.NUMROWS; j++)
    {
        GridParts[i, j] = new GridPart(i, j, currentId);
        currentId *= 2;
    }
}
```

Then every time a unit collides or at the start of the Collisioncheck phase I would find which parts it was contained in by using OR on each part it will enter given it's speed, position and radius:
```
public ulong FindNewParts(GridPart[,] parts, double timeLeft)
{
    var xMax = 6000 + X + Radius + Const.E;
    var yMax = 6000 + Y + Radius + Const.E;
    var xMin = 6000 + X - Radius - Const.E;
    var yMin = 6000 + Y - Radius - Const.E;

    if (VX > 0) xMax += VX * timeLeft;
    else xMin += VX * timeLeft;
    if (VY > 0) yMax += VY * timeLeft;
    else yMin += VY * timeLeft;

    var maxX = (int)Math.Min((xMax) / Const.COLWIDTH, Const.NUMCOLS - 1);
    var minX = (int)Math.Max(0, (xMin) / Const.COLWIDTH);

    var maxY = (int)Math.Min((yMax) / Const.ROWHEIGHT, Const.NUMROWS - 1);
    var minY = (int)Math.Max(0, (yMin) / Const.ROWHEIGHT);

    ulong id = 0;
    for (int i = minX; i <= maxX; i++)
    {
        for (int j = minY; j <= maxY; j++)
        {
           id |= parts[i, j].PartId;
        }
    }
    return id;
}
```

Collision checks is then optimized by:
```
public virtual double Collision_Time(Unit u)
{
    if ((u.ContainedGridParts & ContainedGridParts) == 0) return 2;
    ...
}
```

Gained 20% more simluation by using this.

### Spells
I used fixed positions based on the direction vectors to generate spell targets. 
```
public static Position GetGranadePosition(double target, Game game)
{
    var targetNade = (int)(target * 8 * 2);
    var targetVect = Vector.DirectionVectors[targetNade % 8]; // all 8 direction vectors
    var targetUnit = game.Players[targetNade / 8 + 1].Reaper; // + 1 because I am number 0
    return PositionFactory.GetNewPosition(targetUnit.X + targetVect.Dx * 300, targetUnit.Y + targetVect.Dy * 300);
}
```


### Factories
Create new objects is slow, I created factories for every object I had to "create" during simulation.
Example of a factory:

```
public static class EffectFactory
{
    private static Effect[] _effects = new Effect[100];
    private static int _counter = 0;
    private static Effect[] _used = new Effect[100];
    private static int _usedCounter = 0;

    public static int Created;
    public static Effect GetNewEffect(double x, double y)
    {
        var effect = _effects[_counter--];
        _used[_usedCounter++] = effect;
        effect.X = x;
        effect.Y = y;
        effect.Duration = 3;
        effect.IsMoveCreated = true;
        return effect;
    }

    public static void ResetEffects()
    {
        _counter = 99;
        _usedCounter = 0;
    }

    public static void Initialize()
    {
        for (int i = 0; i < 100; i++)
        {
            _effects[i] = new Effect(0, 0, true, 3);
        }

        _counter = 99;
    }
}
```
`IsMoveCreated` used by my simulation to determine which objects to remove after simulation.


## Enemy prediction
Did a 5 ms search for each enemy given the other players were dummies during their search. 
Dummies were doing straight line 300 thrusts against:
* reaper => wreck (did use 0 thrust if closer than speed)
* Detroyer => tanker
* Doof => best reaper

### Enemy target
Player who's score is closest to me in score, but take highest if equal. This made my AI focus on getting 2nd instead of losing which gained me alot of points.

## Scoring
The scoring defines what behaviours the units will have in a search. My units were trained together and all unit scores added together with the actual score gave the total score of each evaluation.
I scored my units every round of the simulation.

Scoring values:
* My points * 39
* Target enemy * -10 
  * Target enemy if above me or closer than 3 points away * -38
* Other enemy * -9.99
* 1000 * (depth - i + 1) for ending game sooner (i is round and depth is the total search depth)
* 100000 for winning (either by most points on round 200 or by the most points when someone reached 50)
* 10000 for second place
* -100000 for losing
* 0.25 * Rage (Granade = 60 => 15 score, which means it's to expensive to stop 1 point from the enemy)
* 0.0002 * Every distance score

### Doofer
Either get Rage or target a player:
* all scores < 10 => Keep speeding and gain Rage to prepare for later.
* any score > 10 => Focus on Enemy target

I would then score distances to the reaper, reaper's closest wreck and destroyer of that target equally. (giving my doofer the chance to drive around and not just hug 1 position)


### Reaper
Evaluated by 10 times the distance to the closest wreck and 5 times the distance to the destroyer

### Destroyer
5 times the Distance to the closest tanker of the reaper.

#### Vornoi area
No idea if this is a term, but I'll use it. A vornoi area (in my world) is an area of the map which gains a certain score based on the gamestate. I used circles that are created around the map in a Radius of 4000 from the center and every 30 degrees (https://tech.io/snippet/Dtmv8sO)
Every step of my simulation I would find every Tanker that was within the radius(2000) of every circle and give those circles (each tank could be in multiple circles since they overlap) score based on:
* Water
* If it's a wreck => + 1.1
* If it's alive => Capacity - water*0.5
* Also included distances (divided by 8000) to prefer closer circles:
  * Enemy doofers distance / 2
  * - Reaper distance *2
  * - Destroyer distance
  
I gave my Destroyer negative score based on 5 times the  distance to the best circle.

## What didn't work

* Offline training.. Wasted 10+ hours on getting a simulation going with a whole lot of bugs. When it started running the output didn't help at all. Moving back to my initial trainingdata gained me 15 spots compared to the best from the learning.

* Using Gridparts in a list. There is not enough units here to use gridcollision detection in a list. insert and delete is just too slow.
