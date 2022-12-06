# Day 1: Calorie Counting
Official challenge : https://adventofcode.com/2022/day/1

The challenge here lies in parsing out a flat list of calories (numbers) of snacks carried per elf, and any time a separator is found on a line (in this case, a blank line) it indicates the next set of values is for another elf.

To make the data available in my app for this challenge, I decided to just dump the flat list into Excel (and make it a named table) and import it into a Power App as a datasource. What I had not anticipated is that the import strips out the blank lines... Since clearly the challenge is about writing the code to iterate the list, I decided to replace the blank lines with a dash ('-') and import that. The formulas I wrote would work just as well with a blank line, if I took the time to get the data into Power Apps with the blanks to begin with.

The end result is a datasource (which I called `Calories`) with just a `Column1` column containing a number or a '-' to separate the lines.

To write the code for this, the challenge immediately runs into the looping problem in Power Fx. ForAll() is the function to use, but you can't just write some imperative logic inside ForAll() as if it were a traditional loop. Also, ForAll expects you to return a record in each loop, as it returns a table of results. The challenge result is a calculation of total calories summed per elf (the separated values). So changing the list to have an elf number (id?) would then allow us to use the Group() function... having a zero or something for the separator lines allows us to filter those out. GREAT! Now the final challenge is, how do we increment an elf counter inside ForAll() every time we encounter a separator line? You can't use Set() inside the ForAll() (which I'm unclear why, I'm talking to the team about this as it may be bug), but you can use Collect() just fine. So the workaround I went with is to Collect() an incremental number into a temporary collection, and we can always read the Last() record of this collection...

Alright, so here's the code. We want to start with Elf #1 as there is no separator in the list in the beginning, plus we need a beginning number to query for, as you'll see.

```
ClearCollect(temp, 1);
```

Next, we want to loop over our list using ForAll(). We emit a record with an `Elf` column and a `Calories` column. The `Calories` column is just the `Column1` of our list. But, if the `Column1` value is "-" (the separator) then we also add a value (1 more than the last value) to the collection. We use Last() to get the last value in the table. That is another reason we want to make sure our temp table contains a record to begin with. Note that we use a semi-colon (';') to chain the expression together inside the If statement, a Collect() and a record expression when we encounter the separator.

```
ForAll(Calories,
    If(Column1 = "-",
        // If row is a "-", increase the Elf counter and create a record
        Collect(temp, Last(temp).Value + 1); { Elf: Last(temp).Value, Calories: Column1 },
        // Otherwise just create a record
        { Elf: Last(temp).Value, Calories: Column1 }
    ))
```
The result of this will include the separator record as well, but the `Calories` column will have the separator "-" in it. So that's easy enough to wrap the previous expression with a Filter():

```
Filter(ForAll(Calories,
    If(Column1 = "-",
        // If row is a "-", increase the Elf counter and create a record
        Collect(temp, Last(temp).Value + 1); { Elf: Last(temp).Value, Calories: Column1 },
        // Otherwise just create a record
        { Elf: Last(temp).Value, Calories: Column1 }
    )), Calories <> "-")
```
Finally, we'll wrap that to store that resulting table in a variable named `elves`:
```
// Create elves data
Set(elves, Filter(ForAll(Calories,
    If(Column1 = "-",
        // If row is a "-", increase the Elf counter and create a record
        Collect(temp, Last(temp).Value + 1); { Elf: Last(temp).Value, Calories: Column1 },
        // Otherwise just create a record
        { Elf: Last(temp).Value, Calories: Column1 }
    )), Calories <> "-"));
```

Now that we have the data in the queryable shape we need it in, we can use the GroupBy() function on the `elves` table to group the calories into a `Snacks` column with the calorie values:

```
GroupBy(elves, "Elf", "Snacks")
```
This results in a table with two columns. An `Elf` column with the number/ID, and a `Snacks` column that is the list of calories for that elf, based on the original list. Now it's trivial to use AddColumns() to add a sum of the total number of calories, which is what we need to complete the challenge. I also added a `Number of Snacks` column but that wasn't necessary for the challenge. I just wanted to put this resulting collection on screen and look at the data. Finally, since this became the second puzzle for the day (which you get when you complete the first), I sorted this on the `Total Calories` columns, descending, as we'll need to get the top three.

```
// Now group snack-calories by elf, and add a total calories and sum of snacks into a new collection
ClearCollect(CaloriesByElf,
    Sort( AddColumns(GroupBy(elves, "Elf", "Snacks"), "Total Calories", Sum(Snacks, Calories), "Number of Snacks", CountRows(Snacks)), 'Total Calories', SortOrder.Descending) );
```

This concludes the data gathering that we need. I added all of that to a button's `OnSelect` to execute. Final expression:

```
ClearCollect(temp, 1);

// Create elves data
Set(elves, Filter(ForAll(Calories,
    If(Column1 = "-",
        // If row is a "-", increase the Elf counter and create a record
        Collect(temp, Last(temp).Value + 1); { Elf: Last(temp).Value, Calories: Column1 },
        // Otherwise just create a record
        { Elf: Last(temp).Value, Calories: Column1 }
    )), Calories <> "-"));

// Now group snack-calories by elf, and add a total calories and sum of snacks into a new collection
ClearCollect(CaloriesByElf,
    Sort( AddColumns(GroupBy(elves, "Elf", "Snacks"), "Total Calories", Sum(Snacks, Calories), "Number of Snacks", CountRows(Snacks)), 'Total Calories', SortOrder.Descending) );
```

Finally, the actual numbers we need to complete the challenge were the following expressions I put in a label's `Text` property:

Part 1:
```
$"Highest total calories: { First(CaloriesByElf).'Total Calories' }"
```
Part 2:
```
$"Highest 3 total calories: { First(CaloriesByElf).'Total Calories' + Index(CaloriesByElf, 2).'Total Calories' + Index(CaloriesByElf, 3).'Total Calories' }"
```

And that concludes the first day's challenge!