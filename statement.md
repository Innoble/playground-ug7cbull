# What is an endgame book?

Some games become simpler near the end of the game. Chess and checkers are like that. If the amount of material gets low, there are a relatively small amount of states left and you can lookup the result in a table.

Imagine using this inside a search, for example in a monte carlo tree search or minimax. This makes it so that you can lookup the end result without having to search further, saving calculation time and having the exact value of that gamestate.

# What is in the book?

There are several types of "end result". It can be a WLD data base (win/loss/draw). It can also be a DTM database (depth to mate), which tells you how long it takes to win (or lose) from this state. 
For oware we are using neither of those two. 

We are interested in how much net-score you can gain from a gamestate. Gamestates are "scoreless" in this sense. The current score doesn't matter for the lookup itself. For example, say the current score is 21-20 and the lookup tells you -3, you know the second player has won, because he is only 1 behind and will gain a net 3 points. But if the score is 24-17, player 1 still wins.

# Turn limit

Oware has a turn limit (200 turns). I use the word turn to mean "ply" here for those who know the lingo, just a single move by either player. This turn limit makes things a bit more complicated. When a gamestate has only 1 turn left, the net score will often be different than when it has 2 turns left.  
Depending on the number of seeds on the board, the turn limit may have effect even when 100+ turns are left to play. Oware has very long strategies and if the game ends halfway through one of them, then not all points will have been scored.

# Seed count

The more seeds are on the board, the more states are possible. With 1 seed on the board, there are only 12 states, 2 seeds give you 78 states and so on. See the table below:

| Number of seeds on the board | Number of possible states | Cumulative |
|------------------------------|---------------------------|------------|
| 1                            | 12                        | 12         |
| 2                            | 78                        | 90         |
| 3                            | 364                       | 454        |
| 4                            | 1365                      | 1819       |
| 5                            | 4368                      | 6187       |
| 6                            | 12376                     | 18563      |
| 7                            | 31824                     | 50387      |
| 8                            | 75582                     | 125969     |
| 9                            | 167960                    | 293929     |
| 10                           | 352716                    | 646645     |


I've been able to generate books with a maximum of 9 seeds on the board in half a second. Having a maximum of 9 seeds will give you nearly 300k states. However, since we need to know their net score for every amount of turns left, you have to multiply by another 200. In the end this comes down to a total of roughly 60 million gamestates. You can reduce this a little bit once you realize state values repeat after a certain number of turns left. For 9 seeds this happens at 146 tur

Also this is a slight overestimation of the number of states, because some states can't be reached. For example, the player that has just moved always has to have at least 1 empty hole. We ignore this to simplify things. 

# Indexing the lookups

When you generate the endgame book, you need to store it somehow. It is possible to hash the gamestate and use a hashtable like c++'s unordered map, or your own implementation of a hashtable. However, this will be extremely slow. It's better to use a so-called index-function. That way you have the data in an array that is as compact as possible. An array lookup is much faster than a hashtable. During the generation of the book, you also keep using earlier calculated results, so lookups will be the main bottleneck.

Index functions are quite complicated however. It took a lot of thinking for me to figure out a good one for oware. The first order of business is to create a way to count the number of possible states. The result of this you can see in the table above. Run the code below to see how this works.

```C++ runnable
#include <iostream>

using namespace std;

uint64_t StateCounter(int64_t pits, int64_t seeds)
{
	int64_t top = 1;
	int64_t bottom = 1;
	if (pits > seeds)
	{
		for (int64_t i = seeds + 1; i <= seeds + pits - 1; i++)
			top *= i;

		for (int64_t i = 2; i <= (pits - 1); i++)
			bottom *= i;
	}
	else
	{
		for (int64_t i = pits; i <= seeds + pits - 1; i++)
			top *= i;

		for (int64_t i = 2; i <= seeds; i++)
			bottom *= i;
	}

	return top /= bottom;
}

int main()
{
	for (int seeds = 1; seeds <= 10; seeds++)
	{
		int stateCount = StateCounter(12, seeds);
		std::cout << "Seeds: " << seeds << " Possible states: " << stateCount << endl;
	} 
}

```