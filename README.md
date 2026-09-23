# PLCyo

PLCyo is a project to build a custom programming language for choose-your-own-adventure and text-based interactive stories.

The idea is to make it easier to write game scenes, choices, items, and player progress without having to code everything manually in JavaScript.

## What each person is doing

- Person 1: works on the language itself. This includes the syntax, parser, and compiler logic that turns PLCyo code into something the game can run.
- Person 2: handles the browser side. This person builds the HTML/JavaScript output so the game actually works in a web page and supports things like rooms, choices, and inventory.
- Person 3: creates the demo game. This person writes the sample story and tests the language by building a real example that shows how the project works.

## Project goal

The language will let someone define rooms, items, and paths between scenes. Then the compiler will turn that into a playable web game. The main focus is on game flow and state tracking, like what the player has picked up and where they are in the story.

This is meant to stay simple and practical: build a language, generate a working game, and use a real demo to show that it works.
