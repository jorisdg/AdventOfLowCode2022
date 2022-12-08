# Day 4: Camp Cleanup
Official challenge : https://adventofcode.com/2022/day/4

The challenge today is about parsing out a string into range of numbers, and comparing those ranges to find overlaps. Again I dumped the list as-is into Excel. I considered splitting out by comma first, but then realized that 1) I could easily do this in Power Fx and 2) it's technically part of the challenge and 3) if you split it out in Excel it will convert some of the "number-number" into dates =)

This challenge was actually super easy to do. I used Day3's revelation of using With() again to first parse out the list.

We'll do a ForAll() over the list of pairs. We first have to split the string, it comes in format `A-B,C-D` where A,B,C,D are numbers. So first we need to split by the comma (,) to get the two pairs, then split by the dash (-) to get the low and high number of the range. For convenience, I'll immediately convert these to numbers with Value() so we can compare them properly.

We can use the Split() function twice to split the column first by comma, then by dash. Since we only have two ranges, we'll just do it twice and get First() and Last():

```
Split(First(Split(Column1, ",")).Result, "-") // First range
Split(Last(Split(Column1, ",")).Result, "-") // Second range
```

Next, we'll convert this text into numbers by iterating the two resulting text numbers and converting them with Value():

```
ForAll(Split(First(Split(Column1, ",")).Result, "-"), Value(Result))
ForAll(Split(Last(Split(Column1, ",")).Result, "-"), Value(Result))
```

So now, something like `"1-4,3-5"` will result in `[1,4]` and `[3,5]`. I will put this inside a With() to have these values ready inside our loop we'll do over the full list of pairs:

```
With(
{
    Elf1: ForAll(Split(First(Split(Column1, ",")).Result, "-"), Value(Result)),
    Elf2: ForAll(Split(Last(Split(Column1, ",")).Result, "-"), Value(Result))
},
...
```

So now on to the calculation, which is easy enough. We'll do two checks: 1) does Elf1 contain Elf2 and 2) does Elf2 contain Elf1. We can them count the rows where either of those is true, and we have the number we're after.

Remember our Elf1 is a table of two numbers, so the First() and Last() are the upper bounds. All we have to check if the first of one Elf is less or equal to the first of the other, and check if the last is more or equal to the other...

```
{
    Elf1ContainsElf2: First(Elf1).Value <= First(Elf2).Value && Last(Elf1).Value >= Last(Elf2).Value,
    Elf2ContainsElf1: First(Elf2).Value <= First(Elf1).Value && Last(Elf2).Value >= Last(Elf1).Value,
}
```

Putting it all together, here's the formula:

```
Set(pairs,
ForAll(SectionPairs,
    With( { Elf1: ForAll(Split(First(Split(Column1, ",")).Result, "-"), Value(Result)), Elf2: ForAll(Split(Last(Split(Column1, ",")).Result, "-"), Value(Result)) },
        {
            Elf1ContainsElf2: First(Elf1).Value <= First(Elf2).Value && Last(Elf1).Value >= Last(Elf2).Value,
            Elf2ContainsElf1: First(Elf2).Value <= First(Elf1).Value && Last(Elf2).Value >= Last(Elf1).Value
        }
    )
)
);
```

Now to get the result, we just add a label with this formula:

```
$"Pairs where one range fully contains the other: { CountRows(Filter(pairs, Elf1ContainsElf2 || Elf2ContainsElf1)) }"
```

The second part of the challenge is fairly similar. Instead of checking one containing the other, any form of overlap also counts. As "containing" is also a form of overlap, we can just keep the existing calculation and add a secondary that checks only for overlap. We can then check if the edge case of "contains" or any overlap has occurred.

Take an example of `"1-4,3-5"`... they overlap if the lower bound of the first range is lower than the lower bound of the second, and when the higher bound of the first is both higher than the lower bound of the second but also lower than the higher bound of the second. Since we also need to check the opposite, where the second overlaps the first, we do this twice.

```
Elf1OverlapsElf2: First(Elf1).Value <= First(Elf2).Value && Last(Elf1).Value >= First(Elf2).Value && Last(Elf1).Value <= Last(Elf2).Value,

Elf2OverlapsElf1: First(Elf2).Value <= First(Elf1).Value && Last(Elf2).Value >= First(Elf1).Value && Last(Elf2).Value <= Last(Elf1).Value
```

Hope that makes sense. So we just append our original calculation with that, and end up with this full resulting expression:

```
Set(pairs,
ForAll(SectionPairs,
    With( { Elf1: ForAll(Split(First(Split(Column1, ",")).Result, "-"), Value(Result)), Elf2: ForAll(Split(Last(Split(Column1, ",")).Result, "-"), Value(Result)) },
        {
            Elf1ContainsElf2: First(Elf1).Value <= First(Elf2).Value && Last(Elf1).Value >= Last(Elf2).Value,
            Elf2ContainsElf1: First(Elf2).Value <= First(Elf1).Value && Last(Elf2).Value >= Last(Elf1).Value,

            Elf1OverlapsElf2: First(Elf1).Value <= First(Elf2).Value && Last(Elf1).Value >= First(Elf2).Value && Last(Elf1).Value <= Last(Elf2).Value,
            Elf2OverlapsElf1: First(Elf2).Value <= First(Elf1).Value && Last(Elf2).Value >= First(Elf1).Value && Last(Elf2).Value <= Last(Elf1).Value
        }
    )
)
);
```

So now to get the answer for the second part, we just add this label:

```
$"Pairs where one range overlaps the other: { CountRows(Filter(pairs, Elf1OverlapsElf2 || Elf2OverlapsElf1 || Elf1ContainsElf2 || Elf2ContainsElf1)) }"
```

This was probably the easiest challenge so far!