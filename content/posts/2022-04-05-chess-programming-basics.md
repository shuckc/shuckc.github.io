---
title: Chess Programming Basics
date: 2022-04-04 17:00:00
tags:
    - chess
categories:
    - tech
showimageresources: true
showresizedimageresources: true
---

Curiosity
---
One fun aspect of parenting is revisiting things you learnt as a child, as the teacher. Since putting a chess board somewhat prominantly at home the eldest has expressed some interest and learnt a few moves. We bought a [chess book aimed for a young audiance](https://www.amazon.co.uk/Usborne-Chess-Book-Activity-Books/dp/1409598446/) to see the game taught gently, and I found myself learning more than expected.

![Chess Book](/images/usborne-chess-book.jpg)

Once down the rabbit hole of chess, I started to read about the early chess computers, and particularly those with bespoke hardware/software. I read with great interest of subsequent developments through to present with the rise of [stockfish](https://stockfishchess.org), the [introduction of NNUE](https://www.chessprogramming.org/Stockfish_NNUE) to the engine, and the subsequent training improvements using the schemes borrowed from alpha-go.

I wanted to type up some of my notes, and I hope to use them as the basis for this post and some subsequent deeper exploration of the concepts. In particular I am interesting in trying to get a basic chess engine working on an FPGA, ie. using digital logic. This seems to be a relatively unexplored, at least in the public domain.

History
---
The first recognisable chess programming paper is [Claude Shannon's](https://www.chessprogramming.org/Claude_Shannon) paper [Programming a Computer for Playing Chess](https://www.pi.infn.it/%7Ecarosi/chess/shannon.txt) which make some interesting contributions:
* precisely defines a "postion" `P`, ie. the board plus addional state to continue the game in accordance with the rules.
* states the intractability of either fully exploring a search tree (time), or pre-computing the best move for all positions (storage).
* gives an approximate position static evaluation function `f(P)`.
* states the shortcomings of static evaluation in the middle of piece exchanges, and the need to evaluate up to a quiescent (quiet) positions.
* introduces the switch between minimising/maximising the evaluation function during each ply to consider the best reply for each move during the search.
* proposes a binary coding (bit sizes) for position `P`, and a move `m` as tuple `(from, to, promotion)`.
* algorithm to apply a move to `P` to determine the next game state `'P`.
* that the moves for a queen are the sum of a rook and bishop, (what we now recognise as diagonal and manhatten).
* that some moves ('psuado legal') from a move generator may turn out to leave the player in check and therefore be illegal.
* definition of stability `g(P)` and a metric `h(P,m)` to prioritise the exploration of "forceful" moves (capturing, threatening, check).
* the use of opening books.
* changing evaluation weights at open/mid/end game.


Basic Representation
---

The canonical represetation of a chess board is a [Forsight-Edwards or 'fen' string](https://en.wikipedia.org/wiki/Forsyth%E2%80%93Edwards_Notation) that provides the necessary information to restart a game from a particular position. It's a 7-bit safe ASCII string such as `rnb1kb2/pppp4/6p1/4pp2/8/4PP2/PPPP2P1/RNB1KB2 w Qq - 0 9` shown below. There are 6 sections, separated by a space character.

1. Piece placement. Describes the 8 ranks each separated by a `/`, consisting of algebraic notation pieces for white `PNBRQK` and black `pnbrqk`, plus the numbers `12345678` to indicated repeated gaps. So the initial position is `rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR`.
2. Who to play, starting with `w` for white and `b` for black.
3. Castling rights, either a dash `-` for none, or one or more characters from `KQkq` to indicate for kings/queens side castling for white and black respectively.
4. En passant target square, a dash `-` or the algebraic notation `e3` of the square just passed over
5. Halfmove clock, as an ASCII decimal integer
6. Fullmove clock, as an ASCII decimal interger. Starts from 1 and increases after each black move.

{{< fen-diag fen="rnb1kb2/pppp4/6p1/4pp2/8/4PP2/PPPP2P1/RNB1KB2 w Qq - 0 9" >}}

Universal Chess Interface (UCI) Protocol
---
UCI is a [simple protocol](http://wbec-ridderkerk.nl/html/UCIProtocol.html) between chess engines and game UI's. It is again ASCII text only and puts the game UI in control of the chess engine. Either party can send a message to the other, messages as ASCII commands separated by a line break. A simple exchanage might be:

    > uci
    < id name MyEngine
    < uciok
    > isready
    < readyok
    > ucinewgame
    > position <fen string | startpos> <moves>
    > go infinite
    > go searchmoves <e2e4>
    > go movetime <msec>
    > go depth <2>
    > stop
    < bestmove <move>











