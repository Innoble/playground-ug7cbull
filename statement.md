# What is an endgame book?

Some games become simpler near the end of the game. Chess and checkers are like that. If the amount of material gets low, there are a relatively small amount of states left and you can lookup the result in a table.

Imagine using this inside a search, for example in a monte carlo tree search or minimax. This makes it so that you can lookup the end result without having to search further, saving calculation time and having the exact value of that gamestate.

# What is in the book?

There are several types of "end result". It can be a WLD data base (win/loss/draw). It can also be a DTM database (depth to mate), which tells you how long it takes to win (or lose) from this state. 
For oware we are using neither of those two. 

We are interested in how much net-score you can gain from a gamestate. Gamestates are "scoreless" in this sense. The current score doesn't matter for the lookup itself. For example, say the current score is 21-20 and the lookup tells you -3, you know the second player has won, because he is only 1 behind and will gain a net 3 points. But if the score is 24-17, player 1 still wins.

# Turn limit

Oware has a turn limit (200 turns). I use the word turn to mean "ply" here for those who know the lingo, just a single move by either player. This turn limit makes things a bit more complicated. When a gamestate has only 1 turn left, the net score will often be different than when it has 2 turns left. Depending on the number of seeds on the board, the turn limit may have effect even when 100+ turns are left to play. Oware has very long strategies and if the game ends halfway through one of them, then not all points will have been scored.

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


I've been able to generate books with a maximum of 9 seeds on the board in half a second. Having a maximum of 9 seeds will give you nearly 300k states. However, since we need to know their net score for every amount of turns left, you have to multiply by another 200. In the end this comes down to a total of roughly 60 million gamestates. You can reduce this a little bit once you realize state values repeat after a certain number of turns left. For 9 seeds this happens at 146 turns left.

Also this number is a slight overestimation of the number of states, because some states can't be reached. For example, the player that has just moved always has to have at least 1 empty hole. We ignore this to simplify things. 

# Indexing the lookups

When you generate the endgame book, you need to store it somehow. It is possible to hash the gamestate and use a hashtable like c++'s unordered map, or your own implementation of a hashtable. However, this will be extremely slow. It's better to use a so-called index-function. That way you have the data in an array that is as compact as possible. An array lookup is much faster than a hashtable. During the generation of the book, you also keep using earlier calculated results, so lookups will be the main bottleneck.

Index functions are quite complicated however. It took a lot of thinking for me to figure out a good one for oware. The first order of business is to create a way to count the number of possible states. The result of this you can see in the table above. Run the code below to see how this works. The code uses math, but I would not know how to explain it quickly. It is basic math however and just calculates the amount of ways you can distribute x seeds over y pits. The reason I kept the number of pits variable will become clear later. Most of what you see in the function are factorials. This type of math easily overflows, which is why the function looks a little weird and has 64 bit integers. 

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

# Caching state counts

We will have to count states a lot, so it's better to cache the result. During the use of the index function we go through the board from pit 0 to 11. On each pit we have to ask the question, how many distributions of seeds are still possible with the remaining pits (index larger than the current pit). The function below fills an array that caches this result. 

```C++
void FillStateCountLookups()
{
	for (int left = 0; left <= END_GAME_SEEDS; left++) // how many seeds are left to distribute
	{
		for (int house = 0; house < 11; house++)
		{
			for (int seeds = 0; seeds <= left; seeds++) // how many seeds are in the current pit
			{
				uint64_t index = 0;
				for (int j = 0; j < seeds; j++)
				{
					uint64_t stateCount = StateCounter(11 - house, left - j);
					index += stateCount;
				}

				stateCounts[left][seeds][house] = index;
			}
		}
	}
}

```

It is quite difficult to understand how this indexing works, but let's try to use a 3 pit, 3 seed example.

say we're walking through the pits 0 to 11 and we're at pit 9. We still have 3 seeds to distribute over pits 9 to 11. Then we might as well say there are only 3 pits and they are numbered 1 to 3 The indexing is done as follows:

| pit 1 | pit 2  | pit 3  | index |
|-------|--------|--------|-------|
| 0     | 0      | 3      | 0     |
| 0     | 1      | 2      | 1     |
| 0     | 2      | 1      | 2     |
| 0     | 3      | 0      | 3     |
| 1     | 0      | 2      | 4     |
| 1     | 1      | 1      | 5     |
| 1     | 2      | 0      | 6     |
| 2     | 0      | 1      | 7     |
| 2     | 1      | 0      | 8     |
| 3     | 0      | 0      | 9     |

We walk through the pits, starting at 1. If we place 0 seeds in pit 1 and 2, we *must* place 3 seeds in pit 3. This is the lowest state, with index 0. We store this state. 

Next we make a state with 1 seed in pit 2. That means the remaining seeds go in pit 3. The total number of states stored so far is 1, so the index of this is 1. We keep doing this until we have to start changing pit 1. When we pick 1 seed for pit 1, pit 2 and 3 will have fewer possible configurations. In effect you get this table for the remaining two pits:

| pit 2 | pit 3  | index |
|-------|--------|-------|
| 0     | 2      | 0     |
| 1     | 1      | 1     |
| 2     | 0      | 2     |

Because there are 3 ways to do this, there are 3 states with 1 seed in pit 1. The sub-results of index 0,1,2 are added, leading to indices 4, 5 and 6. And so on. Hopefully this gives some idea of how it works. As I said, it's complicated.

The final code for the index function is as follows: 

```c++
uint64_t IndexFunction(uint64_t state, int total) // assume state has pits with 31 seeds max, spaced as 5 bit
{
	uint64_t index = 0;
	int left = total;

	for (int house = 0; house < 11; house++)
	{
		int seeds = 31 & (state >> (house * 5)); // the number of seeds on the house.
		index += stateCounts[left][seeds][house]; // increasing the total index.
		left -= seeds; // less seeds left to distribute.
	}
	return index;
}
```

My oware boardstate is always fully contained in a 64 bit integer. This makes things complicated when there are more than 31 seeds in a pit, as you only have 5 bits per pit. I use the 4 bits that are left (12*5 = 60), to handle the overflow. For our purposes, this does not matter. We are only looking at low seedcounts.  

# Generating the gamestates

The recursive function below is used to generate the states. It is quite simple to understand. We again walk through the pits, trying all possible amounts of seeds for the current pit and recursively finishing the state.
```c++
#include <iostream>
#include <string>

using namespace std;

void PrintBoard(uint64_t board)
{
	string myBoard = "";
	string oppBoard = "";
	for (int i = 0; i < 6; i++)
	{
		int seeds = (board >> (i * 5)) & 31;
		if (seeds == 31)
			seeds += board >> 60;

		myBoard += to_string(seeds) + " ";
	}

	for (int i = 11; i >= 6; i--)
	{
		int seeds = (board >> (i * 5)) & 31;
		if (seeds == 31)
			seeds += board >> 60;
		oppBoard += to_string(seeds) + " ";
	}

	std::cerr << oppBoard << endl;
	std::cerr << myBoard << endl;
}

uint64_t allBoards[10000000] = { 0 };
int stateCount = 0;

void GenerateStates(int seedsLeft, uint64_t board, int house)
{
	if (house < 11)
	{
		for (int i = 0; i <= seedsLeft; i++)
			GenerateStates(seedsLeft - i, board | (uint64_t)i << (house * 5), house + 1);
	}
	else
	{
		board |= (uint64_t)seedsLeft << 55;
		allBoards[stateCount++] = board;
	}
}

int main()
{
	GenerateStates(1, 0, 0);

	for (int i = 0; i < stateCount; i++)
	{
		cerr << "index: " << i << endl;
		PrintBoard(allBoards[i]);		
	}
}
```