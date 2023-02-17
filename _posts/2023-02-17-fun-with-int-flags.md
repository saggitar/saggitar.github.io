---
layout: post
title:  "More fun with fancy enums"
date: 2023-02-17 19:18:38 +0100
categories: coding python
---

One additonal nifty thing you can do with pythons `IntFlag` enums I already used in the [maze generator]({% link _posts/2023-02-12-mazes.md %}) is to use them for building a simple "state machine".

A state machine basically consists of a set of states, and some possible "state changes" (they actually might not change the state, i.e. there might be a "loopback" transition from a state to itself). The most naive version to implement a state machine is with a "switch" statement. Sadly this is very verbose in python (which until recently does not even have a `switch` expression). It would look something like...

```python
from enum import Enum

class State(Enum):
    A = 0
    B = 1
    C = 2


class StateMachine:
    START_STATE = State.A

    def __init__(self):
	self._state = self.START_STATE

    @property
    def state(self):
	return self._state

    @state.setter
    def state(self, new):
	state_change = (self.state, new)
	if state_change == (State.A, State.B):
	    self.on_a_b()
	    self.state = new
	elif state_change == (State.B, State.C):
	    self.on_b_c()
	    self.state = new
	else:
	    print("Impossible state change")

    def on_a_b(self):
	print("Changed from A to B")

    def on_b_c(self):
	print("Changed from B to C")
```

You could use this like

    >>> machine = StateMachine()
    >>> machine.state = State.A
    Impossible state change
    >>> machine.state = State.B
    Changed from A to B
    >>> machine.state = State.A
    Impossible state change
    >>> machine.state = State.C
    Changed from B to C
    >>> machine.state = State.A
    Already in end state

Now think about what would happen if your transition graph gets bigger. Lets say you introduce a new state `ERROR` and there should be a possible transition `on_error` from every state. The more complex the `if / elif / else` gets, the harder it is to parse when reading.

An interesting alternative is to use `IntFlag` enums (if you don't want to build a _real_ state machine and simply need something usable)

```python
from typing import *
from enum import IntFlag, auto

class State(IntFlag):
    A = auto()
    B = auto()
    C = auto()
    ERROR = auto()

    ALL = A | B | C | ERROR
    NO_ERROR = ALL & ~ERROR

    def __contains__(self, item):
	"""check if flag is set"""
	return (self & item) == item

class GetMatching:
    StateChange = Tuple[State, State]

    def __init__(self, mapping: Dict[StateChange, Any]):
	self.mapping = mapping
    
    @classmethod
    def matches(cls, base: StateChange, query: StateChange) -> bool:
	if base == query:
	    return True

	return all(
	    query_val in base_val 
	    for query_val, base_val in zip(query, base)
	)

    def __call__(self, query: StateChange):
	matching = [
	    val for enums, val 
	    in self.mapping.items()
	    if self.matches(enums, query)
	]

	if len(matching) != 1:
	    return None
	else:
	    return matching[0]

class SateMachine:
    START_STATE = State.A

    def __init__(self):
	self._state = self.START_STATE
	self.get_transition = GetMatching(self.state_changes)
    
    @property
    def state(self):
	return self._state

    @state.setter
    def state(self, new):
	if transition := self.get_transition(self.state, new):
	    transition(self)
	    self.state = new
    
    def on_a_b(self):
	print("Changed from A to B")

    def on_b_c(self):
	print("Changed from B to C") 

    def on_error(self)
	print("An error occured")
    
    state_changes = {
	(State.A, State.B): on_a_b,
	(State.B, State.C): on_b_c,
	(State.NOT_ERROR, State.ERROR): on_error
    }
```

The nice thing here is, that the logic is hidden away in the `GetMatching` object, so when we implement the `StateMachine` we _do_ see the actual state change defintions, in an easily understandable form (even allowing us to use the same callback for mutliple possible transitions) instead of having to deal with a possibly very complex, custom logic like in the previous example.

_PS.: Of course in a real application the `GetMatching` logic should have better error handling, you might look at my [reference implementation](https://github.com/saggitar/ubii-node-python/blob/main/src/ubii/framework/util/enum.py) for a not so lazy implementation._
