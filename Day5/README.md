# Day 5: Supply Stacks
Official challenge : https://adventofcode.com/2022/day/5

Today's challenge is... more looping. Given a set of stacks, in this case lettters, follow instructions to move X items from the stack onto another stack. Of course, they have to be moved 1 by 1 so if you move 5 items from one, they end up in reverse order on the other stack. We're given a starting state of stacks, and a set of 502 instructions of how many to move where. In the end, we have to give out the top letter on each stack after all the instructions are completed to solve the puzzle.

This is the starting stack, to give you an idea.

```
[S]                 [T] [Q]        
[L]             [B] [M] [P]     [T]
[F]     [S]     [Z] [N] [S]     [R]
[Z] [R] [N]     [R] [D] [F]     [V]
[D] [Z] [H] [J] [W] [G] [W]     [G]
[B] [M] [C] [F] [H] [Z] [N] [R] [L]
[R] [B] [L] [C] [G] [J] [L] [Z] [C]
[H] [T] [Z] [S] [P] [V] [G] [M] [M]
 1   2   3   4   5   6   7   8   9 
```

I basically decided to hardcode this stack into a table:

```
Set(stacks, Table(
    { ID: 1, Value: "HRBDZFLS" },
    { ID: 2, Value: "TBMZR" },
    { ID: 3, Value: "ZLCHNS" },
    { ID: 4, Value: "SCFJ" },
    { ID: 5, Value: "PGHWRZB" },
    { ID: 6, Value: "VJZGDNMT" },
    { ID: 7, Value: "GLNWFSPQ" },
    { ID: 8, Value: "MZR" },
    { ID: 9, Value: "MCLGVRT" }));
```

Next, the list of instructions is in the form of `move 6 from 1 to 7` which means move 6 letters from stack 1 to stack 7. Since they're moving one by one, the essentially end up there in reverse. This list of instructions was again dumped in Excel as-is and imported into a Power App so we can grab it as a datasource.

I started with the easy way out. I obviously have to loop over the instructions, 502 in total. Then, I was looping again over a sequence of the number of items I'm supposed to move (so for `move 6 from 1 to 7` that would be 6 items). Inside that loop, I was patching the stack's records (letters) with Patch(). Unfortunately, the patching inside the double ForAll() just killed the performance. It couldn't finish without my browser complaining, and after multiple times of telling it I'd wait for it, it ended up killing the app. So, I had to split it up in multiple steps. I knew my logic worked because I'd tested and verified on the first few records. So it was a matter of refactoring.

So instead of trying to do everything in one cool formula like last time, I'll split this up into individual imperative steps. First, let's actually parse the instructions out:

```
ClearCollect(moves, ForAll(ContainerMoves,
    {
        moveCount: Value( Index(Split(Column1, " "), 2).Result ),
        fromStackNumber: Value(Index(Split(Column1, " "), 4).Result),
        toStackNumber: Value(Index(Split(Column1, " "), 6).Result)
    }));
```
Basically, I'm using Split() to divide the instructions (`move 6 from 1 to 7`) based on the spaces. Then I grab the number from that string (position 2, 4 and 6) and also converting them to Value(). That last one is likely not explicitly necessary.

Next, I'm putting the stack state into a collection so I can update it with Patch().

```
ClearCollect(day5Part1Stacks, stacks);
```

I'm still using the With() trick here, to avoid replicating this long formula a few times. Basically, the stack we're moving things from is a string and we need to get individual characters from it. So, we'll use Split() again. We're really only interested in the letters at the end of the string (top of the stack) as those are the ones we need to move. The number of letters we need to move is in the `moveCount` column we created earlier... So:

1. `Index(day5Part1Stacks, fromStackNumber).Value` gives us the string of the stack we need to move from.
2. From that string we do `Right( ... , moveCount )` to get the last amount of characters we need to move (`moveCount`).
3. Then we split this string up into individual characters using `Split( ... , "")`

So this results in:

```
ForAll(moves,
    With( { moveStack: Split( Right( Index(day5Part1Stacks, fromStackNumber).Value , moveCount), "") },

    ...
    )
);
```

So now what are we supposed to do? First, we need to add those characters we've now put in `moveStack` and add them to the destination string, in *reverse* order.

1. To reverse the string which is already split into characters, we do a ForAll() on a Sequence() of the number of characters (`CountRows(moveStack)`). We can index the characters in the string, but instead of using the sequence `Value` we need to index back-to-front, so the `CountRows()` minus the `Value` of the sequence. So for 5 characters, the first sequence would be 1, which equals 4. And the last one would be 5 characters mine the last sequence, which is 5, so equals 0. We're off by one here, so we need to +1 the index... We use `.Result` on the Index() return value because this is a single-column table created using Split() which has a `Result` column.
```
ForAll( Sequence(CountRows(moveStack)), Index(moveStack, CountRows(moveStack) - Value + 1).Result)
```
2. Now that we've reversed the characters, we need to put them back together in one string, which we can easily use Concat() on the previous formula: `Concat( ... , Value)` where we concatenate the single-column `Value` that contains the characters we output from the ForAll().
3. We then append this to the existing stack (string) from the stack we're moving them to, which we get directly from our stack collection: `Index(day5Part1Stacks, toStackNumber).Value`

Ok so to put this all together, here is the new string representing the TO stack after adding the FROM stack's characters in reverse:

```
Index(day5Part1Stacks, toStackNumber).Value & 
    Concat(ForAll(Sequence(CountRows(moveStack)), Index(moveStack, CountRows(moveStack) - Value + 1).Result), Value)
```

Next, we need to remove those characters from the FROM stack. That's a bit easier. We just grab the Left() side of the from string, just for the number of existing characters (Len() of the string) minus the ones we need to remove (`moveCount`):

```
Left(
    Index(day5Part1Stacks, fromStackNumber).Value,
    Len(Index(day5Part1Stacks, fromStackNumber).Value) - moveCount
    )
```

With those recalculated strings, we need to patch our collection and change the stacks. We can use Patch() on the collection, find the original record using Index(), and replace the `Value` column (and leave the ID which is unique for each stack). So taking the string formulas from above, and putting them all inside the ForAll() that loops over the moves, here's the final grand formula:

```
ClearCollect(day5Part1Stacks, stacks);

ForAll(moves,
    With( { moveStack: Split(Right(Index(day5Part1Stacks, fromStackNumber).Value, moveCount),"") },
        Patch(day5Part1Stacks, Index(day5Part1Stacks, toStackNumber),
            {
                Value: Index(day5Part1Stacks, toStackNumber).Value & 
                    Concat(ForAll(Sequence(CountRows(moveStack)), Index(moveStack, CountRows(moveStack) - Value + 1).Result), Value)
            });
        Patch(day5Part1Stacks, Index(day5Part1Stacks, fromStackNumber),
            {
                Value: Left(Index(day5Part1Stacks, fromStackNumber).Value, Len(Index(day5Part1Stacks, fromStackNumber).Value) - moveCount)
            });
        ""
    )
);
```

Awesome. So now we just need to grab the top of the stack (last character of the string) and that is our puzzle answer...

```
$"Top of the stacks: {Concat(day5Part1Stacks, Right(Value, 1))}"
```

After submitting we get Part 2 of the challenge. Turns out it's just a simplification of our original answer! Instead of reversing the string we move, the whole stack is moved together... so no need to reverse. We can basically copy/paste the previous answer, but take out the ForAll() loop that reverses the string... easy peasy:

```
ClearCollect(day5Part2Stacks, stacks);

ForAll(moves,
        Patch(day5Part2Stacks, Index(day5Part2Stacks, toStackNumber),
            {
                /// This now just takes Right(..., moveCount) without reversing
                Value: Index(day5Part2Stacks, toStackNumber).Value & Right(Index(day5Part2Stacks, fromStackNumber).Value, moveCount)
            });
        Patch(day5Part2Stacks, Index(day5Part2Stacks, fromStackNumber),
            {
                Value: Left(Index(day5Part2Stacks, fromStackNumber).Value, Len(Index(day5Part2Stacks, fromStackNumber).Value) - moveCount)
            });
        ""
);
```

The requested puzzle answer is the same, we just grab it from our part2 collection instead:
```
$"Top of the stacks: {Concat(day5Part2Stacks, Right(Value, 1))}"
```