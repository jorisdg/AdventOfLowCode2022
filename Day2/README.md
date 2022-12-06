# Day 2: Rock Paper Scissors
Official challenge : https://adventofcode.com/2022/day/1

The challenge consists of creating logic to calculate of a rock, paper, scissors game based on 2 inputs (from a two-column list), as well as calculating a score based on both the input and the outputs.

Similar to the first day, I just dumped the list into Excel, named the table, and imported the data into a Power App as a datasource.

The input consists of a first column containing an A,B or C which indicate your opponent's moves Rock (A), Paper (B) and Scissors (C). The second column are your moves as X,Y,Z again respectively indicating rock, paper or scissors. Calculating the score goes as follows:
- Your choice (XYZ/second column) is 1 point for rock (X), 2 point for paper (Y) and 3 points for scissors (Z).
- The outcome results in 0 if you lose, 3 for a draw and 6 for a win).

I decided to separate this in to two passes, which I changed up a bit after getting part two of the challenge which changes what the second column of X,Y,Z actually means. Also, to show the data on screen for verification I translate the A/B/C and X/Y/Z into a new column that actually says rock, paper or scissors. Finally, since the meaning of the second column changes in part two, let's add an explicit column for "My Play" which is your move. In this first part, your move IS actually the column. So the input we'll use to calculate the first challenge is this expression going on a button's `OnSelect`:

```
Clear(Game); // Clear any previous game calculations

ClearCollect(Day2Input, // replace the collection we'll use as game input
ForAll(RPSInput,
    {
        Column1: Column1,
        Column2: Column2,
        MyPlay: Column2,
        MyPlayText: Switch(Column2, "X", "Rock", "Y", "Paper", "Z", "Scissors")
    })
)
```
Looks fairly straight forward, just a small mutation of the data. Now we need the actual logic that calculates the game and the scores...

The calculation puts together a result table that contains the opponent's move (column `Them`), your move (column `Me`), the result (column `Result`) as Win/Draw/Lose, and finally the score (column `ResultScore`) based on the scoring calculation outlined earlier.

To calculate the outcome, we'll use ForAll() to loop over our input collection we created earlier. We can do a simple Switch() function over our play (`MyPlay`) and compare it to the opponent (`Column1`):
```
Switch(MyPlay,
    "X", If(Column1 = "C", "Win", Column1 = "A", "Draw", "Lose"),
    "Y", If(Column1 = "A", "Win", Column1 = "B", "Draw", "Lose"),
    "Z", If(Column1 = "B", "Win", Column1 = "C", "Draw", "Lose")
)
```
Similarly, for the scoring, we can re-use most of that logic but we have to add the points for the move we chose as well. Since we're already doing Switch() on our move, we can just add up a number for that with the number for the outcome (which is the same If() statement as above).
```
Switch(MyPlay,
    "X", 1 + If(Column1 = "C", 6, Column1 = "A", 3, 0),
    "Y", 2 + If(Column1 = "A", 6, Column1 = "B", 3, 0),
    "Z", 3 + If(Column1 = "B", 6, Column1 = "C", 3, 0)
)
```

So, fairly simply, our `Game` collection has the following expression:
```
ClearCollect(Game,
ForAll(Day2Input,
    {
        Them: Switch(Column1,
                "A", "Rock",
                "B", "Paper",
                "C", "Scissors"
            ),
        Me: Switch(MyPlay,
                "X", "Rock",
                "Y", "Paper",
                "Z", "Scissors"
            ),
        Result:
            Switch(MyPlay,
                "X", If(Column1 = "C", "Win", Column1 = "A", "Draw", "Lose"),
                "Y", If(Column1 = "A", "Win", Column1 = "B", "Draw", "Lose"),
                "Z", If(Column1 = "B", "Win", Column1 = "C", "Draw", "Lose")
            ),
        ResultScore:
            Switch(MyPlay,
                "X", 1 + If(Column1 = "C", 6, Column1 = "A", 3, 0),
                "Y", 2 + If(Column1 = "A", 6, Column1 = "B", 3, 0),
                "Z", 3 + If(Column1 = "B", 6, Column1 = "C", 3, 0)
            )
    })
)
```

So to complete the challenge, we just have to provide the total score. I just put the following expression in a label's `Text` property:

```
$"Game Score: { Sum(Game, ResultScore) }"
```

For part two of the puzzle, it is revealed the second column wasn't your move. Instead, the X/Y/Z is actually the desired outcome. Since the challenge remains to calculate the total score, we now need to calculate what move we should make to get the expected X/Y/Z. Having split up the input data from the calculation, we only need to update the first collection and manipulate the input.

Instead of blatantly putting `Column2` in the `MyPlay` column we just have to calculate what our play should be, based on the desired outcome. This isn't too complicated and resembles the Switch() function we did for calculating the result and score...

So, to complete Part Two of the second day challenge, we just change the first expression to the one below. We can then just re-run the calculation expression as it was.

```
Clear(Game);

ClearCollect(Day2Input,
ForAll(RPSInput,
    {
        Column1: Column1,
        Column2: Column2,
        MyPlay: Switch(Column2,
            "X", Switch(Column1, "A", "Z", "B", "X", "C", "Y"),
            "Y", Switch(Column1, "A", "X", "B", "Y", "C", "Z"),
            "Z", Switch(Column1, "A", "Y", "B", "Z", "C", "X")
            ),
        MyPlayText: Switch(Column2, "X", "Lose", "Y", "Draw", "Z", "Win")
    })
)
```
Again, this just changes the `MyPlay` column for the input to the calculation. When we re-run the calculation it will now have the scores based on the desired outcome from the input data.

That concludes the second day's challenge!