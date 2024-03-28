This is the second post in a series on Yahtzee strategy. 

What if we only played to roll Yahtzees? That is, on every roll we simply kept whichever number we rolled the most of in chasing that dopamine high of a Yahtzee — after all, it is what the game is named for. How many should we expect to obtain over the course of a single game? And why do we care?

This post emerged from a Yahtzee game with friends where a new player ultimately employed a strategy somewhat along these lines — every roll he either targeted a Yahtzee or a straight. On Yahtzees, he did quite well, throwing two early on and crucially obtaining the massive 100-point bonus Yahtzee. On straights, his luck ran out: not only did his efforts to chase 2-3-4-5-6 get him in hot water with zeroes in four of a kind and full house, but they also decimated his top half. Competing with another friend who had completed her top-half bonus, filled all her lower-half categories including her first Yahtzee, and finished her final round well above the 300-point cutoff for a great game, his chances looked slim. Only yet another Yahtzee could save him entering the final roll. And miraculously it did. It was the first time I had ever seen three Yahtzees thrown by one player in a single game. 

Here, we'll explore what that strategy looks like taken to its extreme: optimizing all 13 rolls to maximize the number of Yahtzees. This should provide an upper bound on the number of Yahtzees we can expect from *any* player in a single game, since by definition there is no strategy with a higher expected number of Yahtzees than this one. 


```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt

np.random.seed(4133)
```

First, we define the basic functions necessary for Yahtzee simulation. These fundamentally allow us to roll the dice using `np.random()` and reroll specific dice while holding others in our hand.


```python
def roll_dice():
    """
    A function which chooses a random number between 1 and 6.
    """
    faces = [1,2,3,4,5,6]
    result = np.random.choice(faces)
    return(result)

def roll_multiple_dice(n_rolls):
    """
    Rolls an arbitrary number of dice.
    """
    roll_result = []

    for i in range(n_rolls):
        roll = roll_dice()
        roll_result.append(roll)

    return(roll_result)
    
def roll_five_dice():
    """
    A special function for playing Yahtzee. Equivalent to roll_multiple_dice(5).
    """
    return(roll_multiple_dice(n_rolls = 5))

def split_roll_input(roll_input):
    """
    Splits a string of 1s and 0s into a list, where each number is an element.
    """
    roll_input = str(roll_input)
    roll_list = [x for x in roll_input]

    return(roll_list)

def reroll_dice(held_dice, previous_roll):
    """
    Takes as an input the string 'held_dice' and the previous roll. 
    Rerolls all 1s and keeps all 0s.
    """
    reroll_result = previous_roll
    for i in range(len(held_dice)):
        if held_dice[i] == 1:
            reroll_result[i] = roll_dice()

    return(reroll_result)
```

Next, we build a function to optimize a single turn for a Yahtzee: this function plays a single turn of Yahtzee while holding the number which came up most frequently on the dice at each reroll and its output only tells us whether or not we rolled a Yahtzee on the turn.


```python
def optimize_yahtzee():
    first_roll = roll_five_dice()
    first_yahtzee_value = pd.Series(first_roll).value_counts().idxmax()
    first_roll_logic = [0 if x == first_yahtzee_value else 1 for x in first_roll]

    second_roll = reroll_dice(first_roll_logic, first_roll)
    second_yahtzee_value = pd.Series(second_roll).value_counts().idxmax()
    second_roll_logic = [0 if x == second_yahtzee_value else 1 for x in second_roll]

    final_roll = reroll_dice(second_roll_logic, second_roll)
    is_yahtzee = 1 if pd.Series(final_roll).value_counts().max() == 5 else 0

    return(is_yahtzee)

```

To play a game, we set up the `play_game()` function with parameter `nrounds` to run our `optimize_yahtzee()` function for `nrounds` turns. Since a standard Yahtzee game is 13 rolls (six upper half rolls and seven lower half rolls), we use 13 as our testing value.


```python
def play_game(nrounds = 13):
    runs = nrounds
    results = []

    i = 0

    while i < runs:
        res = optimize_yahtzee()
        results.append(res)
        i += 1

    yahtzees = sum(results)
    return(yahtzees)

```

Next, we simulate 100,000 games simply by looping over this function. 


```python
games = 100000

game_outcomes = []

j = 0

while j < games:
    game_outcome = play_game(nrounds = 13)
    game_outcomes.append(game_outcome)
    j += 1
```

As it turns out, even playing for a Yahtzee doesn't guarantee that we'll throw one. In 54 percent of games, we still don't even end up with a single Yahtzee despite designing our whole strategy around obtaining one. Hitting that all-important bonus Yahtzee category has just an 11.8 percent chance of occurring under optimal Yahtzee conditions. And *two* bonus Yahtzees or more in a single game? That has just under a 2 percent chance of occurring for a single player in a single game. 

In a six-person game like the one where my friend threw two bonus Yahtzees, the odds of at least one player achieving that feat is 11.4 percent — unlikely, but not as improbable as it seemed. You would need 35 players in a single Yahtzee game for there to be at least a 50 percent chance of one player throwing two bonus Yahtzees. 


```python
game_series = pd.Series(game_outcomes)
yahtzee_counts = game_series.value_counts() / games
yahtzee_counts
```




    0    0.54270
    1    0.33957
    2    0.09781
    3    0.01771
    4    0.00190
    5    0.00029
    6    0.00002
    Name: count, dtype: float64




```python
plt.bar(yahtzee_counts.index, yahtzee_counts)
plt.xlabel("# of Yahtzees Recorded in 13 Turns")
plt.ylabel("Number of Occurrences in 100,000 Sims")
plt.title("Expected Yahtzees When Playing With a Yahtzee Maximization Strategy")
```




    Text(0.5, 1.0, 'Expected Yahtzees When Playing With a Yahtzee Maximization Strategy')




    
![png](yahtzee_maximization_files/yahtzee_maximization_12_1.png)
    


Just for fun, a nice result from these simulations is a heuristic for the odds of throwing a Yahtzee on any given turn provided that you're rolling for a Yahtzee. This works out to about 4.5 percent, which is more likely than throwing five heads in a row with a fair coin. And hypothetically after 15 maximum-Yahtzee rolls in-game, you'd have a 50 percent chance to throw at least one. However, there's also a 10 percent chance you'll go 50 rolls without a single Yahtzee. I'm sure plenty of people can relate. 


```python
turns = 100000

turn_outcomes = []

j = 0

while j < turns:
    turn_outcome = play_game(nrounds = 1)
    turn_outcomes.append(turn_outcome)
    j += 1

turn_series = pd.Series(turn_outcomes)
turn_counts = turn_series.value_counts() / turns
turn_counts
```




    0    0.95466
    1    0.04534
    Name: count, dtype: float64


