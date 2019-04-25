# Yavalath

Recently I've been fiddling with Yavalath, newly introduced to Coding Games. Yavalath is a computer-generated game. It is played on hexagonal board with edge size 5, with 61 cells. Players place stones and first one who gets 4-in-row wins. But there's catch. Having 3-in-row is a lose. The game creates many situations in which player is forced to prevent other player for reaching his goal and at some point, he won't be able to because he would create 3-in-row for himself. Due to advantage first player has, there is swap rule in which second player can either put a new stone or become the first player.

My algorithm of choice is MCTS, although I see people have success with minimax (at the moment of writing 2 top bots use MM).


# Opening books

In this article I will focus on opening books. Opening books are the first moves made in games. For some games they provide strategic advantage, for others not so much. Strong chess engines are equipped with opening books consisting of thousands and even more moves allowing them to start game with favourable position. For most of the history of computer chess, having strong opening moves provided huge advantage, to the point that any agent who left earlier from their opening book, lost. Nowadays chess programs are stronger than most humans and even without programmed openings they surpass them. Nevertheless, opening books are still valuable as they also allow to save time for other critical parts of the game. Not only chess benefits from good opening positions, other games can use them as well like checkers, shogi or to some degree, go. I will show that Yavalath bot (at least my rather weak MCTS one) can also benefit from opening books.

For starters, to see if trying to create good openings is feasible and will bring significant benefits, I ran my CPU vs CPU games. They were the same, except one CPU for the first 3 moves had 20x more time, so in a way I'm 'simulating' having opening book. After thousands of games it was clear - winrate for "openings" CPU was about 65% vs 35%, which is neat considering extra time was given to only 3 moves. In contrast, for Ultimate Tic Tac Toe having extra time for first 10 moves didn't benefit much. In conclusion, it is likely Yavalath bot will benefit from openings book.


## Book generation

Opening books can be hardcoded by humans or generated automatically. Obviously I will focus on the latter. But there is a nice excerpt from chess programming wiki on Opening Book:

> To solve the opening problems of his chess machine, Belle, Ken Thompson typed in opening lines from the Encyclopedia of Chess Openings (in five thick volumes). Religiously, he dedicated one hour a day for almost three years (!) to the tedious pursuit of entering lines of play from the books and having his Belle computer verify them. The result was an opening library of roughly three-hundred thousand moves. The results were immediate and obvious: Belle became a much stronger chess program, and Ken probably aged prematurely. Later Ken developed a program to automatically read the Encyclopedia, allowing him to do in a few days what had taken him three years to do manually.

For minimax, there is i.e. Drop-Out-Expansion algorithm. Since my bot utilizes MCTS, I will use methods provided in paper "Meta Monte-Carlo Tree Search for Automatic Opening Book Generation (2009)". Meta-MCTS is just like MCTS, but simulation/rollout step is replaced by entire MCTS bot. Meta-MCTS in case of the paper is just fancy name, in essence it is doing CPU vs CPU games, with addition of saving game states and results and gathering statistics. And re-using moves that had >50% (or other threshold) winrate. In short, we have State S and respective reply Action A to it. Having a state S i.e. in a form of "First Player's stones|Second Player's stones" automatically deals with transpositions (the same situation in game reachable from different order of moves), because we don't care about the order of moves.

```
class Action {
    Move move;
    int wins;
    int games;
    
    float winrate() {
        return (float)wins / games;
    }
}

Map<State,List<Action>> storedStates;
// ...

while (true) {
    List<State> gameStates;
    List<Action> replies;
    Game game;
    // ...
    
    while(!game.isOver()) {
        State state = game.getState();
        Action bestAction = null;
        if (storedStates.containsKey(state)) {
            for (Action action : storedStates.get(state)) {
                if (bestAction == null || action.winrate() > bestAction.winrate()) {
                    bestAction = action;
                }
            }
        }
        gameStates.add(state);
        if (bestAction != null && bestAction.winrate() > 0.5) {
            game.makeMove(action.move);
            replies.add(action);
        } else {
            Action action = cpu.getBestMove();
            game.makeMove(action.move);
            replies.add(action);
        }
    }
    
    for (int i=0;i<gameStates.size();i++) {
        State state = gameStates.get(i);
        Action action = replies.get(i);
        // add game state and add or update its action to the storedStates
        // and update action's wins and games accordingly
    }
}
```

I am using MCTS solver, so I will use meta MCTS solver as well. To the Action I added boolean terminal and whenever I know the Action has been solved, I set the terminal to true and wins to INFINITY or -INFINITY. Then next time a state is encountered, if winning Action has been found I use it. For losing Actions, I provide them for the cpu to avoid.


## Yavalath symmetries

Yavalath is played on hexagonal board. Luckily, hexagon has more symmetries than square. Hexagon has 6 symmetries and 6 rotations, so in average a state has 12 equivalent variations. We can exploit this fact in few ways. Firstly, naively speaking Yavalath has 61 starting moves, but thanks to symmetry deduplication, there are only 9 distinct moves. And for the few first moves, much deduplication will occur so we're gonna need to explore much less distinct moves for our opening book. For example, there are 328 unique states for 2 stones, instead of 9 x 60 = 540. Secondly, we can support averagily 12 variations if we save just 1 state in the book. This way, we can "compress" our moves by factor ~12, which will be invaluable due to 100k characters limit for code.

The updated pseudocode. It'll take more space but I'm generating book offline so I don't care much. Plus, it's easier to maintain book this way for me.
```
// ...
    for (int i=0;i<gameStates.size();i++) {
        State state = gameStates.get(i);
        Action action = replies.get(i);
        // add game state and add or update its action to the storedStates
        // and update action's wins and games accordingly
        for (each non-repeating variation of state and action) {
            State statePrim = rotateOrSymmetry(state);
            Action actionPrim = rotateOrSymmetry(action);
            // add game state prim and add or update its action prim to the storedStates
            // and update action prim's wins and games accordingly
        }
    }
```


## Final result

We run above algorithm for some time, i.e. I ran overnight with 1s per move and played several thousand games. Now I have statistics of the positions and corresponding replies, what do to with them? Firstly, there is character limit on CG to 100k characters, so I can't use stored all the moves. Secondly, I want to avoid very bad moves. Probably there are several methods to distill what I have so far into relevant openings book. Some ideas I took from paper "A Principled Method for Exploiting Opening Books (2010)", namely the lower confidence bound. With trials and errors, this is roughly what I use now:

```
Map<String, Move> openings;
// ...

Set<State> states = storedActions.keySet();
for (State state : states) {
    if (openings.containsKey(state.toString()) || /* openings contains any variation of the state, so deduplicate symmetries here */) {
        continue;
    }
    Action bestAction = null;
    float bestWinrateLCB = -1;
    for (Action action : storedActions.get(state)) {
        if (action.games < 10 && action.wins != INFINITY) { // 10 adjustable
            continue;
        }
        float winrateLCB = action.winrate() - 2.0f / Math.sqrt(action.games); // 2.0 adjustable
        if (bestAction == null || winrateLCB > bestWinrateLCB) {
            bestAction = action;
            bestWinrateLCB = winrateLCB
        }
    }
    if (bestWinrateLCB > 0.2) { // 0.2 adjustable
        openings.put(state.toString(),bestAction.move);
    }
}

// now the openings contains opening book to use
```

Probably it's gonna change as I improve myself and find better ways to do it.

As for the actual bot, at the beginning of its turn, it first check the game's state and all of its variations (symmetries and rotations) if they have been stored in the map. If yes, use the reply, otherwise use the usual cpu.getBestMove(). I.e. there is a State S in the actual game, and we have it stored as double rotated by 60°, S''. So we have stored Action A'', thus we need to rotate it back twice into A. Now we play the Action A. Voila!


# Experiments

Here are some experiment results. I run my cpu vs cpu. I don't assume swap rule, the first ply (9 distinct moves) is played randomly, so I can get the idea which move to swap or which move to play so it won't be swapped. Also I can exploit my opponent if he decides to start with inferior moves.


## Early attempts

During my early attempts, I ran my algorithm for ~20000 games and gathered statistics. I limited my openings to ~1500 nodes due to poor way of storing them in the code (I had 1500 of map["state"]=move;map["state2"]=move2;...). My cpu with openings vs cpu w/o openings yielded 61:39 winrate. When I submitted new code, my bot went higher in rank, it was significantly stronger. That was encouranging. Against minimax bots the beginning was quite deterministic, because the bots were making the same moves. I could see for many occassions my bot "learned" triangle patterns which were strong. Unfortunately, some moves stored in the book were very bad. For example, after 2 moves from the book, it turned out my bot was in situation of sure loss, which was exploited by the minimax bot. I had to manually remove the move. The move was picked up in the first place, because it had good statistics of winning. So while meta-mcts made my bot stronger, it still contained very bad moves, probably due to its greedy nature, which was addressed in the meta-mcts paper.


## Later attempts

With time, my bot improved due to enhancements I provided into its code. I decided to generate new openings book from scratch. I added some random factor in choosing move, so even if move had 90% winrate, there was chance it wouldn't be chosen and bot would play by itself instead. This way it should be more exploratory. Also, after several thousand games, I decided to make 2 first plies move random, so it'd make book more shallow, but more exploratory. I ran this for longer time, for ~75k games. I also optimized the code, now I hardcode string into code and decode it into map in 1st turn. Now I have ~5500 nodes in the book. Cpu with openings vs cpu w/o openings gave 68:32 winrate, so better than last time. It also stayed longer in the book. When I submitted the code, my bot slightly improved. Interestingly, it stayed longer in the book against minimax bots than (as far as I presume) mcts bots. Still there was one move (as far as I know), which after leaving the book led to loss.


## Future

It is tempting to see if using much more games would give yet better results, or maybe solve some moves (AFAIK move in center is proven win). But for now I'll spend more time to improve the simple MCTS. Perhaps in the meantime I'll find better algorithm for opening moves generation or find a better way to verify the moves.


# Conclusion

Yavalath benefits from having stronger openings. I think this is due to many shallow but out-of-horizon trap states, which should be avoided. Thanks to hexagonal shape of the board, there are many symmetries allowing to save space and reducing the time needed to explore the beginning of game. Thanks to atomatic generation, I can run my program overnight and wake up to stronger bot. It is some form of learning, but in the age of reinforcement learning and neural net, rather primitive one.

