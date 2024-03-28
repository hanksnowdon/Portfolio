+++
title = 'Fantasy Baseball Roto Draft Optimization'
date = 2024-03-19T16:48:05-05:00 
draft = false
summary = 'Using integer programming to select an optimal fantasy baseball roster.'
tags = ["Sports", "Fantasy Baseball"]
categories = ["Sports"]
+++


## Introduction

I recently joined a Gladiator-Style (draft and hold) rotisserie fantasy baseball league hosted by the smart people at The Athletic's Rates & Barrels podcast. After a quick glance at the rules, I immediately knew I wanted to employ an optimization algorithm to choose my team.

The league rules are as follows:

- Each team sets their roster before the season starts, and cannot change it again.
- Any player can be chosen, provided that you stay under the "$280" budget (Fantrax, where the league is hosted, assigns an auction value to every player. Ronald Acuna Jr. costs $56, for example).
- Teams must roster 2 C, 1 1B, 1 2B, 1 3B, 1 SS, 2 CI, 2 MI, 6 OF, 2 UT, and 13 P.
- Scoring is 5x5 roto: HR, RBI, R, SB, AVG, IP, SV, K, ERA, WHIP

Since I can choose whoever I want for my roster and don't have to contend with other team's selecting players in a draft setting, this is a prime opportunity for an optimization algorithm.

## Determining an Approach to Pick an Optimal Team

The most common approach to drafting in roto leagues is to build a balanced team with high Z-scores across all scoring categories, and that's the angle I'm taking when constructing this roster. I won't go into length regarding exactly how this works mathematically, but the general idea is to field a roster of players who are maximally above-average in aggregate across all categories. This Fangraphs article is a great jumping off point into the world of auction-valuing players for FBB: (<https://fantasy.fangraphs.com/using-the-auction-calculator-in-2022-a-beginners-guide/>).

To calculate the Z-scores for each player in each category I use the (discussed in the article above) Fangraphs Auction Calculator. This great tool uses 2024 Steamer projections to generate the Z-scores I need, and is exportable via CSV. After a little Excel magic to merge in the player prices from Fantrax and to add Utility to each hitters position, CI to 1B and 3Bs, and MI to 2B and SSs, I have a ready-to-use dataset:

![alt](data.png)

## Structuring the Optimization Problem

With the data wrangled, it's time to set up the optimization problem. I use Julia to code up the problem and solve via the GLPK solver.

### Specifying Model Variables

The first step is to specify the decision variables. In the case of this problem, there will be n (number of players) * m (number of positions) binary variables, each corresponding to whether a player is selected for my roster at a specific position.

```julia
# Create position dictionary
position_requirements = Dict(
    "C" => 2,
    "1B" => 1,
    "2B" => 1,
    "3B" => 1,
    "SS" => 1,
    "CI" => 2,
    "MI" => 2,
    "OF" => 6,
    "UT" => 2,
    "P" => 13
)
# Specify decision variables
@variable(model, x[1:length(players), 1:length(position_requirements)], Bin)
```

### Constraints for Player Selection

The first constraint is the available budget to spend: $280.

```julia
budget = 280
# Constraint: budget constraint
@constraint(model, sum(x[i, j] * players[i].price for i in 1:length(players), j in 1:length(position_requirements)) <= budget)
```

There must also be a set number of players per position:

```julia
# Constraint: positional requirements
for (j, (position, requirement)) in enumerate(position_requirements)
    @constraint(model, sum(x[i, j] for i in 1:length(players)) == requirement)
end
```

Players must only be allowed to occupy positions they are eligible for:

```julia
# Constraint: player positional eligibility 
for i in 1:length(players)
    for (j, (position, _)) in enumerate(position_requirements)
        if position in players[i].positions
            @constraint(model, x[i, j] <= 1)
        else
            @constraint(model, x[i, j] == 0)
        end
    end
end
```

Finally, each player can only be selected once:

```julia
# Constraint: each player can only be selected once
for i in 1:length(players)
    @constraint(model, sum(x[i, j] for j in 1:length(position_requirements)) <= 1)
end
```

### Defining the Objective Function

The objective function I use here is simple: maximize total Z-score across all ten categories. This obviously potentially contradicts my goal of the project: to create the best and **most balanced** roster possible. In a league like this I want to pick a team which won't kill me in any one category, and can be very strong in as many as possible. But instead of building this into the objective function, I will instead take the easier approach of tweaking the constraints later to prevent any one category from being too strong or weak.

```julia
# Objective function: maximize sum of Z scores
@objective(model, Max, 
    sum(x[i, j] * (players[i].AVG + players[i].RBI + players[i].R + players[i].SB + players[i].HR + players[i].SV + players[i].ERA + players[i].WHIP + players[i].SO + players[i].IP) 
        for i in 1:length(players), j in 1:length(position_requirements))
)
```

## Results -- Tweaking Constraints to Pick the Best Team

Running the optimization problem yields the following initial roster:

![alt](p1.png)
![alt](h1.png)

A few observations:

- The model only selects two players who are projected to accrue any saves: Mason Miller and Trevor Megill. I'll need to constrain the results ensure more closers are selected, as I don't think these two players will be enough to rank well in the saves category.
- This roster seriously hunts batting average, likely to a fault. While the sums of Z scores aren't directly comparable across categories (for a variety of reasons), I want to generally avoid any one category from being way stronger or weaker than another. While overall "sum of Z scores" will decrease, my team's final rank will be more likely to be better if I try to spread the wealth across categories.

With this in mind, I'll first constrain AVG Z-score to have an upper limit of 15:

![alt](p2.png)
![alt](h2.png)

This roster seems slightly more balanced and the total sum of Z scores decreased by less than 2, but the pitching still seems too weak. Half the scoring categories are won by pitchers, and I need to make that group stronger.

I'll start by supplying a lower limit for SV Z-score of -2:

![alt](p3.png)
![alt](h3.png)

- The model responds to this new constraint by adding vaunted closer Pete Fairbanks and setup man Caleb Ferguson.
- While Fairbanks is a good add, I don't actually want any setup men on my roster. One problem with the Z-score method is that it cannot distinguish between low-ERA players who throw lots of innings (good starters) or fewer innings (good relievers). Ferguson, while having a good ERA, doesn't actually give me much because he won't throw enough innings for it to matter.

To address this, I'll now choose to overemphasize IP in my model, setting a lower bound of IP Z-score of 9:

![alt](p4.png)
![alt](h4.png)

- This roster takes another slight hit in total Z-score: I started over 90 and am now down to 83.8.
- That being said, the roster is looking much more balanced. The team has a steady supply of saves, performs well across pitching categories, and shines in every hitting category but stolen bases.

I'll make one more tweak -- setting a lower limit for ERA Z-score of 4, up from the current 3.44, just to make my pitchers a bit stronger comparatively:

![alt](p5.png)
![alt](h5.png)

This team looks great. It's balanced and strong in almost every category -- I'm not too worried about stolen bases or steals, as these categories are notoriously difficult to acquire in roto leagues, and my opponents will be facing a similar challenge.

## Concluding Thoughts

All things considered, I'm very pleased with the results of this exercise! The integer programming optimization model is able to find an optimal collection of undervalued players, and with only a little bit of tweaking through artificial constraints, I'm able to create a team that also passes the eye test.

Regardinging future improvements, I would love to incorporate injury probabilities into the selection algorithm. Success in this league will likely be overwhelmingly determined by which teams stay the healthiest, and there's no doubt that my final roster is somewhat injury-prone (Carlos Rodon, Chris Sale, Alex Cobb, etc.). Adding a constraint for minimum average injury score or building injury history into the objective function are defintiely things I will consider if I return to improve this model in years to come.

For now, though, the only thing left to do is play! I'll post again at the end of the season to update on how the team performs.

(Full code for this project can be found [here](https://github.com/hanksnowdon/PortfolioSite/blob/master/full_code_notebooks/RotoDraftOptimization.ipynb).)
