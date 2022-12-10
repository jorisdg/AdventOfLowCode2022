# Day 6: Tuning Trouble
Official challenge : https://adventofcode.com/2022/day/6

Today's challenge consists of going through a string and comparing the character with the last 4 characters encountered. If any are duplicate, keep going. Find the position of the first character in long string that makes a set of 4 without duplicates.

Honestly, took me longer to understand the question than to write the code. The examples help a lot, so go read the description.

Since we're iterating the characters, I'll start by already splitting it up in a table:

```
Set(day6CharacterTable, Split(""));
```

Inside those quotes goes the actual string input, which is so long I'm not putting it here. Since we need to keep some type of state inside the loop we're doing, I'll use a collection. We need to compare the last 4 characters, so we'll start with putting the first four characters in the collection already:

```
ClearCollect(lastFour, ForAll(Sequence(4), { Index: Value, Character: Index(day6CharacterTable, Value).Result }));
```

We're storing both the index where we found the character (column `Index`) and the character itself (column `Character`).

So, next we'll loop over all our characters. We use a sequence based on the number of character in our string. As I'm writing this up I realize there's a kinda-bug here but it doesn't matter. We already stored the first four, but this loop will start from 1 anyway. It doesn't result in any errors though, and it avoids more code later so we'll keep it.

```
ForAll( Sequence(CountRows(day6CharacterTable)),
```

So now, how do we check if a character is duplicated? Well, I'm just going to group that collection, and check if I have less than 4. If any are duplicated, they will be grouped and the GroupBy() will result in a table of less than 4...

```
If( CountRows(GroupBy(lastFour, "Character", "Indexes")) < 4,
```

Ok so if we have a duplicate, we want to move on to the next character. That means remove the First() character in our collection, and add the next one from our original character set. We store both the Index and the character itself, if you remember.

```
Remove(lastFour, First(lastFour));
Collect(lastFour, { Index: Value, Character: Index(day6CharacterTable, Value).Result });
```

Now, if we didn't find any duplicates, we have our answer. Unfortunately this is not a while loop, and we don't have a way to break out of this loop. Our answer is available in the `lastFour` collection: the `Index` column of the last record in that table. So we basically just have to do nothing. Our ForAll() will keep looping over our characters for no good reason, but our If() will keep saying there's no duplicates so will keep hitting the second expression. We'll just output 0. And to have both the true and false expressions of If() have the same type, we'll return 0 in the chained expression with the Remove() and Collect() as well.

So our final expression:

```
ForAll( Sequence(CountRows(day6CharacterTable)),
    If( CountRows(GroupBy(lastFour, "Character", "Indexes")) < 4,
        Remove(lastFour, First(lastFour)); Collect(lastFour, { Index: Value, Character: Index(day6CharacterTable, Value).Result }); 0,
        0
    )
)
```

The result of Part 1 of the puzzle is now:
```
$"Character: { Last(lastFour).Index }"
```

Nice. Part two of the puzzle turns out to be the exact same thing. Except now we need to check 14 characters instead of 4. Easy peasy. We just replace `4` with `14` in a copy/pasted expression:

```
ClearCollect(lastFourteen, ForAll(Sequence(14), { Index: Value, Character: Index(day6CharacterTable, Value).Result }));

ForAll( Sequence(CountRows(day6CharacterTable)),
    If( CountRows(GroupBy(lastFourteen, "Character", "Indexes")) < 14,
        Remove(lastFourteen, First(lastFourteen)); Collect(lastFourteen, { Index: Value, Character: Index(day6CharacterTable, Value).Result }); 0,
        0
    )
)
```

And the answer is:
```
$"Character: { Last(lastFourteen).Index }"
```