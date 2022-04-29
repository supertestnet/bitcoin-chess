# Bitcoin Chess
Play a game of chess and store every move on bitcoin's blockchain. An educational experiment with key tweaking.

# Inspiration

In [this blog post](https://rubin.io/bitcoin/2021/12/14/advent-17/) Jeremy Rubin featured an ethereum-based chess game and raised the question, "Why isn’t [its inventor] working on Bitcoin?" It made me wonder if chess logic can be implemented in bitcoin like it can in eth. Later, Lightning Labs announced [taro](https://lightning.engineering/posts/2022-4-5-taro-launch/) which uses key tweaking to hide non-bitcoin tokens inside bitcoin pubkeys. It made me want to illustrate how key tweaking works so that other developers can make other cool things. These two inspirations (chess on eth & key tweaking in taro) were the seeds of Bitcoin Chess.

# How to play

Go here and follow the instructions: https://supertestnet.github.io/bitcoin-chess/bitcoin-chess.html

You'll need to give your pubkey to the other player and enter their pubkey when they give it to you. You'll also need some testnet coins. Get some here: [https://bitcoinfaucet.uo1.net/](https://bitcoinfaucet.uo1.net/) About 10,000 sats should be enough for a full game of chess.

# How it works

# First ingredient: key tweaking

Bitcoin addresses have two main components: a public key and a private key. The private key is something you're supposed to keep secret because it lets you move the money in the bitcoin address. The public key is fine to share with people, it's practically the same thing as a bitcoin address except addresses use a special encoding so that bitcoin users can recognize them as being intended for usage within a bitcoin wallet. You're supposed to share your bitcoin address with people in order to receive money from them and it's also fine to share the underlying public key.

Public keys and private keys are both made of numbers. Imagine a simplified public keys system that works like this:

```
123 <-- that's a public key
ABC <-- that's a private key
```

```
456 <-- that's another public key
DEF <-- that's another private key
```

One of the things you can do with public key cryptography is called point addition, also known as tweaking. Point addition allows you to add two public keys and two private keys together and still get a valid keypair. For example:

```
142536 <-- that's 123 and 456 added together
ADBECF <-- that's ABC and DEF added together
```

Those are both still perfectly valid public keys and private keys, and moreover, the "combined private key" is capable of spending money that someone sent to the "combined public key."

Bitcoin Chess takes advantage of tweaking to store a chess move in a bitcoin pubkey. The game takes a string like "e2 to e4," which is how chess players sometimes say what move they are making, and it converts that into a 32 byte string. Then it tweaks the player's pubkey with those 32 bytes, resulting in a new, tweaked pubkey. The player's browser then sends some money to a bitcoin address that is generated from that pubkey, and by doing this he effectively announces his move on bitcoin's blockchain. As long as both players do this over and over, they can play a full chess game.

# Second ingredient: reveal skipping

I'm familiar with several protocols that use key tweaking and the protocols that I'm familiar with usually do something called a "commit-reveal" scheme. That is, first you commit to some data inside a tweaked bitcoin pubkey and put that pubkey on the blockchain, then you reveal your tweak and your pubkey to someone later if you ever want to prove what data you committed to. But in my scheme there's no "reveal" step. Committing the data to the blockchain *simultaneously reveals it, without needing an extra step.* This is the only thing I did that I think might be innovative. I skip the reveal step by making each player compute some commitments that their opponent might make at the end of their opponent's turn.

From any given chess position there are a limited number of possible moves that a player can make. That number rarely exceeds 40, though it can be as high as 120. When it's Player 1's turn in my chess game, Player 2 checks all of the moves that Player 1 can make. Suppose Player 1 can make two moves, e2 to e4 or d6 to d7. Player 2 knows that because he can see the board. And he also knows Player 1's pubkey because Player 1 gives it to him at the start of the game. Knowing these pieces of data, Player 2 computes two ways that Player 1 might tweak his pubkey: he might tweak it with a 32 byte string of the data "e2 to e4" or he might tweak it with "d6 to d7." Once he knows what the two possible tweaked keys are, he generates the bitcoin addresses that correspond to those pubkeys and checks to see which one his opponent sends money to. As long as Player 1 sends money to one of those, Player 2 will know which move he made. And if Player 1 sends money to some other address, it means he made an invalid move and therefore forfeited the game. Player 2's knowledge of all the moves Player 1 can make is powerful.

Another way to think of reveal skipping is that it's a way to store data inside a bitcoin address instead of just committing to it. In a commit-reveal scheme the reveal step is there so that outside observers can learn what data the user committed to, and the actual data is stored externally with some sort of host. But in my scheme the reveal step is skipped because outside observers can compute all possible pieces of data that the user might commit to. Therefore the user's commitment to that data is more than a commitment -- it's actual data storage, without the need for an external data host.

# Isn't this game a big pile of chain spam?

Yes, don't actually do this on bitcoin's mainnet. The game is supposed to be educational. By playing the game on bitcoin's testnet, you might learn something. Scroll down to "Lessons to learn from this" for some examples.

# Limitations

The blockchain logic I wrote does not implement or enforce the logic of chess. This game only *signals* your chess moves, all the actual chess logic is done client-side by your browser. Your browser checks the blockchain to see what your opponent did, then displays his move as long as it's a valid move. It may be possible to implement the rules of chess in bitcoin's built-in programming language, bitcoin script, but if it is possible it's beyond my current abilities.

I didn't implement instructions for what should happen when a pawn makes it to the other side of the board. You're supposed to change it into a queen or another piece but right now, if you try, the game doesn't do anything because I didn't tell it what to do. So be aware, you might lose money if you try to promote a pawn.

If you play too quickly or with too many tabs open you'll run into the rate limits of Blockcypher's free api. I'm using blockcypher to watch the blockchain and they currently have a rate limit of 200 queries per hour. Each chess turn involves several queries to blockcypher, and the game also queries their site every 60 seconds to check if your opponent moved yet, so it's already probably pretty close to the limit. Playing rapidly could cause your moves to suddenly stop making it onto the blockchain, or you might not be able to see your opponent's moves anymore.

The browser logic treats a move as final as soon as it enters the mempool and it won't update the board if the move gets ejected from the mempool. This means it's possible to double-spend a chess move, which I think is a hilarious concept. Maybe someday I'll add a button for that, just for fun.

There is not currently any support for the lightning network but I did make the game lightning compatible so all it needs is for someone to write a plugin for a lightning wallet to make it work. If you played it in a lightning channel, you wouldn't have any public proof of your moves or your opponent's moves, but each move would be signed and committed to inside a raw bitcoin transaction which you could keep in your wallet's database. Maybe someday.

I haven't implemented very much "game-ending" logic. There are supposed to be several ways to end the game, namely, you can forfeit by moving your coins to an address that does not commit to a valid chess move given your current position, you can checkmate your opponent or get checkmated, and you can take more than 50 moves. But the only one of those that has any actual implementation logic is the forfeit option -- your opponent's browser will say you forfeited if you withdraw your coins. If you actually manage to checkmate your opponent, the game will not do anything special, and assuming you withdraw your coins after that your opponent's browser will say you forfeited because all it knows is you moved your coins, it doesn't know the game already ended. Maybe I'll fix that someday.

# Lessons to learn from this

## Key tweaking and reveal skipping

The main lesson is that you can commit to any data in a tweaked bitcoin key. I explained that earlier. I'll just add here that you don't have to commit to a chess move, you can commit data about the location and quantity of a token you made up, or you can commit the terms of a contract, or you can commit a song you wrote to prove when you wrote it, or you can commit a love letter, or a payment memo. The sky is the limit.

## Client side validation

Your browser assesses the validity of your opponent's moves on its own. It doesn't need to ask the blockchain if your opponent's moves are valid because your browser already knows the rules of chess. If your opponent violates the rules of chess, you can even *prove* that to a third party without asking the blockchain for anything except what movement(s) happened. When I realized what that means, it was a light bulb moment for me. The moment you can prove to a third party that you permanently committed to some data, anyone can write software that attaches consequences to that proof. Did you commit to the data "send a diamond token to Bill"? Then Bill's cell phone can display a diamond picture as soon as you give him the proof, and he can share the proof with other people or put it on a website and prove *to anyone* that you signed the message "send a diamond token to Bill."

## Limiting out of band communication

To start a game with someone you only need to know their pubkey and the rules of chess. Everything else you can figure out on your own. If an outside observer wants to follow your chess game, they only need two of the pubkeys that got used in the game. Also, since you reveal your bitcoin pubkey whenever you spend your money, an outside observer might not even need to ask you for your pubkey. They can ask you for roughly what time you played and check the blockchain to see what happened on it around that time. They can filter out spends that don't look like the ones this chess game produces and then, for each remaining pubkey, check to see what pubkey came before it and what pubkey came after it (if any). Once they have a pair of pubkeys where one spent to another, they can check if the second one can be produced by tweaking the first one with a valid chess move. If it checks out, they can keep doing that in the "forward" direction to find the latest state of the game or go backward til they reach the starting positions. To do all of this someone only needs a general time when you played. I think that's neat.

# What's next?

Not sure. Maybe I'll fix the game's bugs, maybe I'll implement it inside a lightning channel, maybe I'll just let it sit here. People who play it on testnet can learn a thing or two about programming with bitcoin and that's the main thing I want. It's a brain building tool, after all, and that's what chess was always meant to be.
