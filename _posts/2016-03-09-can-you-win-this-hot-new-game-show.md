---
layout: post
title: "Can You Win This Hot New Game Show?"
description: "Expanding on the Riddler's Problem"
output: html_document
date: 2016-03-09 12:00:00 -0500
category: r
tags: [r]
comments: true
---





So the latest [Riddler puzzle on FiveThirtyEight](http://fivethirtyeight.com/features/can-you-win-this-hot-new-game-show/) goes like this:

> Two players go on a hot new game show called “Higher Number Wins.” The two go into separate booths, and each presses a button, and a random number between zero and one appears on a screen. (At this point, neither knows the other’s number, but they do know the numbers are chosen from a standard uniform distribution.) They can choose to keep that first number, or to press the button again to discard the first number and get a second random number, which they must keep. Then, they come out of their booths and see the final number for each player on the wall. The lavish grand prize — a case full of gold bullion — is awarded to the player who kept the higher number. Which number is the optimal cutoff for players to discard their first number and choose another? Put another way, within which range should they choose to keep the first number, and within which range should they reject it and try their luck with a second number?

My initial thought was to try and solve this problem via simulation. The following code will generate 100,000 rounds of play between two players. For the sake of efficiency, we will draw twice for each player now and consider how the selection of the first or second number influences the outcome of the game.


{% highlight r %}
require(dplyr)
require(magrittr)
require(ggplot2)

set.seed(938747)

# number of rounds to play
n_draw <- 100000
{% endhighlight %}


{% highlight r %}
p1 <- cbind(runif(n_draw), runif(n_draw))
p2 <- cbind(runif(n_draw), runif(n_draw))

head(p1)
{% endhighlight %}



{% highlight text %}
##        [,1]    [,2]
## [1,] 0.5322 0.05078
## [2,] 0.8662 0.60570
## [3,] 0.1850 0.21139
## [4,] 0.1196 0.26063
## [5,] 0.8719 0.47793
## [6,] 0.2358 0.38871
{% endhighlight %}



{% highlight r %}
head(p2)
{% endhighlight %}



{% highlight text %}
##        [,1]   [,2]
## [1,] 0.8189 0.7178
## [2,] 0.4383 0.2433
## [3,] 0.6300 0.5607
## [4,] 0.7649 0.4286
## [5,] 0.8377 0.3405
## [6,] 0.4543 0.7741
{% endhighlight %}

## Naive Players

The key thing to realize is that if the players adopt the same strategy, each player will always have a 50% chance of victory. Let's consider a very simple strategy first: no matter what happens, always keep the first number. After all, the number is generated randomly so there is no guarantee that the second number will be larger than the first number. How well does player 1 do if both players employ this strategy?


{% highlight r %}
sum(p1[,1] > p2[,1]) / n_draw
{% endhighlight %}



{% highlight text %}
## [1] 0.4985
{% endhighlight %}

With this strategy, each player has a roughly 50% chance of winning the game. We can also see this is the same if each player always takes the second number.


{% highlight r %}
sum(p1[,2] > p2[,2]) / n_draw
{% endhighlight %}



{% highlight text %}
## [1] 0.5031
{% endhighlight %}

Okay, so this strategy is a bit naive. Why not be more sophisticated? Perhaps instead, players will only take the second number if their first number is low. An arbitrary cutpoint might be .5. That is, if either player gets a number less than .5 in the first draw, they will take whatever they get in the second draw. How does this strategy work? Here I write a short function to select the appropriate value from each round based on a specified cutpoint.


{% highlight r %}
cutoff <- function(draw, cutpoint = .5){
  ifelse(draw[, 1] < cutpoint, draw[, 2], draw[, 1])
}

sum(cutoff(p1) > cutoff(p2)) / n_draw
{% endhighlight %}



{% highlight text %}
## [1] 0.4994
{% endhighlight %}

Because each player chose a cutpoint of .5, their probability of winning is still 50%.

This applies to all cutpoints, as long as both players select the same cutpoint.


{% highlight r %}
cutpoints <- seq(0, 1, by = .01)
results <- lapply(cutpoints,
                  function(cutpoint) sum(cutoff(p1, cutpoint = cutpoint) >
                                           cutoff(p2, cutpoint = cutpoint))) %>%
  unlist

results_full <- data_frame(cutpoint = cutpoints,
                           win = results,
                           win_prop = win / n_draw)
results_full
{% endhighlight %}



{% highlight text %}
## Source: local data frame [101 x 3]
## 
##    cutpoint   win win_prop
##       (dbl) (int)    (dbl)
## 1      0.00 49849   0.4985
## 2      0.01 49846   0.4985
## 3      0.02 49897   0.4990
## 4      0.03 49896   0.4990
## 5      0.04 49923   0.4992
## 6      0.05 49930   0.4993
## 7      0.06 49959   0.4996
## 8      0.07 49965   0.4996
## 9      0.08 50011   0.5001
## 10     0.09 50051   0.5005
## ..      ...   ...      ...
{% endhighlight %}


{% highlight r %}
ggplot(results_full, aes(cutpoint, win_prop)) +
  geom_line() +
  scale_y_continuous(labels = scales::percent, limits = c(0, 1)) +
  labs(x = "Cutpoint",
       y = "Player 1 Win Percentage") +
  theme_bw(base_size = 16)
{% endhighlight %}

![center](/figs/2016-03-09-can-you-win-this-hot-new-game-show/cut_plot-1.png)

## Strategic Players

Okay, so the issue with this analysis thus far is that this assumes the same strategies by both players. What would happen if two players do not use the same cutpoint? After all, the players cannot communicate with one another so there is no reason to expect that they would choose the same cutpoint. Let's consider what happens if player 1 takes the second number if her first number is less than .5, but player 2 only takes the second number if his first number is less than .9.


{% highlight r %}
cutoff_2 <- function(p1, p2, cut1 = .5, cut2 = .5){
  sum(cutoff(p1, cutpoint = cut1) > cutoff(p2, cutpoint = cut2))
}

cutoff_2(p1, p2, cut1 = .5, cut2 = .9) / n_draw
{% endhighlight %}



{% highlight text %}
## [1] 0.5713
{% endhighlight %}

Hmm, now we're on to something. Player 1 wins 57% of the rounds. But what if Player 2 knows this and adjusts his cutpoint? And Player 1, expecting this, adjusts her's? Essentially, we want to determine the equilibrium strategy for this game.

### Analytical Solution

So full disclosure: I didn't do the math on this one. md46135 did the calculus and the full explanation can be found [here](http://forumserver.twoplustwo.com/25/probability/riddler-1594214/#post49516577). The full probability equation is as follows (here, player 1 is named **hero** and player 2 is named **villain**):

[$$\Pr(H = 1) = - \frac{1}{2}(h^2 - 1)v + h \left( - \frac{h^2}{2} + (h - 1)v + \frac{1}{2} \right) + \frac{hv}{2} - \frac{v^2}{2} + (v - 1) v + \frac{1}{2}$$](https://www.wolframalpha.com/input/?i=(h*v)%2F2+%2B+v*(v+-+1)+-+v%5E2%2F2+%2B+h*(v*(h+-+1)+-+h%5E2%2F2+%2B+1%2F2)+-+(v*(h%5E2+-+1))%2F2+%2B+1%2F2)

This is the joint probability of victory for player 1 (or the hero) given four potential states: hero and villain both keep their first numbers, hero keeps her first and villian keeps his second, hero keeps her second and villain keeps his first, and both hero and villain keep their second numbers. In order to calculate the equilibrium strategy for both players, calculate the partial derivatives with respect to *h* and *v*:

[$$\frac{\partial \Pr(H = 1)}{\partial h} = \frac{1}{2} (-3h^2 + 2hv -v + 1)$$](https://www.wolframalpha.com/input/?i=(h*v)%2F2+%2B+v*(v+-+1)+-+v%5E2%2F2+%2B+h*(v*(h+-+1)+-+h%5E2%2F2+%2B+1%2F2)+-+(v*(h%5E2+-+1))%2F2+%2B+1%2F2+partial+derivative+with+respect+to+h)

[$$\frac{\partial \Pr(H = 1)}{\partial v} = \frac{1}{2} (h^2 - h + 2v - 1)$$](https://www.wolframalpha.com/input/?i=(h*v)%2F2+%2B+v*(v+-+1)+-+v%5E2%2F2+%2B+h*(v*(h+-+1)+-+h%5E2%2F2+%2B+1%2F2)+-+(v*(h%5E2+-+1))%2F2+%2B+1%2F2+partial+derivative+with+respect+to+v)

Then set each derivative equal to 0, and solve for the appropriate values of *h* and *v*.

[$$h = \frac{\sqrt{5}}{2} - \frac{1}{2}, v = \frac{\sqrt{5}}{2} - \frac{1}{2} $$](https://www.wolframalpha.com/input/?i=v%2F2+-+h*v+-+h*(h+-+v)+%2B+v*(h+-+1)+-+h%5E2%2F2+%2B+1%2F2%3D0,++h%2F2+%2B+v+%2B+h*(h+-+1)+-+h%5E2%2F2+-+1%2F2%3D0)

### Simulation Solution

We can also use R to find this solution via [Monte Carlo simulation](https://en.wikipedia.org/wiki/Monte_Carlo_method). This I solved for myself. Essentially we do a [grid search](https://en.wikipedia.org/wiki/Hyperparameter_optimization#Grid_search) over possible combinations of two player cutpoints and find the values which leave each player winning as close to 50% of the rounds as possible.


{% highlight r %}
cut_lite <- seq(0, 1, by = .025)

cut_combo <- expand.grid(cut_lite, cut_lite) %>%
  tbl_df %>%
  rename(cut1 = Var1, cut2 = Var2) %>%
  mutate(p1_win = apply(., 1, FUN = function(x) cutoff_2(p1, p2, cut1 = x[1], cut2 = x[2])),
         p1_prop = p1_win / n_draw)
cut_combo
{% endhighlight %}



{% highlight text %}
## Source: local data frame [1,681 x 4]
## 
##     cut1  cut2 p1_win p1_prop
##    (dbl) (dbl)  (int)   (dbl)
## 1  0.000     0  49849  0.4985
## 2  0.025     0  51071  0.5107
## 3  0.050     0  52301  0.5230
## 4  0.075     0  53432  0.5343
## 5  0.100     0  54427  0.5443
## 6  0.125     0  55381  0.5538
## 7  0.150     0  56270  0.5627
## 8  0.175     0  57164  0.5716
## 9  0.200     0  57962  0.5796
## 10 0.225     0  58657  0.5866
## ..   ...   ...    ...     ...
{% endhighlight %}


{% highlight r %}
ggplot(cut_combo, aes(cut1, cut2, fill = p1_prop)) +
  geom_raster() +
  # geom_line(data = results_full, aes(cutpoint, cutpoint, fill = NULL)) +
  geom_vline(xintercept = sqrt(5) / 2 - (1 / 2), linetype = 2, alpha = .5) +
  geom_hline(yintercept = sqrt(5) / 2 - (1 / 2), linetype = 2, alpha = .5) +
  geom_point(data = data_frame(x = sqrt(5) / 2 - (1 / 2),
                               y = sqrt(5) / 2 - (1 / 2)),
             aes(x, y, fill = NULL)) +
  scale_fill_gradient2(midpoint = .5, labels = scales::percent) +
  labs(x = "Player 1 Cutpoint",
       y = "Player 2 Cutpoint",
       fill = "Player 1 Win\nPercentage") +
  theme_bw(base_size = 16) +
  theme(legend.position = "bottom",
        legend.text = element_text(size = 8))
{% endhighlight %}

![center](/figs/2016-03-09-can-you-win-this-hot-new-game-show/vary_cut_plot-1.png)

As can be seen above, if either player adjusts his or her strategy, then one of them will be more successful in the long run. In order to maintain the balance, each player should discard their first number if it is less than approximately 0.618.





