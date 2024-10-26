# Statement - An Event-Driven State Machine
Statement is an event-driven state machine implementation library.
Statement is easy to use, and provides a great deal of flexibility
around how state machines are defined.

# How do I use it?
Statement is organized around the idea that you typically want a 
state machine per instance for a potentially large number of business
entities of the same type. These might be TCP connections, web sessions,
hotel reservations, orders, or anything else that goes through a 
predictable set of states when events happen.

# Example
````
#[derive(Eq, PartialEq)]
enum StateMachineMessage {
    GoToTwo
}

// State here is just an integer
let factory = StateMachineFactory::new()
    // Evaluate further conditions
    .cycle(true)
    .with_event_transition(
        &StateMachineMessage::GoToTwo,
        1,
        2
    )
    .with_auto_transition(
        2,
        3
    ).lock();

let mut sm = factory.build(1, ());

assert_eq!(1, sm.state);
match sm.handle_event(StateMachineMessage::GoToTwo) {
    Ok(state) => {
        assert_eq!(3, *state);
    }
    Err(StateMachineError::EffectError(from, to, e)) => {
        return Err(anyhow!("error changing state from {} to {}: {}", from, to, e));
    }
};
assert_eq!(3, sm.state);
Ok(())
````