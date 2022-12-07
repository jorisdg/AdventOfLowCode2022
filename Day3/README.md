# Day 3: Rucksack Reorganization
Official challenge : https://adventofcode.com/2022/day/3

The challenge today is essentially to compare strings to one another to find common letters. The second part of the challenge is to find the common letters between three strings. Again I dumped the data into an Excel named table and imported into Power Apps as a datasource to work off.

For the first challenge, a list of seemingly random strings is given. We need to split the string in half, and find the common letter between the two halves. We can safely assume the strings are even and can be split exactly in half, and we can assume there is exactly one letter in common. Meaning, we don't need to worry about checking for errors as that is not what the challenge is about.

For this challenge I realized the value of the With() function. In previous challenges I used AddColumn or create a second list to simplify the expressions a bit (to avoid duplicating calculations). But I realized you do have sort of an equivalent of having variables in "the scoop of your loop" by essentially wrapping it in a With() call that has a hard-coded record with the values you need. &lt;insert exploding head emoji&gt;

So in this first challenge, I will first split each string (`Column1`) in two (supposedly the two compartments of the rucksack). That's easily done:

```
ForAll(RucksackData,
    {
        Compartment1: Left(Column1, Len(Column1)/2),
        Compartment2: Right(Column1, Len(Column1)/2)
    })
```

Now, I want to add a third column that has the letter that is shared between the two compartments. But I can't use `Compartment1` and `Compartment2` to calculate a third column because I'm still inside the record definition itself. Instead of duplicating the calculation again multiple times, I'm just going to put the previous calculated record inside a With() context, which allows me to use `Compartment1` and `Compartment2` for the actual record I want to return via ForAll(). I had initially named them the same, but I've renamed them with prefix `temp` to illustrate better what's going on.

```
ClearCollect(rucksack, ForAll(RucksackData,
    With({ tempCompartment1: Left(Column1, Len(Column1)/2), tempCompartment2: Right(Column1, Len(Column1)/2) }, // With() "context" with tempCompartment1 and tempCompartment2
        {
            Compartment1: tempCompartment1,
            Compartment2: tempCompartment2
        }
    )
));
```

So now, I can add the third column which actually does the comparison between `tempCompartment1` and `tempCompartment2`. What I will do is for each letter in the first string, I will check if it exists in the second. If so, I return the letter itself, otherwise Blank().<br/>
To iterate each letter in the string, I create a table of characters by using Split() on the "" separator (basically no separator, so each character separately). Then I check if the `Result` of that appears exactly in the other string (need to use `exactin` to match on casing per the instructions... if you use `in` it will be case insensitive).

```
ForAll(Split(tempCompartment1, ""), If( Result exactin tempCompartment2, Result, Blank() ) )
```

Then to get the letter, we can filter out the blanks and just grab the First() record (since there will only be one).

```
First(
    Filter(
        ForAll(Split(tempCompartment1, ""), If( Result exactin tempCompartment2, Result, Blank() ) )
        , Not(IsBlank(Value))
    )
).Value
```

So putting all this together, here's the full expression:

```
ClearCollect(rucksack, ForAll(RucksackData,
    With({ tempCompartment1: Left(Column1, Len(Column1)/2), tempCompartment2: Right(Column1, Len(Column1)/2) },
        {
            Compartment1: tempCompartment1,
            Compartment2: tempCompartment2,
            DuplicateLetter: First(
                Filter(
                    ForAll(Split(tempCompartment1, ""), If( Result exactin tempCompartment2, Result, Blank() ) )
                    , Not(IsBlank(Value))
                )
            ).Value
        })
    ));
```

Nice! So to get the answer for part 1, we're supposed to calculate the "priority" of each letter. Letters "a" to "z" have values 1 through 26, and letters "A" through "Z" have values 27 through 52. To accomplish this, I made a quick lookup table:

```
// Add lower case characters with priority 1->26
ClearCollect(priorities, ForAll(Sequence(26), { Char: Char(96 + Value), Priority: Value }));
// Add upper case characters with priority 27-52
Collect(priorities, ForAll(Sequence(26), { Char: Char(64 + Value), Priority: 26 + Value }));
```

I create a sequence of 26, and basically add that value to the ASCII value of the character prior to "a" (96), to actually get the character "a" when I doing Char(96 + Value). This gives me a table with `{ Char: "a", Priority: 1 }, { Char: "b", Priority: 2}` etc. Then I do the same thing again to append the upper case characters. Those start at 65 so I have to add 64+Value, and obviously the priority of A->Z starts at 27, so `Priority` equals `26 + Value`.

So all that's left to do to get the answer, is use Sum() of of the priority of each letter we found. Nothing a quick LookUp() can't fix:

```
$"Sum of priorities: {Sum(rucksack, LookUp(priorities, Char = DuplicateLetter).Priority)}"
```

For part 2, as usual it's similar but not quite. For each group of three rucksacks, we need to find the common letter and again add up the overall priority...<br/>
So we need to loop and group every three records in the original datasource. The quickest way I figured would be to ForAll() on a sequence that is one third of the total amount of rows we have, then Index() it. Clearly, this is not performant on a large dataset as each time I call Index() it is going to seek through the table, so this gets progressively longer and longer as we get to the end of the table - essentially exponentially getting slowed (look up Big O notation). Using ForAll() on the table itself would be a more linear approach - and we did this in the Day1 challenge. Splitting up is likely the better choice - what I did here is kind of nasty and performs badly :-)<br/>
But it makes for a more interesting one-line expression =)<br/>

Again I'm using the With() so we can generate some temporary data we can then use in the same ForAll() expression. First, we'll get the string by indexing the data. Our sequence `Value` column starts at 1, and we want to index three records on every value. So `(Value - 1) * 3` gives us the base index, and then we add +1, +2 or +3 for the three records we want to get. So we'll Index() three records this way:

```
Index(RucksackData, ((Value-1) * 3) + 1).Column1
Index(RucksackData, ((Value-1) * 3) + 2).Column1
Index(RucksackData, ((Value-1) * 3) + 3).Column1
```

For each of those, we want to again create a table for each character using Split(), and this time we'll make sure we only end up with the Distinct() characters, no duplicates. That makes our context record for With() look like this:

```
{
    Elf1: Distinct(Split(Index(RucksackData, ((Value-1) * 3) + 1).Column1, ""), Result),
    Elf2: Distinct(Split(Index(RucksackData, ((Value-1) * 3) + 2).Column1, ""), Result),
    Elf3: Distinct(Split(Index(RucksackData, ((Value-1) * 3) + 3).Column1, ""), Result)
}
```

So what we'll do is we first grab the characters common between `Elf1` and `Elf2`. Then find the common characters between that result and `Elf3`. As before, we'll use `exactin` and return Blank() where it's not in, and then we filter that out. So first comparing `Elf1` and `Elf2`:

```
Filter(ForAll(Elf1, If(Result exactin Elf2, Result, Blank())), Not(IsBlank(Value)))
```

Then we wrap that expression in virtually the same expression to compare it to `Elf3`:

```
First(Filter(ForAll(
    Filter(ForAll(Elf1, If(Result exactin Elf2, Result, Blank())), Not(IsBlank(Value)))
    , If(Value exactin Elf3, Value, Blank())), Not(IsBlank(Value))
)).Value
```

Note that this highlights our somewhat inconsistent usage of single-column tables having either a `Value` or a `Result` column. ForAll() returns `Value` whereas Distinct() returns `Result`. This is something we are planning to correct so `Value` is used everywhere.

And basically, that's the whole dataset. To look at the complete terribly-performing formula, here it is:

```
ClearCollect(rucksackGroups,
    ForAll(Sequence(CountRows(RucksackData) / 3),
        With({
            Elf1: Distinct(Split(Index(RucksackData, ((Value-1) * 3) + 1).Column1, ""), Result),
            Elf2: Distinct(Split(Index(RucksackData, ((Value-1) * 3) + 2).Column1, ""), Result),
            Elf3: Distinct(Split(Index(RucksackData, ((Value-1) * 3) + 3).Column1, ""), Result)
            },
                First(Filter(ForAll(
                    Filter(ForAll(Elf1, If(Result exactin Elf2, Result, Blank())), Not(IsBlank(Value)))
                    , If(Value exactin Elf3, Value, Blank())), Not(IsBlank(Value))
                )).Value
            )
        )
    );
```

Now again to get the Sum() of the priorities for the letters we found, the following expression does exactly what we need.

```
$"Sum of priorities: {Sum(rucksackGroups, LookUp(priorities, Char = Value).Priority)}"
```

And that concludes the third day's challenge!

P.S.: Don't tell my colleagues about the exponential Big-O performance problem I created here.