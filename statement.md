# Yavalath

Recently I've been fiddling with Yavalath, newly introduced to Coding Games. Yavalath is a computer-generated game. It is played on hexagonal board with edge size 5, with 61 cells. Players place stones and first one who gets 4-in-row wins. But there's catch. Having 3-in-row is a lose. The game creates many situations in which player is forced to prevent other player for reaching his goal and at some point, he won't be able to because he would create 3-in-row for himself. Due to advantage first player has, there is swap rule in which second player can either put a new stone or become the first player.

My algorithm of choice is MCTS, although I see people have success with minimax (at the moment of writing 2 top bots use MM).

# Opening books

In this article I will focus on opening books. Opening books are the first moves made in games. For some games they provide strategic advantage, for others not so much. Strong chess engines are equipped with opening books consisting of thousands and even more moves allowing them to start game with favourable position. For most of the history of computer chess, having strong opening moves provided huge advantage, to the point that any agent who left earlier from their opening book, lost. Nowadays chess programs are stronger than most humans and even without programmed openings they surpass them. Nevertheless, opening books are still valuable as they also allow to save time for other critical parts of the game. Not only chess benefits from good opening positions, other games can use them as well like checkers, shogi or to some degree, go. I will show that Yavalath bot (at least my rather weak MCTS one) can also benefit from opening books.

For starters, to see if trying to create good openings is feasible and will bring significant benefits, I ran my CPU vs CPU games. They were the same, except one CPU for the first 3 moves had 20x more time, so in a way I'm 'simulating' having opening book. After thousands of games it was clear - winrate for "openings" CPU was about 65% vs 35%, which is neat considering extra time was given to only 3 moves. In contrast, for Ultimate Tic Tac Toe having extra time for first 10 moves didn't benefit much. In conclusion, it is likely Yavalath bot will benefit from openings book.

## Book generation

Opening books can be hardcoded by humans or generated automatically. Obviously I will focus on the latter. But there is a nice excerpt from chess programming wiki on Opening Book:

> To solve the opening problems of his chess machine, Belle, Ken Thompson typed in opening lines from the Encyclopedia of Chess Openings (in five thick volumes). Religiously, he dedicated one hour a day for almost three years (!) to the tedious pursuit of entering lines of play from the books and having his Belle computer verify them. The result was an opening library of roughly three-hundred thousand moves. The results were immediate and obvious: Belle became a much stronger chess program, and Ken probably aged prematurely. Later Ken developed a program to automatically read the Encyclopedia, allowing him to do in a few days what had taken him three years to do manually.

For minimax, there is i.e. Drop-Out-Expansion. Since my bot utilizes MCTS, I will use methods provided in paper "Meta Monte-Carlo Tree Search for Automatic Opening Book Generation (2009)". Meta-MCTS is just fancy name, in essence it is doing CPU vs CPU games, with addition of saving game states and results and gathering statistics. And re-using moves that had >50% (or other threshold) winrate. But before doing that, let's take a closer look on Yavalath construction.

## Yavalath symmetries

Yavalath is player on hexagon board, and luckily, hexagon has more symmetries than square.

This Java template lets you get started quickly with a simple one-page playground.

```java runnable
// { autofold
public class Main {

public static void main(String[] args) {
// }

String message = "Hello World!";
System.out.println(message);

//{ autofold
}

}
//}
```

# Advanced usage

If you want a more complex example (external libraries, viewers...), use the [Advanced Java template](https://tech.io/select-repo/385)

### mmmmm
