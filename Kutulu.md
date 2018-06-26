# Code of Kutulu

Complex little game with loads of fun and frustration (the ranking random madness). 5th, C# and bearly beaten by MSmiths.

## Strategy

Simulation with random/dfs search. I used an array of doubles as my solution and created moves based on the current value * possible moves at this position. Since possible movements on a given position never changed (like in WW or Tron) it would still be a valid move, even if the oponent did something else (nearly true, but I didn't simulate YELLs making it true for my simulation).  

## Searches

A search is an array of doubles, size beeing search depth, which is sent to an eval function.

### Evaluation
Example of the evaluation of an array of doubles:

```
public static double Evaluate(int depth, Game game, double[] data, Explorer me, List<TrainedData> trained)
{
    var score = 0.0;
    for (var i = 0; i < depth; i++)
    {
        DoMove(data[i], me);
        if (trained != null)
        {
            foreach (var t in trained)
            {
                if (t._depth > i && !t.Explorer.IsDead && t.Explorer != me)
                    DoMove(t.Data[i], t.Explorer);
            }
        }

        game.GameLoop();

        score += ScorePos(game, me) + 10000; // add 10000 to be sure to live longer
        if(me.IsDead) break;
    }

    game.Reset();
    return score + (me.IsDead ? -1000000 : 0);
}

public static void DoMove(double value, Explorer current)
{
    var moves = Moves[current.X, current.Y]; // All possible moves from this location
    var selectedAction = moves[(int)(moves.Count * value)];
    current.X += selectedAction.Dx;
    current.Y += selectedAction.Dy;
}

public class TrainedData
{
    public int Depth;
    public double[] Data;
    public Explorer Explorer;
}

```

Score pos evaluates the gamestate, using these properties and values:
- 4 * Sanity
- -79.999 if I only have 2 moves (standing still + another, indicating a dead end)
- -0.2 * Math.Max(1, SanityLoss - SanityLossGroup) * Dist to explorers
	- -5.2 * Math.Max(1, SanityLoss - SanityLossGroup) for closest if dist > 2. Made it walk through a wanderer if it's too dangerous to walk alone.
- -5/dist of wanderers
- -5 if wanderer target is me / someone on the same spot
- -79 if wanderer closer than 3 and someone can stun me
- -5/dist of slashers closer than 10
- -79 if slasher can attack me if someone stuns
- If I am on the line of a rushing slasher and doesn't get hit. Lose 7 sanity

I tried to use decay of score, but I found it better without.

Values of -79 (previously -3) was added as I went to work 1 hour before the end of the contest. No idea if it was worth it.. (but I saw some bugs, and I think it fixed them, and added new ones...)

### State of the game
After I'm done with a search, I reset the game state. This means I allways keep the same instance, saving object creation time.
Game.Reset calls every Unit's reset function. As an example, Explorer:
```
public override void Reset()
{
    base.Reset(); // Reset X and Y
    IsDead = false;
    Sanity = _sanity;
    LightEffect = _lightEffect;
    PlanEffect = _planEffect;
    Stunned = _stunned;
}
```

### MC - with mutations
Created a random array, checked score. Repeated for N ms while increasingly doing mutations instead of random on the best found solution.
Example: https://tech.io/snippet/sRwzsFe

### DFS
Search which, recursively made a double array with every possible values given the position of the explorer at that depth.
Did a full game simulation on the full depth after every new value and saved best found on the way.
(this could be very optimized, but I didn't have the time or motivation. Since I'm searching the full depth, every time I change a value)

### Total flow of search

Search was done in this order:

- Start timer
- Foreach explorer 
	- Remove all other explorers
	- Do a full 3 depth DFS for this explorer 
	- Add best found to a list of best moves
- while timer.time < 25 ms
	- pick random explorer(incuding me)
	- Do a full 4 depth DFS for this explorer using `TrainedData` for other players towards their trained depth, wait if deeper.
	- Replace Data and Depth in list of best moves for this explorer
- while timer.time < 40 ms
  - Do a random search for me with a depth of 5

## Skills

I used skills when my search returned a wait. No part of my search weighted skill usage in any way. (except for checking if an enemy could stun me)

Yell:
Any explorer closer than 2 and any wanderer closer than 3. (A little too stupid, but I failed to use simulation and kept my logic)

Plan:
Sanity < 220, and more than 0 explorers nearby. I find it more important to use above 200, since you delay the spawn of slashers making you live a little longer (maybe)

Light:
Any wanderer more than 4 away and any explorer more than 10 away. (Using actual path distance)

## MISC optimizations

### Path distances
I did a FloydWarshall at the first round to find all distances to every other node. 
Every path has 5 values:
- Length
- 4 values of where to walk next given round % 4 towards this target. (used in simulation by wanderers and slashers

### Possible moves
Every board position has a list of all possible moves from this location (including standing still).
This was used to speed ut the calculation of which move to use, since I used a double array

### Light calculation in sim
Light is used by wanderers and slashers to move towards a target. I added some simple logic to select which target they would choose given explorer lights.

```
public int FindPathLength(Explorer explorer)
{
    var dist = RealDist(explorer); // Floyd distance
    if (explorer.AlliedLightEffect != null) // Updated every round to account for the closest light I stand on, still active.
    {
        var toSource = RealDist(explorer.AlliedLightEffect);
        var toExplorer = RealDist(explorer);
        if (toSource > toExplorer)
        {
            dist += Math.Min(toExplorer, (LIGHTRANGE - (toSource - toExplorer))) * 3;
        }
        else if (toSource < toExplorer)
        {
            dist += Math.Min(toExplorer, (Math.Min(LIGHTRANGE, toSource) + Math.Min(LIGHTRANGE, toExplorer - toSource))) * 3;
        }
        else dist += Math.Min(LIGHTRANGE, toExplorer * 3);
    }

    return dist;
}
```
I never actually tested this though, but it's running in my submitted version.


## What didn't work
- FloodFill: Tried to floodfill up to N depth from my current location, stopping at walls or wanderers. Guess it didn't indicate the actual score of the position + it is slow making for a lot less simulations.
- Skills in sim: Tried using skills in sim, but guess it was too many failures.. I might try to only use it at the first round as explained by euler.
