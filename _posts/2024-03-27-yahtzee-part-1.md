---
layout: post
title: Adventures in Yahtzee probability, part 1
subtitle: Or why getting three sixes is always such a pain
author: Ben Wieland
---

This is the first post in a blog series on Yahtzee strategy. 

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt

np.random.seed(4133)
```

In Yahtzee, we frequently find ourselves attempting to maximize a specific number. Especially in the late game when attempting to hit that all-important 63 mark in the upper section (equivalent to a three-of-a-kind in every category ones through sixes), an unlucky early game in terms of upper-half rolls can force painful situations where we have no choice but to go for one number entering a roll or else risk losing our 35-point upper section bonus. In this post, we use simulation to calculate the probabilities of rolling each possible set of values in an "all-or-nothing" round with a particular emphasis on the odds of obtaining a three of a kind, four of a kind, or Yahtzee. We then calculate failure rates, or the chances of not obtaining the necessary three-of-a-kind or better to stay on pace for an upper section bonus, for varying numbers of attempted rolls. 

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
    Takes as an input the string 'held_dice' and the previous roll. Rerolls all 1s and keeps all 0s.
    """
    reroll_result = previous_roll
    for i in range(len(held_dice)):
        if held_dice[i] == 1:
            reroll_result[i] = roll_dice()

    return(reroll_result)
```

Next, we write the code to optimize a Yahtzee roll for a certain number. Here, we choose sixes because they are the optimal target roll in Yahtzee due to knock-on effects from taking a high-six hand in your chance, three of a kind, or four of a kind squares. However, the probabilities would be identical regardless of the chosen target.


```python
def optimize_sixes():
    '''
    Plays a hand of Yahtzee (three rolls) while maximizing the number of sixes obtained by holding all sixes and re-rolling all other dice. 
    Returns the number of sixes obtained.
    '''
    init_roll = roll_five_dice()
    # print(init_roll)
    reroll_logic = [1 if x != 6 else 0 for x in init_roll]
    second_roll = reroll_dice(reroll_logic, init_roll)
    # print(second_roll)
    second_logic = [1 if x != 6 else 0 for x in second_roll]
    final_roll = reroll_dice(second_logic, second_roll)
    # print(final_roll)
    sixes = [1 if x == 6 else 0 for x in final_roll]

    return(sum(sixes))

    
```

Now, we simulate one million hands of Yahtzee while attempting to maximize sixes. 


```python
results = []
runs = 1000000

i = 0

while i < runs:
    res = optimize_sixes()
    results.append(res)
    i += 1

results = pd.Series(results)
counts = results.value_counts() / (runs)
```

Obtaining two of our desired roll is the most likely outcome, occurring almost 35 percent of the time. We throw at least a three of a kind 35.5 percent of the time, at least a four of a kind 10.4 percent of the time, and a Yahtzee 1.3 percent of the time.


```python
plt.bar(counts.index, counts)
plt.xlabel("# of Times Rolled")
plt.ylabel("Number of Occurrences in 1,000,000 Sims")
plt.title("Expected Outcome of Targeting a Single Number on a Yahtzee Roll")
```

    
![png](https://github.com/bbwieland/bbwieland.github.io/assets/img/yahtzee_sim_8_1.png)
    

```python
print("Probabilities of different outcomes:")
print(counts.sort_index())
```

    Probabilities of different outcomes:
    0    0.065102
    1    0.235807
    2    0.344101
    3    0.250545
    4    0.091220
    5    0.013225
    Name: count, dtype: float64



```python
print(f"Three of a Kind probability: {counts.filter([3,4,5]).sum().round(3) * 100}%")
print(f"Four of a Kind probability: {counts.filter([4,5]).sum().round(3) * 100}%")
print(f"Yahtzee probability: {counts.filter([5]).sum().round(3) * 100}%")
```

    Three of a Kind probability: 35.5%
    Four of a Kind probability: 10.4%
    Yahtzee probability: 1.3%


Now, we compute the "failure rate" of not rolling a three of a kind at least once given a set number of rolls. If we're willing to devote two rolls to maximizing a specific number, our rate of obtaining at least a three-of-a-kind in that category rises to just over 58 percent. Three rolls brings that number up to over 73 percent.


```python
success_rate = counts.filter([3,4,5]).sum()
failure_rate = 1 - success_rate
```


```python
print(f"Failure rate given 1 attempt: {failure_rate.round(3) * 100}%")
print(f"Failure rate given 2 attempts: {(failure_rate ** 2).round(3) * 100}%")
print(f"Failure rate given 3 attempts: {(failure_rate ** 3).round(3) * 100}%")
```

    Failure rate given 1 attempt: 64.5%
    Failure rate given 2 attempts: 41.6%
    Failure rate given 3 attempts: 26.8%

