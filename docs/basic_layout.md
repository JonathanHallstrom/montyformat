
## High Level Layout

Each file is a series of games, and each game is represented as a compressed board, followed by castling files, the games outcome, then a series of moves along with some extra information, and terminated by a null move (two zero bytes).
For each move of the game we write which move was played, the score given to the position by the search, the number of legal moves, and the visit distribution.

So a full game looks as follows:

\[Board]\[CastlingFiles]\[WDL\](\[Move]\[Score]\[MoveCount]\[VisitDistribution])*[NullTerminator]

All values are written in little endian.

## Compressed Board

The basic layout of the board is as follows, using C-like syntax.

```c
struct CompressedBoard {
    uint64_t bitboards[4];
    uint8_t side_to_move;
    uint8_t ep_square;
    uint8_t castling_rights;
    uint8_t halfmove_clock;
    uint16_t fullmove_clock;
};
```

### Bitboards

This part is a little complicated, because to save space the 8 bitboards (2 for each side, 6 for each piece) are stored in an overlapping way utilising the fact that no two pieces can be on the same square.

The compression is as follows:

```c
bitboards[0] = black;
bitboards[1] = rooks   | queens  | kings;
bitboards[2] = knights | bishops | kings;
bitboards[3] = pawns   | bishops | queens;
```

To decompress we do the following (justification for why this works is pretty straightforward, left as an exercise to the reader):

```c
uint64_t occupancy = bitboards[1] | bitboards[2] | bitboards[3];
uint64_t black = bitboards[0];
uint64_t white = occupancy ^ black;
uint64_t kings = bitboards[1] & bitboards[2];
uint64_t queens = bitboards[1] & bitboards[3];
uint64_t rooks = bitboards[1] ^ kings ^ queens;
uint64_t bishops = bitboards[2] & bitboards[3];
uint64_t knights = bitboards[2] ^ bishops ^ kings;
uint64_t pawns = bitboards[3] ^ bishops ^ queens;
```
### Side To Move
A single `uint8_t`, with white=`0` and black=`1`.

### Side To Move

A single `uint8_t`, representing the target for en passant with `0`=A1 `1`=A2 `3`=A3, ..., `8`=B1, ... `63`=H8.
If there is no valid ep target this value will be zero.

### Castling Rights

Here we only need the lower 4 bits, and the layout is `0b0000'QKqk` with each of the used bits representing whether that castling right is available.

### Halfmove Clock and Fullmove Clock

Halfmove clock is the number of plies since the last capture or pawn move.
Fullmove clock is the number of moves since the game started, starting from 1, standard stuff.

## Castling Files

In QKqk order we write the files on which the rooks are for castling.

If a castling right isn't available you should write the file farthest on that side of the board, for queenside castling this means the A file and for kingside this means the H file.

If you're only supporting standard chess you can just write the standard `HAha` rook files.

Each file is written as a `uint8_t` with `0` meaning the A file, `1` meaning the B file and so on.


## Game Outcome

A single `uint8_t`, 0 for loss, 1 for draw, 2 for win (from the perspective of white). So if white won the game you would write `2`, and if black won the game you would write `0`. If the game is a draw you write `1`.

## Moves and their associated information

### Moves

Moves are represented by `uint16_t`s with this layout: 4 bit flag, 6 bit destination [0-64), 6 bit source [0-64). 

To compute the representation for a move we do the following:
```c
uint64_t move = flag | (destination << 4) | (source << 10);
```

The source and destination squares are represented as follows: `0`=A1 `1`=A2 `3`=A3, ..., `8`=B1, ... `63`=H8

Note that castling is represented by the king moving from its starting position to either the G or C files, not as king takes rook.

The flags are as follows:

0. Quiet move
1. Pawn double push
2. Kingside Castling
3. Queenside Castling
4. Capture
5. En Passant
6. (unused)
7. (unused)
8. Knight Promotion
9. Bishop Promotion
10. Rook Promotion
11. Queen Promotion
12. Knight Promotion Capture
13. Bishop Promotion Capture
14. Rook Promotion Capture
15. Queen Promotion Capture

A few examples:

Knight from G1 to F3: \
Source: 6 \
Destination: 21 \
Flag: 0 \
Final result: 0 | (21 << 4) | (6 << 10) = 6480


Pawn from E2 to E4: \
Source: 12 \
Destination: 28 \
Flag: 1 \
Final result: 1 | (28 << 4) | (12 << 10) = 12737


Pawn from E2 to E1 capturing piece and promoting to a rook: \
Source: 12 \
Destination: 4 \
Flag: 14 \
Final result: 14 | (4 << 4) | (12 << 10) = 12366

### Score

The score of the position is represented by one `uint16_t`.
Typically the search will have given the position a score from `0` to `1`, in which case we write `65535` times that score.

If your search instead returns something else you should convert it to a score from `0` to `1` first, then multiply as above.

### Visit Distribution

For each of the legal moves we write what portion of the visits they received during the search, compared to the move that got the most visits.

Each move gets one `uint8_t` to represent how many visits it got, and the value is computed as the number of visits times 255 divided by the number of visits received by the move with the most visits.

To write this distribution we first write the number of moves.
Then we write the `uint8_t`s we computed earlier, ordering by the `uint16_t` representing the move they are for (in the format described above).

## Null Terminator

This is very simple, just write two null bytes for a null move.
