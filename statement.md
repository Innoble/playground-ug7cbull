# What is an endgame book?

Some games become simpler near the end of the game. Chess and checkers are like that. If the amount of material gets low, there are a relatively small amount of states left and you can lookup the result in a table.
This makes it so that you can lookup the end result without having to search further. There are several types of "end result". It can be a WLD data base (win/loss/draw). It can also be a DTM database (depth to mate), which tells you how long it takes to win (or lose) from this state. 
For oware we are using neither of those two. We are interested in how much net-score you can gain from a gamestate. Gamestates are "scoreless" in this sense. The score doesn't matter for the lookup itself.
For example, say the score is 21-20 and the lookup tells you -3, you know the second player has won. But if the score is 24-17, player 1 still wins. So it's a "score lookup".

# Turn limit

Oware has a turn limit (200 turns). This makes things a bit more complicated. When a gamestate has only 1 turn left, the net score will often be different than when it has 2 turns left.  
Depending on the number of seeds on the board, the turn limit may have effect even when 100+ turns are left to play. Oware has very long strategies and if the game ends halfway through one of them, then not all points will have been scored.

# Indexing the lookups

However you are going to generate the endgame book, you need to store it somehow. 



