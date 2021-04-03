This started out as a
[toy project](https://github.com/LeoKavanagh/gamethings) for playing with Scala.js, Scala 3, and
mutating state (fairly) functionally.

At time of writing it is Scala.js,
Scala 2.13 (because Scala.js DOM is not on Scala 3 yet),
and mutating state with vars and global variables.

But it does what I want it do to, and it works on the very basic web hosting
I'm using.

## Description

The whole thing is a simple web page with a few buttons on it.
The buttons let you roll the dice or draw one card at a time from a deck:

![GameThings](/assets/games.png)


I like to think that I'm helping to keep things Covid-safe by letting people
avoid touching the same dice or cards.

The web page itself is created with copied-and-pasted Scala.js in a fairly
messy and imperative way. I'll clean it up some day...

### Dice Game

The "Roll The Dice" button generates prints two random numbers
between 1 and 6, and then prints them to the screen.

### Card Game

This is the maintaining-state issue that I mainly wanted to play with when I
got started.

We need to maintain lists of seen and unseen cards in the deck.
Each time a user presses either the "Pick a Card"  or "Shuffle the Deck"
button on the web page, both lists are updated.

I tried playing with `cats` State monads but eventually just went with an immutable
`var` inside a class. Judging by [this comment](https://stackoverflow.com/questions/11386559/val-mutable-versus-var-immutable-in-scala#comment15014587_11386867)
on Stack Overflow it was a reasonable choice to make.

The "Pick A Card" button picks a card from a (shuffled) standard deck of 52
cards and prints it to the screen.

Repeated presses of the button generate new card draws. When you have dealt
all 52 cards (or earlier, if you like), you can reset the whole process and
generate a new shuffled deck by pressing the "Shuffle The Deck".

I did not put any effort into deck shuffling algorithms or
random number generators. That's a subject for another day.

[Have a go here](https://leokavanagh.com/games).

