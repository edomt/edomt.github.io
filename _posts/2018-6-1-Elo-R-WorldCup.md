---
layout: post
title: How to rank the 32 teams in the 2018 FIFA World Cup with R and the 'elo' package
excerpt_separator: <!--more-->
---

The 2018 World Cup is upon us! If you're tempted to do a little betting,
or you're taking part in a friendly forecast competition with friends or
colleagues, read on. In this tutorial, we'll learn how to use R and the
'elo' package to create Elo rankings for the 32 teams in the tournament,
and how to use those rankings to predict the result of football matches.

<!--more-->

What is Elo?
------------

In its most basic definition, Elo is a rating system which lets us rank
teams (or players in individual sports or games) relative to one
another, and predict the likely outcome of a given match-up. I won't try
to explain it better than Wikipedia:

> The Elo rating system is a method for calculating the relative skill
> levels of players in zero-sum games such as chess. It is named after its
> creator Arpad Elo, a Hungarian-American physics professor.
> 
> The Elo system was originally invented as an improved chess rating
> system over the previously used Harkness system, but is also used as a
> rating system for multiplayer competition in a number of video games,
> association football, American football, basketball, Major League
> Baseball, Scrabble, board games such as Diplomacy and other games.
> 
> The difference in the ratings between two players serves as a predictor
> of the outcome of a match. Two players with equal ratings who play
> against each other are expected to score an equal number of wins. A
> player whose rating is 100 points greater than their opponent's is
> expected to score 64%; if the difference is 200 points, then the
> expected score for the stronger player is 76%.
> 
> A player's Elo rating is represented by a number which increases or
> decreases depending on the outcome of games between rated players. After
> every game, the winning player takes points from the losing one. The
> difference between the ratings of the winner and loser determines the
> total number of points gained or lost after a game. In a series of games
> between a high-rated player and a low-rated player, the high-rated
> player is expected to score more wins. If the high-rated player wins,
> then only a few rating points will be taken from the low-rated player.
> However, if the lower rated player scores an upset win, many rating
> points will be transferred. The lower rated player will also gain a few
> points from the higher rated player in the event of a draw. This means
> that this rating system is self-correcting. A player whose rating is too
> low should, in the long run, do better than the rating system predicts,
> and thus gain rating points until the rating reflects their true playing
> strength.

Interestingly, the Elo ranking is featured in a [scene of the movie *The Social
Network*](https://www.youtube.com/watch?v=10AeyTCeZJM), where Mark Zuckerberg and Eduardo Saverin discuss how to rank
women on [Facemash](https://en.wikipedia.org/wiki/History_of_Facebook#FaceMash). (While the
scene is a nice introduction to Elo, the idea that Zuckerberg would need
his roommate to "give him the algorithm" – as opposed to just looking it
up himself, say, on the Internet – is a bit naive. But anyway.)

[![Elo in The Social Network](https://raw.githubusercontent.com/edomt/edomt.github.io/master/images/social-network1.jpg)](https://www.youtube.com/watch?v=10AeyTCeZJM)

Getting our data
----------------

In order to build reliable Elo rankings, we'll need a good dataset of
historical results for the 32 teams in the World Cup, going as far back
as possible.

Thankfully, an awesome person named Mart Jürisoo is maintaining [such a
dataset on Kaggle.com](https://www.kaggle.com/martj42/international-football-results-from-1872-to-2017). (You'll need to create an account on Kaggle to download the
dataset.)

I will be using the [dplyr package](https://dplyr.tidyverse.org/) a lot throughout this tutorial. If
you've never used dplyr before, you really should! It will greatly
simplify your data manipulations and analyses.

```r
library(dplyr)
matches <- readr::read_csv('results.csv')
```

As of 1 June 2018, this dataset includes data on 38,949 international
football matches from 30 November 1872 to 28 May 2018.

The data is easy enough to understand. There are only 9 variables, with
self-explanatory names: date, home\_team, away\_team, home\_score,
away\_score, tournament, city, country, neutral (whether the match was
played on neutral ground, or at the home team's stadium).

To illustrate how this looks, here's an anecdote: the current record for
most goals in an international match belongs to [Australia vs. American
Samoa in 2001](https://en.wikipedia.org/wiki/Australia_31%E2%80%930_American_Samoa), with a cruel 31-0.

```r
matches %>%
  select(date, home_team, away_team, home_score, away_score) %>%
  mutate(total_goals = home_score + away_score) %>%
  arrange(-total_goals) %>%
  head(1)
```

	##         date home_team      away_team home_score away_score total_goals
	## 1 2001-04-11 Australia American Samoa         31          0          31

Preparing our data
------------------

Our historical data is stored in `matches`, but we'll need to create
another data.frame separately, to store each team's Elo rating, and
update it after each match.

```r
teams <- data.frame(team = unique(c(matches$home_team, matches$away_team)))
```

To start our Elo ratings, we need to assign an initial Elo value to all
the teams in our dataset. Traditionally in Elo-based systems this
initial value is set to 1500.

```r
teams <- teams %>%
  mutate(elo = 1500)
```

For each match, we'll also create a variable that tells us who won.
Because of how the 'elo' package works, this variable will take the
following values:

- `1` if the home team won;
- `0` if the away team won;
- `0.5` for a draw.

```r
matches <- matches %>%
  mutate(result = if_else(home_score > away_score, 1,
                          if_else(home_score == away_score, 0.5, 0)))
```

We can also get rid of a bunch of variables we won't need, and make sure
that our historical data is ordered by date.

```r
matches <- matches %>%
  select(date, home_team, away_team, result) %>%
  arrange(date)
```

Here's what our two datasets look like now:

```r
head(matches)
```

    ##         date home_team away_team result
    ## 1 1872-11-30  Scotland   England    0.5
    ## 2 1873-03-08   England  Scotland    1.0
    ## 3 1874-03-07  Scotland   England    1.0
    ## 4 1875-03-06   England  Scotland    0.5
    ## 5 1876-03-04  Scotland   England    1.0
    ## 6 1876-03-25  Scotland     Wales    1.0

```r
head(teams)
```

    ##               team  elo
    ## 1         Scotland 1500
    ## 2          England 1500
    ## 3            Wales 1500
    ## 4 Northern Ireland 1500
    ## 5              USA 1500
    ## 6          Uruguay 1500

The 'elo' package
-----------------

Implementing the Elo algorithm from scratch would be a bit long and
complex, but as for so many others things in life, there is an R package
for that!

```r
library(elo)
```

We'll only be using one function from this package to create our
rankings: `elo.calc()`. This function takes 4 arguments:

-   `wins.A`: whether team A won or not. This is what we've created and
    stored in our `result` variable, with 3 possibles values (1, 0,
    0.5);
-   `elo.A`: the pre-match Elo value for team A;
-   `elo.B`: the pre-match Elo value for team B;
-   `k`: this is called the K-factor. This is basically how many Elo
    points are up for grabs in each match. Make this too small (e.g. 1)
    and each match will have almost no effect on the rankings. Make it
    too large (e.g. 100) and each match will completely change the
    rankings. The Wikipedia page on Elo ratings has an [entire section on
    this](https://en.wikipedia.org/wiki/Elo_rating_system#Most_accurate_K-factor).
    Based on what's written in there, a value of 30 seems reasonable for
    our purposes.

Computing the ratings
---------------------

Now, let's write our program. The idea is to loop over each game in
`matches`, get the pre-match ratings for both teams, and update them
based on the result. We'll get two new ratings, which we'll use to
update our data in `teams`.

```r
for (i in seq_len(nrow(matches))) {
  match <- matches[i, ]
  
  # Pre-match ratings
  teamA_elo <- subset(teams, team == match$home_team)$elo
  teamB_elo <- subset(teams, team == match$away_team)$elo
  
  # Let's update our ratings
  new_elo <- elo.calc(wins.A = match$result,
                      elo.A = teamA_elo,
                      elo.B = teamB_elo,
                      k = 30)
  
  # The results come back as a data.frame
  # with team A's new rating in row 1 / column 1
  # and team B's new rating in row 1 / column 2
  teamA_new_elo <- new_elo[1, 1]
  teamB_new_elo <- new_elo[1, 2]
  
  # We then update the ratings for teams A and B
  # and leave the other teams as they were
  teams <- teams %>%
    mutate(elo = if_else(team == match$home_team, teamA_new_elo,
                         if_else(team == match$away_team, teamB_new_elo, elo)))
}
```

After a few minutes, you should get a nice `teams` data.frame, with the
most up-to-date international Elo ratings for June 2018.

```r
teams %>%
  arrange(-elo) %>%
  head
```

    ##        team      elo
    ## 1    Brazil 2032.956
    ## 2     Spain 1975.339
    ## 3   Germany 1958.013
    ## 4    France 1937.242
    ## 5 Argentina 1920.864
    ## 6   England 1906.218

The 2018 World Cup
------------------

We still have to do one thing: subset our `teams` data.frame to only
keep the 32 teams that have qualified for the 2018 World Cup.

```r
WC_teams <- teams %>%
  filter(team %in% c("Russia", "Germany", "Brazil", "Portugal", "Argentina", "Belgium",
                     "Poland", "France", "Spain", "Peru", "Switzerland", "England",
                     "Colombia", "Mexico", "Uruguay", "Croatia", "Denmark", "Iceland",
                     "Costa Rica", "Sweden", "Tunisia", "Egypt", "Senegal", "Iran",
                     "Serbia", "Nigeria", "Australia", "Japan", "Morocco", "Panama",
                     "Korea Republic", "Saudi Arabia")) %>%
  arrange(-elo)
```

Finally, here are our World Cup Elo rankings, from strongest to weakeast team:

```r
print.data.frame(WC_teams)
```

    ##              team      elo
    ## 1          Brazil 2032.956
    ## 2           Spain 1975.339
    ## 3         Germany 1958.013
    ## 4          France 1937.242
    ## 5       Argentina 1920.864
    ## 6         England 1906.218
    ## 7         Belgium 1890.040
    ## 8        Portugal 1883.412
    ## 9        Colombia 1865.608
    ## 10           Peru 1854.196
    ## 11         Mexico 1834.088
    ## 12    Switzerland 1825.958
    ## 13        Croatia 1816.654
    ## 14        Uruguay 1814.028
    ## 15           Iran 1808.954
    ## 16         Poland 1795.867
    ## 17        Denmark 1755.689
    ## 18        Senegal 1752.168
    ## 19 Korea Republic 1748.506
    ## 20          Japan 1747.581
    ## 21         Sweden 1739.752
    ## 22      Australia 1733.484
    ## 23         Russia 1732.650
    ## 24        Morocco 1732.194
    ## 25         Serbia 1725.170
    ## 26     Costa Rica 1723.551
    ## 27        Iceland 1708.682
    ## 28        Nigeria 1705.434
    ## 29          Egypt 1695.344
    ## 30        Tunisia 1691.193
    ## 31         Panama 1682.273
    ## 32   Saudi Arabia 1653.511

We can also look up which teams didn't make it to the World Cup this
year, despite high Elo ratings:

```r
teams %>%
  filter(elo > 1800, !team %in% WC_teams$team)
```

    ##          team      elo
    ## 1         USA 1808.151
    ## 2 Netherlands 1847.749
    ## 3       Italy 1846.751
    ## 4       Chile 1832.832

Going further
-------------

If you want to use this data for predictions and forecast competitions,
here are two things you can do:

### 1. Calculating probabilities for individual matches

In the 'elo' package, the `elo.prob()` function lets you calculate the
probability that team A will win a match against team B, given their
respective Elo ratings.

For example, in the opening match of the competition (Russia vs. Saudi
Arabia), the probability of Russia winning would be 61%:

```r
russia <- subset(WC_teams, team == "Russia")$elo
saudi_arabia <- subset(WC_teams, team == "Saudi Arabia")$elo
elo.prob(russia, saudi_arabia)
```

    ## [1] 0.611961

Two more examples:

```r
# A balanced match-up: France vs. Argentina
france <- subset(WC_teams, team == "France")$elo
argentina <- subset(WC_teams, team == "Argentina")$elo
elo.prob(france, argentina)
```

    ## [1] 0.523551

```r
# A very un-balanced one: Brazil vs. Iceland
brazil <- subset(WC_teams, team == "Brazil")$elo
iceland <- subset(WC_teams, team == "Iceland")$elo
elo.prob(brazil, iceland)
```

    ## [1] 0.8660723

### 2. Simulating the entire competition

I won't show the details of this here because the code would be much
longer, but essentially you can use the probability generated by
`elo.prob()` to simulate the outcome of each match (using
the `sample()` function and its `prob` argument to choose a random
winner between Russia and Saudi Arabia, but with a 61% probability of
choosing Russia), and update the Elo ratings throughout the competition.

This way, you can simulate the entire competition all the way from the
group stage to the final. And if you repeat this process many (thousands
of) times, you will get detailed probabilities for each team to make
it to the each stage of the competition. This is essentially what
websites like FiveThirtyEight do for [their sport predictions](https://projects.fivethirtyeight.com/2018-mlb-predictions/), with
probabilities based on 100,000 simulations of the rest of the season.
