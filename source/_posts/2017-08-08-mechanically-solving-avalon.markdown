---
layout: post
title: "Mechanically Solving Avalon"
date: 2017-08-08 21:48:48 +0100
comments: true
categories: Avalon
---

I've been thinking for a while if it is possible for the 'good' team in Avalon to follow some optimal strategy that would always achieve victory. To simplify things this post will focus on finding an optimal strategy for 5 person Avalon when the Merlin(Commander) and the Assassin are both in play. It is important to note in this variant the 'good' team at best can only win on average 2/3 of games because the 'evil' team can always randomly pick the Merlin at the end of the game and will guess correctly 1 in 3 times. This question has also been answered before on [stackexchange](https://boardgames.stackexchange.com/questions/20476/solution-to-avalon-board-game) but we will try and answer it without crypto. But we will use something very similar to crypto and similar strategies so I'm not sure if we are adding anything useful.

The 5 person Avalon game is interesting because there are only 2 'evil' and 3 'good' people. This creates an easy mechanism to solve the game.

1. After the game has been setup everyone secretly writes down a list of people they know are good. Merlin writes down the people who knows are good. Other good people pretend to write down a list of people but instead write gibberish. 

2. The lists are shuffled.

3. There will be either 1, 2 or 3 non-gibberish lists depending on what the 'evil' people do. 

4. If there is 1 non-gibberish list the 'good' teams wins by just following the list because this is Merlin's list.

5. If there is 2 or 3 non-gibberish lists then follow one list until it fails then switch to the other list. It is only possible for at most 2 missions to fail because after a list fails it is no longer used to pick teams. If only 2 missions fail then 3 missions succeed and the good team wins.

The strategy also shows why the Assassin is so important in the 5 player variant. Because without the Assassin the Commander could just announce themselves and even if the evil team optimally bluffed and impersonated the Commander themselves they would still be guaranteed to lose.

This strategy only works when there are 2 'evil' players so only in the 5 and 6 player variants. Once a third 'evil' player is introduced it is possible to fail 3 missions using 3 evil lists.

Can we solve 7+ player games using this strategy? 

If we have 3 evil players then we need to eliminate a list initially or run a mission that will eliminate two lists instead of one.

Unlike the stackexchange solution we don't have a way to identify the author of a list so some of the strategies for eliminating lists do not work. However, we can still:

1. If a mission fails then any list that contains all the members of the failing mission is also a bad list.

2. If a mission fails and none of the members of the failing mission is in the compliment of a list then this list is a bad list.

3. Any members common to all lists are good.

To test whether evil players have a winning strategy it should be possible to brute force all the combinations of evil lists: 7C4^3 + 7C4^2 + 7C4 => 44135 and see if any of them can't be solved by an optimal good team.

