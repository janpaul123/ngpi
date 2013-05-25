# Next-generation programming interfaces

**TL;DR. Programming is broken, let's fix it together. I'm proposing an architecture for a next-generation programming interface.**

There have been quite a few efforts to significantly improve our programming tools. I've been working myself for over a year on [jsdares](http://www.jsdares.com), which also [aims to make programming better](http://www.jsdaers.com/blindfold). It seems like the time is there to join efforts and build a next-generation programming interface for mainstream programming. In this document I propose some goals, requirements based on existing interface ideas, and a proposal for an architecture.

Most importantly, I'd like spark some discussion on how to make this happen. There must be many people interested in such a project, given how much attention these recent efforts have gotten.

## Goals

This project is about devising how to build next generation programming tools. "What are those," you might ask. Well, if we would know already they would hardly be next-gen.

It's easier to say what we *do not* want to do:
- We're not inventing a new programming language. We're taking the [tools first] approach.
- We're not inventing completely new paradigms. Our tools have to be familiar enough to be used by many people.
- We're not building toys. Our tools have significant professional value, at least in some domains.
- We're not breaking compatibility. Too much. Our tools work with existing technology.

It would probably be good to focus on web programming, as that's sort of a *lingua franca*.

## Requirements

These are some initial ideas of what a next-generation programming interface might do, heavily biased by my own experiences and the work of [Bret Victor](http://worrydream.com):
- Running Javascript programs normally should be without a (significant) performance impact.
- Editing the program should update the output on the fly.
- Manipulating variables in the program should be visually facilitated.
- Manipulating program structure should be visually facilitated.
- Partial statements should be auto-completed (with thumbnails).
- When hovering over a method call the corresponding output should be highlighted.
- When hovering over a method call (or argument) inline help should be shown.
- When hovering over a method call (or argument) guides should be shown in the output.
- When throwing an error, it should be as contextual/specific as possible (not just which method, but also which other methods are involved, and which arguments).
- When inspecting the program in detail, execution should be paused.
- When inspecting, program flow should be visualised by going through the program step-by-step.
- When visualising program flow, thumbnails of operations should be shown.
- When visualising program flow, hovering a call should highlight the corresponding output.
- When inspecting, hovering over output should highlight method calls (also related method calls / arguments?).
- When inspecting, previous events should be inspected as well (rewinding time).
- When rewinding time, information about the event that was triggered should be shown.
- When inspecting, it should be possible to abstract over time (highlighting method calls also for previous/future events).
- When inspecting, method calls should be editable by manipulating the output directly (e.g. dragging of shapes on the canvas).
- When inspecting, method calls should be added to the program by manipulating the output (e.g. adding shapes to the canvas).
- When inspecting, the current state of the program / outputs should be visualised.

Other non-functional requirements: 
- Features should degrade gracefully. This would mean that we can support a lot of different components without building full support for all of them right away.
- Implementation should be light on the side of components, and heavy on the side of the runner interface.

If you think of any other goals or requirements, feel free to add them!

## Architecture

Let me sketch the architecture I have in mind. Of course, this may not be the best one, so if you have any ideas please add them!

The first requirement is performance, which means that normally we just execute programs normally. Since we can live-edit programs, we simply execute the program again whenever a change is made. This means the program has to be structured to allow for this, e.g. by only setting function definitions, but many Javascript programs are already built this way, so that's acceptable. We could have an "init" or "main" method that is always called once, at the start.

Now, there are a few goals that can be met without further inspecting the code thoroughly (although often a syntax tree helps). One is the manipulation of variables, we could show interface elements for playing with numbers, with HTML colour strings, etc. We could also show an interface for changing program structure, such as abstracting code to a function or inserting a loop. This would be very similar to IDEs with refactoring help, only more visually appealing.

Current IDEs also allow for auto-completion and in-line help. This is very doable for static languages, because all types are known just by looking at the code, so the interface will know which methods to show for auto-completion and what help text to show when hovering over a method call. In Javascript things are more tricky, since types are not known beforehand. For example:

```
var someFunc = function(object1, object2) {
    object1.draw(10, 20); // What help text do we show here?

    object2. // What auto-completion do we show here?

    var object3 = object1.something;
    object3.update(); // And what do we show here?
};
```

How can we know types of variables in Javascript while it's a dynamic language? Using the good old `console.log(variable);` would do this, at runtime. We could inject code into the Javascript to store this information for us at runtime:

```
var someFunc = function(object1, object2) {
    logger.log(42, object1.draw); // Save function definition for syntax tree node 42
    object1.draw(10, 20);

    logger.log(43, object2); // Save function definition for syntax tree node 43
    // object2. // Don't render syntax node 43 since it's an autocompletion node

    logger.log(44, object1.something);
    var object3 = object1.something;

    logger.log(45, object3.update);
    object3.update();
```

However, this introduces a significant performance penalty if we always store this for every scope within the program. But we're only interested in a specific scope, since that's where we're hovering the mouse, so if the program is running fast enough (e.g. many rendering events per second), we can just inject this code for the next run, store the information, and restore the program again in the run after that.

* * * * *

This only works if the program is getting lots of events, otherwise nothing would happen! That's not acceptable, there are lots of programs that don't run often. In that case, we can still solve it, making three assumptions:
1. When there are not lots of events, we can get away with a bigger performance impact.
2. Program execution is completely deterministic.
3. A snapshot of the state of the system is made periodically.
4. All events since the last snapshot are available.

If we meet these assumptions, we can do the following: we can reset the program state to a previously saved snapshot, then generate the program which stores type information for every scope, and then replay all events that occurred since the last snapshot. Since program execution is deterministic, it will execute in exactly the same way as before, only this time while storing type information. Of course, this execution takes time, but the program isn't doing a lot anyway, otherwise we could have just injected this logging for next event execution.

This is the basic idea that allows for *all* goals/requirements stated above. After all, if we have a snapshot of a previous state of the system, and the events since then, we can go back in time to an arbitrary event. Even better, many of the requirements only make sense when pausing the program, such as inspecting the program flow at a certain event, and when the program is paused the performance penalty doesn't matter that much any more! We can save any data we want by generating a program that stores that information and reruns the events.

But the assumptions have to be met. First of all, we have to be able to store a snapshot of the state of the system periodically. This has to be done while the program is running, so it has to be fast, and not happen too often. We could even play with partial snapshots at some point for better runtime performance. We have to store the state of all global variables, of the entire DOM, and every element within the DOM, e.g. canvas.

Another tricky part is that program execution has to be deterministic. Luckily, most program execution is already deterministic, so we only need to accommodate for some objects and functions, such as `Math.random`, `Time`, `XMLHttpRequest`, etc. Most importantly, we have make sure that non-deterministic functions that we don't support get completely disabled.

Finally, to make all the goals / requirements work, we'll have to devise an API between the components (DOM, canvas, etc.) and the editing interface, describing where we store and how to access all the data needed.

## Call to action

Let's form a community of people who want to make this happen! Please leave a comment below if you're interested. What would you want in a next-generation programming interface? How could that work? And how to make this happen?
