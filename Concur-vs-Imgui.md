
# Concur vs. Imgui

One quenstion that gets asked quite a lot is: Why can't you just use plain ImGui? Isn't the Concur abstraction just another confusing layer of abstraction which doesn't bring many benefits?

I actually started programming GUI tools in ImGui. I experienced a marked decrease in maintainability as the apps grew larger, and this actually led me to create Concur. In this post, I will try to ellucidate how ImGui doesn't necessarily scale too well.

## How to Use ImGui

ImGui has been used to create some very complex and impressive applications. For me, bigger projects always became brittle and hard to modify. Perhaps my lack of knowledge and/or discipline is to blame. I invite you to prove my points wrong, and show how things should be done instead.

I use ImGui in Python with the [PyImGui](https://github.com/swistakm/pyimgui) bindings. Code belows assumes this setup. So how can you write applications in ImGui?

### Take 1: Direct State Modification

Widgets in ImGui typically return two values: `(changed, state)`, where "changed" is a `bool` flag, and "state" is whatever interesing stuff the given widget does. For example:

```python
changed, state.name = input_text("Name", state.name, 100)
changed, state.surname = input_text("Surname", state.surname, 100)
```

This form of direct assignment to state is perhaps the most natural way to use the provided widgets. This has a drawback, however. The order of widgets matters.

Some order dependence is probably unavoidable, because state changes have to be applied in *some* order. But in this case, even semantically harmless reordering leads to visual artifacts. You end up drawing different parts of the UI with different states within a single frame, which leads to flicker/tearing.

In other words, there are spurious dependencies between widgets which are not reflected in code. These must be mentally juggled by the programmer. It feels like trying to navigate a mine field, particularly with asynchronous events and/or continuous state changes.

### Take 2: Indirect State Modification

How about updating only a state copy, and swapping it for the old state at the end of the frame?

```python
while True: # main loop
    new_state = deepcopy(state)

    changed, new_state.name = input_text("Name", state.name, 100)
    changed, new_state.surname = input_text("Surname", state.surname, 100)

    state = new_state
```

Now the application is consistent with no flicker. However, we just traded visual consistency for semantic inconsistency. We may now update `new_state` based on stale `state`. If, for example, an earlier widget changes the number of list elements, and a later widget decides to rename one of them, stuff will go wrong.

This is typically not a problem for user-triggered events, since ImGui makes sure only one interaction happens in any one frame. But in presence of async events such as video playback, this is an issue. In my experience, asynchronous behaviour tends to proliferate as the application grows and gets polished. We can give async events special treatment (execute them at the beginning), but this leads to non-composability - they can't be nested inside other code.

We and up having to mix-and-match the approaches above, but it comes with significant cognitive load.

## Take 3: event-based system

If state modification is no good, let's turn to the hitherto-ignored `changed` variable. We could return only events from our UI code, and then modify state later, based on those events. This separates UI from logic, which is nice for many reasons, not only preventing UI inconsistencies.

```python
# draw UI
name_changed, name_state = input_text("Name", state.name, 100)
surname_changed, surname_state = input_text("Surname", state.surname, 100)

# react to events
if name_changed:
    state.name = name_state
if surname_changed:
    state.surname = surname_state
```

This code is unfortunately no good, since the UI code and state reaction can't live in separate functions. There are coupled by all the event variables. This is a better take:

```python
events = []

# draw UI
changed, value = input_text("Name", state.name, 100)
if changed:
    events.append(("Name", value))

changed, value = input_text("Surname", state.surname, 100)
if changed:
    events.append(("Surname", value))

# react to events
for tag, value in events:
  if tag == "Name":
      state.name = value
  elif tag == "Surname":
      state.surname = value
```

This basically replicates the **Take 1** with lots of boilerplate, but without inconsistent frames. The order of widgets still matters (events have to be applied in *some* order), but doesn't affect visual representation. This style could support large applications, and it is actually well-proved in web development as the model-view-update architecture. It consists of three parts:

1. **Model** - our `state` class, which holds the entirety of application state.
2. **View** - the code which draws the application according to **Model**, and emits events reacting to user input.
3. **Update** - update the **Model** according to the received events.


So can you make scalable code using the ImGui paradigm? Yes, definitely. But you have to write code in a very convoluted way. Look at the last example: we created a 14 line behemoth from a 4-line example (adding two lines for Model definition). For each widget, instead of 2 lines, we now need to write 6 LOC. **Would anybody write code like that?** Not me, that's for sure. You just do what's easy, and worry about consequences later. And frequently, "later" actually means "too late".

## Take 4: Concur

Concur makes the event-based architecture idiomatic. It decreases syntactic noise and verbosity in event-based code. Maybe more importantly, it makes state changes inside GUI view code harder to write. This means you typically don't. Look at the last example rewritten in Concur:

```python
tag, value = yield from c.orr([
    c.input_text("Name", state.name),
    c.input_text("Surname", state.name),
    ])

if tag == "Name":
    state.name = value
elif tag == "Surname":
    state.surname = value
```

This is 10 lines of code (including model). For each widget, you need a minimum of 4 LOC. This is a middle ground between the two approaches outlined above. It shartens particularly the UI specification code, which now requires 1 line per widget instead of 3.

You may now think that the boilerplate is still very bad. Couldn't **most** application be written in the *naive* ImGui style (**Take 1**), and only delegate the event-based complexity (**Take 3**) to a thin outer layer where needed?

You are partly right: Concur has some boilerplate, and it gets a bit annoying. I haven't found a way yet to make it better.

But I think that using mostly *naive* style is not feasible. It is the unfortunate nature of mutable state that it is infectious. You can't really abstract it away. If an inner component modifies state directly, we are back to inconsistent UI drawing, no matter what. If we instead let components play on their own local copies of state, they may diverge in non-merge-able ways by the point we get to event handling. It seems like we truly need events all the way down.

<!--
This may actually be a good approach, which I haven't fully explored yet. It basically combines all three **Takes** above:

* Write the non-complicated UI components in the direct **Take 1** style,
* Abstract away their changes to state by `deepcopy`-ing a local copy of state in **Take 2** style,
* Apply the changes to the global state using a **Take 3** evented style.
-->
