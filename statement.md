# What is an endgame book?

Some games become simpler near the end of the game. Chess and checkers are like that. If the amount of material gets low, there are a relatively small amount of states left and you can lookup the result in a table.

Imagine using this inside a search, for example in a monte carlo tree search or minimax. This makes it so that you can lookup the end result without having to search further, saving calculation time and having the exact value of that gamestate.

# What is in the book?

There are several types of "end result". It can be a WLD data base (win/loss/draw). It can also be a DTM database (depth to mate), which tells you how long it takes to win (or lose) from this state. 
For oware we are using neither of those two. 

We are interested in how much net-score you can gain from a gamestate, assuming both sides play perfectly. Gamestates are "scoreless" in this sense. Whatever the current score is, has no bearing on which moves give the maximum net-score. So the current score doesn't matter for the lookup itself. For example, say the current score is 21-20 and the lookup tells you -3, you know the second player has won, because he is only 1 behind and will gain a net 3 points. But if the score is 24-17, player 1 still wins.

# Turn limit

Oware has a turn limit (200 turns). I use the word turn to mean "ply" here for those who know the lingo, just a single move by either player. This turn limit makes things a bit more complicated. When a gamestate has only 1 turn left, the net score will often be different than when it has 2 turns left. Depending on the number of seeds on the board, the turn limit may have effect even when 100+ turns are left to play. Oware has very long strategies and if the game ends halfway through one of them, then not all points will have been scored and the net-score will be different.

# Seed count

The more seeds are on the board, the more states are possible. With one seed on the board, there are only 12 possible states. Two seeds give you 78 possible states and so on. See the table below:

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


I've been able to generate books with a maximum of 9 seeds on the board in half a second. Having a maximum of 9 seeds will give you nearly 300k states. However, since we need to know their net-score for every amount of turns left, you have to multiply by another 200. In the end this comes down to a total of roughly 60 million gamestates. You can reduce this a little bit once you realize state values repeat after a certain number of turns left. For 9 seed gamestates this happens at 146 turns left or more.

Also this number is a slight overestimation of the number of states, because some states can't be reached. For example, the player that has just moved always has to have at least 1 empty hole. We ignore this to simplify things. 

# Indexing the lookups

When you generate the endgame book, you need to store it somehow. It is possible to hash the gamestate and use a hashtable like an unordered map, or your own implementation of a hashtable. However, this will be quite slow and use more memory. It's better to use a so-called index-function. That way you have the data in an array that is as compact as possible. An array lookup is much faster than a hashtable. During the generation of the book, you also keep using earlier calculated results, so lookups will be the main bottleneck.

Index functions are quite complicated however. It took a lot of thinking for me to figure one out for oware. The first order of business is to create a way to count the number of possible states. The result of this you can see in the table above. Run the code below to see how this works. The code uses math, but I would not know how to explain it quickly. It is basic math however and just calculates the amount of ways you can distribute x seeds over y pits. The reason I kept the number of pits variable will become clear later. Most of what you see in the function are factorials. This type of math easily overflows, which is why the function looks a little weird and has 64 bit integers. 

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

We will have to count states a lot, so it's better to cache the results. During the use of the index function we go through the board from pit 0 to 11. On each pit we have to ask the question, how many distributions of seeds are still possible with the remaining pits (those with index larger than the current pit). The function below fills an array that caches this result. 

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

Because there are 3 ways to do this, there are 3 states with 1 seed in pit 1. The sub-results of index 0,1,2 are added, leading to indices 4, 5 and 6. And so on. Hopefully this gives some idea of how it works. As I said, it's complicated. If you really want to understand the math, you will have to work it out for yourself on a piece of paper. 

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

My oware boardstate is always fully contained in a 64 bit integer. This makes things complicated when there are more than 31 seeds in a pit, as you only have 5 bits per pit. I use the 4 bits that are left (12*5 = 60), to handle the overflow. For our purposes, this does not matter. We are only looking at low seedcounts for the endgame book.  

# Generating the gamestates

The recursive function below is used to generate the states. It is quite simple to understand. We again walk through the pits, trying all possible amounts of seeds for the current pit and recursively finishing the state. You call the function by starting with an empty board and at house index 0. The "seedsLeft" variable has to be whatever amount of seeds you want there to be on the board. You can run the code below, but be careful not to call the function with too many seeds. You'll get an enormous amount of output and it will start skipping.

```C++ runnable
#include <iostream>
#include <string>

using namespace std;

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

# Retrograde analysis

The final code does a process called "retrograde analysis". The "retro" part means going backwards. You start at the very last turn of the game, which is turn 200 and calculate the net-scores for all the states, then you do the same at turn 199. The net-score of turn 199 is the capture score minus the netscore of the opponent at turn 200. For this we need to flip the board (switch player sides). We keep doing this all the way to turn 1. Actually... I stop at turn 200-146 = 54. This is because all the net-scores of turn 1 to 53 are identical to turn 54. Apparently there are no strategies longer than 146 turns at 9 seeds on the board. 

I decided to share my sim, so it's down below excepting 1 part. For pits with seeds > 31 you will have to come up with a creative solution. The endgame book generator doesn't need this.

```C++ runnable
#pragma GCC optimize("Ofast","unroll-loops", "omit-frame-pointer", "inline")
#pragma GCC option("arch=native", "tune=native", "no-zero-upper")
#pragma GCC target("rdrnd", "popcnt", "avx", "bmi2")

#include <iostream>
#include <sstream>
#include <random>
#include <string.h>
#include <algorithm>
#include <time.h>
#include <math.h>
#include <string>
#include <chrono>
#include <immintrin.h>
#include <unordered_map>
#include <unordered_set>
#include <unordered_map>

using namespace std;

#ifdef _MSC_VER
# include <intrin.h>
#  define __builtin_ctz ctz
#  define __builtin_ctzl ctzl
#  define __builtin_popcount __popcnt
#  define __builtin_popcountl __popcnt64

inline uint32_t ctz(uint32_t x)
{
	unsigned long r = 0;
	_BitScanForward(&r, x);
	return r;
}

inline uint64_t ctzl(uint64_t x)
{
	unsigned long r = 0;
	_BitScanForward64(&r, x);
	return r;
}

#endif

const uint64_t START_BOARD = 0x210842108421084;
const int END_GAME_SEEDS = 9;
const int SEED_9_STATECOUNT = 293929;
const int PATTERN_LIMIT_9 = 146;
auto start = std::chrono::high_resolution_clock::now();
uint64_t sowing[384] = { 0 };
uint32_t capturing[384] = { 0 };
uint8_t nextHouse[144] = { 0 };
uint64_t stateCounts[END_GAME_SEEDS+1][END_GAME_SEEDS+1][12] = { 0 };

uint64_t FlipBoard(uint64_t board)
{
	uint64_t p1 = board & 0x3FFFFFFF;
	uint64_t p2 = (board >> 30) & 0x3FFFFFFF;
	uint64_t extra = (board >> 60);
	return extra << 60 | p1 << 30 | p2;
}

int SeedsOnBoard(uint64_t board)
{
	int total = board >> 60;

	for (int i = 0; i < 60; i = i + 5)
		total += (board >> i) & 31;

	return total;
}

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

uint64_t IndexFunction(uint64_t state, int total) // assume state has pits with 31 seeds max, spaced as 5 bit
{
	uint64_t index = 0;
	int left = total;

	for (int house = 0; house < 11; house++)
	{
		int seeds = 31 & (state >> (house * 5));
		index += stateCounts[left][seeds][house];
		left -= seeds;
	}
	return index;
}

void FillStateCountLookups()
{
	for (int left = 0; left <= END_GAME_SEEDS; left++)
	{
		for (int seeds = 0; seeds <= min(left, END_GAME_SEEDS); seeds++)
		{
			for (int house = 0; house < 12; house++)
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

void SowingArray()
{
	for (int i = 0; i < 384; i++)
	{
		uint64_t changes[12] = { 0 };
		int houseOrigin = i % 12;
		int seeds = i / 12;
		int house = houseOrigin;
		while (seeds > 0)
		{
			house++;
			if (house == 12)
				house = 0;
			if (house == houseOrigin)
				continue;
			changes[house]++;
			seeds--;
		}
		for (int j = 0; j < 12; j++)
			sowing[i] += changes[j] << (j * 5);
		if (houseOrigin < 6 && house < 6 || houseOrigin > 5 && house > 5)
			sowing[i] |= 6ULL << 60;
		else
			sowing[i] |= (uint64_t)(house % 6) << 60;
	}
}

void CaptureArray()
{
	for (int i = 0; i < 384; i++)
	{
		int house = i >> 6;
		uint8_t code = 0;
		for (int j = house; j >= 0; j--)
		{
			if (i & (1 << j))
				code |= 1 << j;
			else
				break;
		}
		capturing[i] = code;
	}
}

void NextHouseArray()
{
	for (int i = 0; i < 144; i++)
	{
		int origin = i / 12;
		int current = i % 12;

		current++;
		if (current > 11)
			current = 0;

		if (origin == current)
		{
			current++;
			if (current > 11)
				current = 0;
		}
		nextHouse[i] = current;
	}
}

inline uint64_t BoardFromHouseArray(int houses[])
{
	uint64_t board = 0;
	for (int i = 0; i < 12; i++)
	{
		if (houses[i] > 31)
		{
			board |= 31ULL << (5 * i);
			board |= ((uint64_t)(houses[i] - 31)) << 60;
		}
		else
			board |= ((uint64_t)houses[i]) << (5 * i);
	}

	return board;
}

inline int ApplyNoCheck(int move, uint64_t& board, int player)
{
	// careful this function does not work if a pit has > 31 seeds. That's never the case for endgame books
	int opponent = player ^ 1;
	int shift = 5 * move + 30 * player;
	int originSeeds = (board >> shift) & 31;
	uint64_t sowingLookup = sowing[12 * originSeeds + move + 6 * player];
	board += sowingLookup & 0xFFFFFFFFFFFFFFF;
	board &= ~(31ULL << shift);
	int lastHouse = sowingLookup >> 60;
	if (lastHouse == 6)
		return 0;

	int opponentShift = 30 * opponent;

	uint32_t oppBoard = board >> opponentShift;
	uint32_t bit2 = oppBoard & 0x4210842;
	uint32_t bit3 = (oppBoard & 0x8421084) >> 1;
	uint32_t bit4 = (oppBoard & 0x10842108) >> 2;
	uint32_t bit5 = (oppBoard & 0x21084210) >> 3;
	uint32_t combined = bit2 & ~bit3 & ~bit4 & ~bit5;

	if (combined == 0)
		return 0;

	uint32_t captureKey = lastHouse << 6 | _pext_u32(combined, 0x4210842);
	uint8_t captureLookup = capturing[captureKey];
	uint32_t selectedBits = _pdep_u32(captureLookup, 0x4210842);
	uint32_t selectedExtended = selectedBits | selectedBits >> 1;
	uint32_t allCapturedBits = selectedExtended & oppBoard;

	uint32_t tempBoard = oppBoard & ~allCapturedBits;   // check if the board doesn't become empty

	if (tempBoard == 0)
		return 0;
	else
	{
		board &= ~(0x3FFFFFFFULL << opponentShift);
		board |= (uint64_t)tempBoard << opponentShift;
		int bit2Count = __builtin_popcount(selectedBits);
		int bit12Count = __builtin_popcount(allCapturedBits);
		return bit2Count + bit12Count;
	}
}

inline bool PlayerHasSeeds(int player, uint64_t board)
{
	if (((board >> (30 * player)) & 0x3FFFFFFF) == 0)
		return false;
	else
		return true;
}

inline bool HouseHasSeeds(int houseIndex, int playerIndex, uint64_t board)
{
	int houseShift = 5 * (6 * playerIndex + houseIndex);
	return (board & (31ULL << houseShift)) != 0;
}

inline int GetPlayerSeeds(int player, uint64_t board)
{
	uint32_t playerBoard = board >> (30 * player);
	int playerSeeds = 0;
	for (int i = 0; i < 6; i++)
		playerSeeds += (playerBoard >> (5 * i)) & 31;

	return playerSeeds;
}

struct SimCache
{
	uint64_t board;
	int childIndexBuffer[6];
	int8_t capturedBuffer[6];
	int8_t moveCount;
	int8_t currentSeeds;

	SimCache(){};
};

SimCache buffers[SEED_9_STATECOUNT];
int stateCount = 0;
int8_t endSeeds[SEED_9_STATECOUNT * PATTERN_LIMIT_9];
int arrayStarts[END_GAME_SEEDS + 1];

inline int GetSeedScore(uint64_t childBoard, int turnsLeft, int seedsOnBoard)
{
	if (turnsLeft > 146)
		turnsLeft = 146;
	int stateIndex = arrayStarts[seedsOnBoard] + IndexFunction(childBoard, seedsOnBoard);
	int index = (turnsLeft - 1) * stateCount + stateIndex;

	return endSeeds[index];
}

inline void SetSeedScore(int stateIndex, int turnsLeft, int seeds)
{
	endSeeds[(turnsLeft - 1) * stateCount + stateIndex] = seeds;
}

void GenerateStates(int seedsLeft, uint64_t board, int house, int current)
{
	if (house < 11)
	{
		for (int i = 0; i <= seedsLeft; i++)
			GenerateStates(seedsLeft - i, board | (uint64_t)i << (house * 5), house + 1, current);
	}
	else
	{
		board |= (uint64_t)seedsLeft << 55;
		buffers[stateCount].currentSeeds = current;
		buffers[stateCount++].board = board;
	}
}

void GenerateBook()
{
	stateCount = 0;

	for (int currentSeeds = 1; currentSeeds <= END_GAME_SEEDS; currentSeeds++)
	{
		arrayStarts[currentSeeds] = stateCount;
		GenerateStates(currentSeeds, 0, 0, currentSeeds);
	}

	for (int s = 0; s < stateCount; s++)
	{
		uint64_t board = buffers[s].board;
		int moveCount = 0;
		int bestScore = -100;

		for (int house = 0; house < 6; house++)
		{
			if (!HouseHasSeeds(house, 0, board))
				continue;
			
			uint64_t childBoard = board;
			int captured = ApplyNoCheck(house, childBoard, 0);
			bestScore = max(captured, bestScore);

			if (!PlayerHasSeeds(1, childBoard))
				continue;
			else
			{
				int seedsLeft = buffers[s].currentSeeds - captured;
				buffers[s].capturedBuffer[moveCount] = captured;
				buffers[s].childIndexBuffer[moveCount] = arrayStarts[seedsLeft] + IndexFunction(FlipBoard(childBoard), seedsLeft);
				moveCount++;
			}
		}
		buffers[s].moveCount = moveCount;
		if (moveCount == 0)
			bestScore = GetPlayerSeeds(0, board);

		SetSeedScore(s, 1, bestScore);
	}

	for (int turn = 2; turn <= PATTERN_LIMIT_9; turn++)
	{
		for (int s = 0; s < stateCount; s++)
		{
			int bestScore = -100;

			if (buffers[s].moveCount == 0)
				bestScore = GetPlayerSeeds(0, buffers[s].board);
			else
			{
				for (int m = 0; m < buffers[s].moveCount; m++)
				{			
					int captured = buffers[s].capturedBuffer[m];
					int childIndex = buffers[s].childIndexBuffer[m];
					int score = captured - endSeeds[(turn - 2) * stateCount + childIndex];
					bestScore = max(score, bestScore);
				}
			}

			SetSeedScore(s, turn, bestScore);
		}
	}

	auto now = std::chrono::high_resolution_clock::now();
	auto calcTime = std::chrono::duration_cast<std::chrono::milliseconds>(now - start).count();

	std::cerr << "End games done. Time: " << calcTime << " ms. States: " << stateCount << endl;
}

int main()
{
	SowingArray();
	CaptureArray();
	NextHouseArray();
	FillStateCountLookups();
	GenerateBook();
}
```