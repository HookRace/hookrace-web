---
layout: post
title: "Haskell: Game Programming with GIF Streams"
tags: Programming Haskell
permalink: /blog/haskell-game-programming-with-gif-streams/
---

Note: This is translated from the [German original](https://github.com/def-/gifstream/blob/master/Aufgabe.pdf), which was used as a homework for the Programming Paradigms course at Karlsruhe Institute of Technology a long long time ago when I was co-holding the practical courses.
<!--more-->

Snake is a computer game in which a snake has to be moved through a playing field.
Eating food increases the snake's length.
When the snake collides with a wall or itself the game ends.

[![snake.gif](/public/snake.gif)](/public/snake.gif)

In this homework you will implement Snake in Haskell.
For this you will require the framework from the [gifstream repository](https://github.com/def-/gifstream).

The output of the game happens in an animated GIF stream, which you can watch in your browser.
64 colors are supported, which can be respresented as Int tuples of `(0,0,0)` up to `(3,3,3)`.

{% highlight haskell %}
type RGB = (Int,Int,Int)
{% endhighlight %}

A single frame of a GIF is defined as a list of rows, wherein each row is a list of RGB values.

{% highlight haskell %}
type Frame = [[RGB]]
{% endhighlight %}

The framework provides a \texttt{server} function, which runs an HTTP server on the supplied port.
The server sends each client a new frame of the GIF animation at the set interval.
In the passed logic function new frames will be generated dynamically.

{% highlight haskell %}
server :: PortNumber -> Int -> Logic -> IO ()
{% endhighlight %}

The [Snake.hs](https://github.com/def-/gifstream/blob/master/Snake.hs) file contains the basis for writing the Snake game.
Compile the game and run it (You'll need to install the `network` and `random` Haskell packages too):

{% highlight shell %}
$ ghc -O3 -threaded Snake.hs
[1 of 2] Compiling GifStream        ( GifStream.hs, GifStream.o )
[2 of 2] Compiling Main             ( Snake.hs, Snake.o )
Linking Snake ...
$ ./Snake
Listening on http://127.0.0.1:5002/
{% endhighlight %}

Open the supplied address in a browser.
By pressing the WASD keys in the terminal you can influence the GIF in your browser.

Other participants in your network can watch the GIF stream as well, by using your network IP address instead of `127.0.0.1`.

Furthermore it is possible to record the GIF stream and watch it later:

{% highlight shell %}
wget -O game.gif http://127.0.0.1:5002/
{% endhighlight %}

The most important function in [Snake.hs](https://github.com/def-/gifstream/blob/master/Snake.hs) is `logic`:

{% highlight haskell %}
logic wait getInput sendFrame = initialState >>= go
  where
    go (State oldAction snake food) = do
      input <- getInput

      -- Generate new state
      let action = charToAction input oldAction
      let newSnake = snake
      let newFood = food

      let frame = case action of
            MoveUp    -> replicate height (replicate width (3,0,0))
            MoveDown  -> replicate height (replicate width (0,3,0))
            MoveLeft  -> replicate height (replicate width (0,0,3))
            MoveRight -> replicate height (replicate width (3,3,3))

      sendFrame (scale zoom frame)

      wait
      go (State action newSnake newFood)
{% endhighlight %}

The `logic` function creates an initial state for the Snake game and passes it to the `go` function.
This function in turn uses `getInput` to read the key which was pressed last.
Afterwards a new game state is generated.
The shown frame is chosen based on the pressed key.
Finally `sendFrame` send the new frame to all connected clients.
As part of this function each frame is scaled by the `scale`.
The call to `wait` causes waiting for the set time `delay`, which defaults to 100 ms.
At the end of the function it calls itself tail recursively with the newly generated state.
The goal of this task is to extend the game logic in `logic` step by step so that you can play Snake in the end.

## Printing the Playing Field
Use the current state to generate an image and print this instead of the simple single-color image.

Write a list `fieldPositions` which saves the coordinates of the playing field in its corresponding position.

{% highlight haskell %}
fieldPositions :: [[Position]]
{% endhighlight %}

The size of the field is saved in `width` and `height`.
For a field of the size 3x4 `fieldPositions` would look as follows:

{% highlight haskell %}
fieldPositions = [[(0,0),(1,0),(2,0)]
                 ,[(0,1),(1,1),(2,1)]
                 ,[(0,2),(1,2),(2,2)]
                 ,[(0,3),(1,3),(2,3)]]
{% endhighlight %}

Implement a `colorize` function which maps a single position of the image to a color so that the new frame can be created through `let frame = map (map (colorize newSnake newFood)) fieldPositions`.
A field should be colored differently depending on whether this position is part of the snake, food or background.

{% highlight haskell %}
colorize :: [Position] -> Position -> Position -> RGB
{% endhighlight %}

## Snake Behavior

Next implement state changes so that the game logic can be written as `let newSnake = moveSnake snake food action`.

{% highlight haskell %}
moveSnake :: [Position] -> Position -> Action -> Position
{% endhighlight %}

A snake is defined as a list of positions.
The new snake receives a new head depending on the passed action.
`Action` is defined as follows:

{% highlight haskell %}
data Action = MoveLeft | MoveRight | MoveUp | MoveDown deriving Eq
{% endhighlight %}

At the tail the last element is cut off, except when the snake just reached food.

It has to be ensured that the user-chosen action is even possible.
Write a function `validateAction` so that the game logic can be extended by `let action = validateAction oldAction (charToAction input oldAction)`.
For this `validateAction` shall only return a new action when this is possible.
Otherwise the old action shall be returned.

[![listmonster.png](/public/listmonster.png)](/public/listmonster.png)
Picture from [Learn You A Haskell](http://www.learnyouahaskell.com/)

## Food Behavior

Next implement the state change of the food so that the game logic can be extended by `newFood <- moveFood newSnake food`.

{% highlight haskell %}
moveFood :: [Position] -> Position -> IO Position
{% endhighlight %}

When the snake doesn't currently eat the food the old position of the food can be returned directly.
Otherwise the new position of the food shall be chosen.
Avoid that the food appears inside of the body of the snake

Random numbers between x and y (inclusively) can be generated in the `do` syntax using `r <- randomRIO (x,y)`.
Thus import the `System.Random` module.

## End of the Game

Adapt the end of `logic` so that with `newSnake` the validity of the new state is checked.
In an invalid state the game shall be restarted by calling `initialstate >>= go`.

{% highlight haskell %}
checkGameOver :: [Position] -> Bool
{% endhighlight %}

A full solution is available in the [GitHub repository](https://github.com/def-/gifstream/blob/master/SnakeFinished.hs).

## Bonus
Program another game with GIF stream output, for example Pong, Tetris or Conway's Game of Life.

Related: [time.gif](/blog/time.gif/)
