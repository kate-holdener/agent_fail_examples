# Tourney - Chess tournament management software

Repo: 

The software is used to pair up players in chess tournaments. In swiss style tournament, the pairing rules are complex, and the software calls an open source tool `bbpPairing` to get the pairings for next round. It also keeps track of players' overall scores, so that the tournament director can figure out the winners.

## Problem 1: Duplicated code

Players' standings in a tournament depends on their scores. Score is a sum of a player's points, and points are awearded for wins and draws (and sometimes byes). Sometimes, multiple players will end up with the same score at the end of the tournament, so tie breaker rules are applied. There are various ways to calculate tie breakers, and sometimes you need multiple tie breakers.

Agent: Claude Code
Model: Sonet 4.6

I asked the agent to implement tie breaker mechanisms, one at a time. All tie breakers have the same interface: a score function that takes a player id and a round number and returns a float. The return value is a tie breaker score (the higher the better). When the agent was implementing the third tie breaker, I noticed that it added the following code to that specific tie breaker algorithm:
```
   WIN = 1.0                                         
   DRAW = 0.5
   LOSS = 0.0                                        
   PAIRING_ALLOCATED_BYE = 1.0
   HALF_POINT_BYE = 0.5
```

Those are the default points associated with different game outcomes. I thought it was strange that the agent was adding these constants to the 3rd tie breaker implementation (certainly, these constants had been added already). I asked the agent if it just duplicated this code, and it admitted that it did. 

The agent then fixed the problem by, moving these constants into a reusable component, and updating the relevant code. However, if I didn't call it out on this, it would have left the duplicated values in place. 

The project needs to support configurable point values (it didn't yet, at this point). The duplication of default values could have led to hidden bugs, when configurable point values are added.

## Problem 2: More duplication

Upon discovering that Claude duplicated the constants, I asked it to review the code it recently added for calculating the standings for more duplications. It did a review and found two problems:
```
1. _round_complete is defined in both buchholz.py and get_standings.py independently — get_standings.py should not have its own copy.   
2. _player_scores is defined in buchholz.py but sonneborn_berger.py imports it from there — a tie-breaker importing internals 
   from another tie-breaker is wrong. It should live somewhere both can import from cleanly. 
Let me check the Round entity before deciding where _round_complete belongs: 
Round.is_complete already exists. _round_complete(round_) is a free-function duplicate of it. 
And _player_scores should move out of buchholz.py into entity/scoring.py since it's shared scoring logic.
```

I reviewed Claude's changes - indeed, it updated the entity/scoring.py file to have a function for calculating player's score. However, the cumulative tie breaker had very similar code - it was calculating each player's running score at each round. So same logic, but slightly different approach: rather tan calculating the total score, it needed a running score on per round basis. So I pointed this out to Claude:
```
it seems to me that cumulative tie breaker has the same duplicated player score logic                                                   
```                                                                                     
The response was:
```
You're right. The per-round score logic in cumulative.py — handling byes, requested byes, withdrawals, and game results — 
is the same as what's inside player_scores. I can extract a score_for_round helper into entity/scoring.py and use it in both.                       
```

It proceeded to make necessary changes...

## Problem 3 - Inefficient implementation

As Claude was working on fixing problem 2, I realized that it implemented something very redundant and inefficient, so I shared my realization:
```
Since all these tie breakers are based on scores, why are we re-calculating the scores each time? 
when we compute Standings, we calculate all the scores. Can we just pass the scores table to the tie breaker implementations, and avoid re-calculations? 
```
The agent agreed with me:
```
Good catch. Currently BuchholzTieBreaker and SonnebornBergerTieBreaker each call player_scores(section) once per player,
so for a 100-player tournament they each recompute all scores 100 times. The fix is to change the interface to accept 
a pre-computed scores dict, and have _standings compute it once and pass it in.
```
