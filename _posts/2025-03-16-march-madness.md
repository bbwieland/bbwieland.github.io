---
layout: post
title: My submission for March Machine Learning Mania 2025
subtitle: Efficiency ratings with extra steps
author: Ben Wieland
tags: [python, bayes]
mathjax: true
---

For my Kaggle March Madness submission this year, I planned to put together a canonical "baseline" model to predict team-level efficiency ratings based solely on pace and effiency. From there, the goal was to ensemble with a fancier model to pick up on possible interactions between different styles of play while anchoring to the baseline predictions so we didn't overfit too heavily to those style-of-play interactions.

However, time and work got the best of me, so all I managed to get around to was fitting the baseline model! However, I still think it's valuable to share my modeling approach as a reference for how to implement what is essentially Bayesian KenPom. There are all sorts of improvements that could be made on this model, but it's what I've got, so it's what I'm writing up here.

Code for the project lives in [this repository](https://github.com/bbwieland/march-machine-learning-mania-2025).

Team ratings for every team live on [this Google Sheet](https://docs.google.com/spreadsheets/d/1n8kDnzpBRsgWfMBb8dPV3YbTGfIFRopLRNXfn4gsrws/edit?usp=sharing).

# How the model works

Consider a matchup between a home and away team: $i$ and $j$. Our goal for the competition this year is simply to predict whether team $i$ beats team $j$ in a given matchup -- a problem that naturally seems to lend itself to some sort of categorical model, given the binary nature of the response variable. However, we can easily turn this into a continuous prediction problem. Since the probability of $i$ beating $j$ is equal to the probability of $i$ out-scoring $j$ and the final score of a game is a continuous variable, we can write the probability of $i$ beating $j$ as $P(S_i > S_j)$ where $S$ denotes the score.

Notationally, we'll stick with estimating the distribution of team $i$'s final score and then see that team $j$'s score can be estimated via a very similar model. In keeping with the simple efficiency-rating approach, team $i$'s modeled final score will be a function of a few key inputs related to teams $i$ and $j$:

- Each team's observed efficiency in terms of points per possession over the course of the season.

- Each team's observed pace over the course of the season.

- League-wide home-field-advantage, pace, and efficiency averages over the course of the season.

This simplicity lends itself to a very generalizable model. All we need to know to estimate the full model for a season's worth of data is information on the teams involved in each game, the final scores, the game locations, and the number of possessions.

The high-level approach will be to estimate team $i$'s offensive efficiency as a linear combination of $i$'s offense, $j$'s defense, and a home-court effect to allow for home-field advantage. Then, we scale that points-per-possession value by some number of possessions estimated as a function of $i$ and $j$'s historic pace. The last step is to account for game-to-game stochasticity: teams aren't robots who play to their true talent level every single time they take the court -- partially due to shooting variance, partially due to other forms of on-court variance, and partially just due to varying levels of play! Regardless, the better team certainly doesn't win 100 percent of the time, and we'd have a terrible model if we predicted that to be the case.

With that said, let's start to introduce some notation:

$S_i = E_i * V_i$

A team's final score is a function of its efficiency $E$ and its "volume," or pace, $V$.

$E_i \sim N(\mu_i, \sigma_i)$

The efficiency of a team in a given game is normally distributed about some average efficiency $\mu_i$. Note that we are abusing the notation of $i$ here -- we'll use $i$ to index team IDs and also, as we do when we write $E_i$, use $i$ to represent the specific matchup between teams $i$ and $j$. 

$\mu_i = \mathcal{E} + \mathcal{H} * H_i + \theta_{i,O} - \theta_{j,D}$ 

Team $i$'s expected efficiency is a function of four variables: the league-average efficiency $\mathcal{E}$, the league-average home-court advantage $\mathcal{H}$ multiplied by an indicator variable $H_i$ equal to 1 if team $i$ is playing at home and 0 otherwise, and the team-strength parameters $\theta$. 

These parameters are indexed by team -- the $i$ and $j$ -- and whether the team is on offense or defense - the $O$ and $D$ - and we define them such that a higher value is better on both sides of the court. For example, if $\theta_{i,O} = 0.1$ then team $i$ is 10 points better offensively per 100 possessions than the average team and if $\theta_{j, D} = 0.1$ then team $j$ is 10 points better defensively per 100 possessions than the average team. 

For estimating the efficiency of team $j$, we can simply flip the sign before the home-court advantage effect.

$V_i = \mathcal{V} * \omega_{i} * \omega_{j}$ 

The average pace for a given game is a function of league-average pace $\mathcal{V}$ as well as multiplicative pace effects $\omega$ estimated at the team level. These are multiplicative and not additive based on some public research done on college basketball pace modeling, but could very easily be set up identically to our $\theta$ effects above.

$\sigma_i = \sigma_{eff.} * (\frac{V}{\mathcal{V_i}})$

The game-level variance in expected value is a function of game-level efficiency, along with the ratio of expected possessions to the league-wide number of expected possessions. This is a function of the fact that as more possessions occur in a game, it becoems more likely that each team's efficiency will converge to their "expected" efficiency and a reminder that good teams should try to play fast to avoid upsets -- a fact my Virginia Cavaliers would be wise to internalize.

From there, we simply need to set priors on each parameter we defined within the model.

We estimate the team-level parameters via *partial pooling*, a common strategy when estimating hierarchical models which "learns" the variance of a set of parameters via a "hyperparameter" learned as a function of the data. For estimating team strengths, this hyperparameter will allow us to dynamically determine how much efficiency varies across teams.

- $\theta_{O} \sim N(0, \eta_{off.})$ for each team.

- $\theta_{D} \sim N(0, \eta_{def.})$ for each team.

- $\omega \sim HalfNormal(1, \rho_{pace})$ for each team. Note that we need a half-normal because these effects are multiplicative.

We use standard Half-Cauchy priors on the hyperparameters:

- $\eta_{off.} \sim HalfCauchy(1)$

- $\eta_{off.} \sim HalfCauchy(1)$

- $\rho_{pace} \sim HalfCauchy(1)$

Finally, we set priors on some of the "average" league-wide effects. These were just guesses based on my knowledge of college basketball coming into the project:

- $\mathcal{E} \sim N(1, 0.1)$: league-average efficiency.

- $\mathcal{H} \sim N(0.03, 0.015)$: league-average home-court advantage.

- $\mathcal{V} \sim HalfNormal(70, 5)$: league-average possessions per game.

And that's it! We can plug this model into any probabilistic programming language and sample it, then use the posterior distribution to simulate each team matchup. I ran 1000 matchups between each team to get my final probabilities that I submitted on Kaggle, but theoretically it's very easy to run more. I used PyMC with the default sampler, 1000 burn-in draws, and 1000 posterior draws across 4 chains. Running everything on my Apple M1 chip with 8GB RAM took about 4 minutes total for the 2025 season -- 2 minutes for the men's and women's predictions, respectively -- and better specs would make it very easy to run faster. 

# Team strength estimates

By summing a team's $\theta_{O}$ and $\theta_{D}$ parameters, we can estimate their net rating. This rating can be interpreted as how many points per possession the team would outscore an average team by on a neutral court.

For ease of interpretation, I've multiplied all these values by 100 to put them on the same scale as other efficiency metrics such as KenPom. The values in the table can be interpreted as how many points per 100 possessions a team would outscore an average team by on a neutral court. To translate per-100-possession values to game-level values, as a rule of thumb you can multiply by 0.7. So, for example, Duke would be expected to outscore an average team by 34.3 * 0.7 = 24 points in a standard-length game of 70 possessions.

## Men's Top 10

| Team | Net Rating |
| ---- | ---------- |
| Duke | 34.3 |
| Auburn | 30.7 |
| Houston | 30.7 |
| Florida | 29.6 |
| Alabama | 25.6 |
| Tennessee | 25.6 |
| Gonzaga | 25.4 |
| Texas Tech | 24.2 |
| Maryland | 23.8 |
| Iowa State | 22.9 |

## Women's Top 10

| Team | Net Rating |
| ---- | ---------- |
| UConn | 54.7 |
| South Carolina | 53.8 |
| Texas | 50.0 |
| UCLA | 46.1 |
| Notre Dame | 45.5 |
| USC | 45.0 |
| Duke | 41.4 |
| TCU | 40.5 |
| LSU | 38.9 |
| Kansas State | 38.8 |

# Backtesting the model

We can calculate historic Brier scores for this model dating back to 2015. As expected, the women's tournament is easier to predict than the men's tournament given the relative lack of upsets on the women's side and the higher frequency of unequal matchups. 

| Season | M Brier | W Brier |
| -----| ------- | ------- |
| 2015 | 0.173 | 0.123
| 2016 | 0.188 | 0.153
| 2017 | 0.180 | 0.135
| 2018 | 0.202 | 0.158
| 2019 | 0.171 | 0.136
| 2021 | 0.212 | 0.156
| 2022 | 0.223 | 0.145
| 2023 | 0.215 | 0.170
| 2024 | 0.194 | 0.131

# Some final thoughts

## On the model outputs

- For the 2025 men's season, our estimated values of the $\eta$ parameters which govern the team-to-team variation in skill ratings were higher than for any previous men's season. This reflects the increasing inequality in NCAAM basketball, which plenty of pundits have speculated on as a byproduct of the transfer portal and NIL era. For what it's worth, I think this critique is spot-on.

- By contract, the $\eta$ parameters for the *women's* game, while still higher than the men's, are smaller than they've been since 2015. The playing field is more level in NCAAW than it's been for a decade.

- 2025 Duke earned the highest rating of any men's team between 2015 and 2025, very narrowly edging 2015 Kentucky. Other standout teams with net ratings above 0.3: 2015 Wisconsin, 2017 Gonzaga, 2019 Gonzaga, 2024 Houston, 2025 Auburn, 2025 Houston.

- The highest-ranked overall teams in the backtesting timeframe? Those would be 2015, 2016, and 2018 UConn on the women's side, the only three teams with estimated net ratings above 0.6 in the dataset.

## On the competition

- I've submitted two versions of my predictions here: one with the outputs as-is and one which "cheats" by over-confidently rounding all predictions to be at least 65 percent sure of the winner in each game. This makes your expected Brier score worse, but improves your tail probability of scoring a high enough submission to win the competition.

- It's a feature of the scoring format that the long-run optimal prediction model is *not* the best submission to place 1st in the competition. I believe this model setup, which is fairly canonical within the matchup-modeling literature, would outperform most submissions to the Kaggle competition in the long run given its performance relative to the expert baseline proposed on Kaggle but stands zero chance of actually winning in any given year. The longer I submit to this contest, the more I believe that the optimal way to submit is to fit a "canonical" model similar to this one or an Elo-based model and then use those outputs as inputs to a downstream deep-learning model. Alternatively, one could ensemble the "simple" predictions with a fancier model. There's some balance between novelty and overall prediction accuracy that needs to be struck such that your model is still *good* but also *different* from the other "good" models.