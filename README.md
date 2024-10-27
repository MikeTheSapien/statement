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

Much more information is available in the docs: https://docs.rs/statement/latest/statement/

# Example
````rust
use anyhow::{anyhow};
use statement::{StateMachineFactory, StateMachineError};

fn test_double_transition<'a>() -> anyhow::Result<()> {
    #[derive(Eq, PartialEq)]
    enum StateMachineMessage {
        GoToTwo
    }

    // State here is just an integer
    let factory = StateMachineFactory::new()
        // Evaluate all transitions in a loop
        // until no transition occurs
        .cycle(true)
        // When we receive a GoToTwo event
        // while in state 1, go to state 2
        .with_event_transition(
            &StateMachineMessage::GoToTwo,
            1,
            2
        )
        // When we transition to state 2,
        // immediately transition to state 3
        .with_auto_transition(
            2,
            3
        )
        // Lock the factory object so that
        // we can build a state machine
        .lock();

    // Build the state machine, with an empty () as data
    // (we don't care about data for this example)
    let mut sm = factory.build(1, ());

    // The StateMachine starts in state 1
    assert_eq!(1, sm.state);

    // Handling an event tells us what state we end up in
    match sm.handle_event(StateMachineMessage::GoToTwo) {
        Ok(state) => {
            assert_eq!(3, *state);
        }
        Err(StateMachineError::EffectError(from, to, e)) => {
            return Err(anyhow!("error changing state from {} to {}: {}", from, to, e));
        }
    };

    // Because of the two transitions that we defined,
    // we end up in state 3
    assert_eq!(3, sm.state);
    Ok(())
}
````

This is a longer example, showing use of state machine data 
and more complex transitions:

````rust
use std::sync::atomic::Ordering::SeqCst;
use anyhow::anyhow;
use atomic_float::AtomicF64;
use statement::FromState::{Any, AnyOf};
use statement::{StateMachineFactory, StateTransitionEffectData};
use statement::ToState::Same;

struct CalcData {
    pub input_value: AtomicF64,
    pub stored_value: AtomicF64,
}

#[test]
fn calculator_test() -> anyhow::Result<()> {
    #[derive(Copy, Clone, Eq, PartialEq, Debug)]
    enum States {
        Idle,
        Adding,
        Subtracting,
        Multiplying,
        Dividing
    }

    #[derive(Copy, Clone, Eq, PartialEq, Debug)]
    enum Events {
        Clear,
        Digit { digit: u8 },
        Add,
        Subtract,
        Multiply,
        Divide,
        Equals
    }

    impl Events {
        fn is_digit(&self) -> bool {
            if let Events::Digit { digit: _ } = self { true } else { false }
        }
    }

    let mut init_data = CalcData {
        input_value: AtomicF64::new(0f64),
        stored_value: AtomicF64::new(0f64)
    };

    let mut sm = StateMachineFactory::<Events, States, &CalcData>::new()
        // This is an example of a logger that runs before any other transition, but doesn't
        // do anything in terms of state transitions itself.
        .with_transition_effect(
            Any,
            Same,
            |d| {
                print!("user sent {:?} event", d.event);
                Ok(())
            })
        .with_predicated_transition_effect(
            Any,
            Same,
            |d| d.event.is_digit(),
            |d| {
                if let Events::Digit { digit } = d.event {
                    append_digit(d.data, digit.clone());
                }
                Ok(())
            })
        .with_predicated_transition_effect(
            AnyOf(vec![States::Adding, States::Subtracting, States::Multiplying, States::Dividing]),
            States::Idle,
            |d| {
                match d.event {
                    Events::Add | Events::Subtract | Events::Multiply | Events::Divide | Events::Equals => true,
                    _ => false
                }
            },
            |d| {
                apply_function(d);
                Ok(())
            })
        .with_event_transition_effect(&Events::Add, States::Idle, States::Adding, |d| {
            swap(d.data);
            Ok(())
        })
        .with_event_transition_effect(&Events::Subtract, States::Idle, States::Subtracting, |d| {
            swap(d.data);
            Ok(())
        })
        .with_event_transition_effect(&Events::Multiply, States::Idle, States::Multiplying, |d| {
            swap(d.data);
            Ok(())
        })
        .with_event_transition_effect(&Events::Divide, States::Idle, States::Dividing, |d| {
            swap(d.data);
            Ok(())
        })
        // This is an example of a logger that runs after any other transition, but doesn't
        // do anything in terms of state transitions itself. It continues the log lines from
        // the earlier logger
        .with_transition_effect(
            Any,
            Same,
            |d| {
                println!(", input value is {}, stored value is {}", d.data.input_value.load(SeqCst), d.data.stored_value.load(SeqCst));
                Ok(())
            })
        .lock().build(States::Idle, &mut init_data);

    let error_mapper = |_| { anyhow!("error transitioning") };
    sm.handle_event(Events::Digit {digit: 2}).map_err(error_mapper)?;
    sm.handle_event(Events::Add).map_err(error_mapper)?;
    sm.handle_event(Events::Digit {digit: 0}).map_err(error_mapper)?;
    sm.handle_event(Events::Subtract).map_err(error_mapper)?;
    sm.handle_event(Events::Digit {digit: 1}).map_err(error_mapper)?;
    sm.handle_event(Events::Multiply).map_err(error_mapper)?;
    sm.handle_event(Events::Digit {digit: 1}).map_err(error_mapper)?;
    sm.handle_event(Events::Digit {digit: 2}).map_err(error_mapper)?;
    sm.handle_event(Events::Digit {digit: 6}).map_err(error_mapper)?;
    sm.handle_event(Events::Divide).map_err(error_mapper)?;
    sm.handle_event(Events::Digit {digit: 3}).map_err(error_mapper)?;
    sm.handle_event(Events::Equals).map_err(error_mapper)?;

    assert_eq!(42f64, sm.data.input_value.load(SeqCst));

    return Ok(());

    fn append_digit(d: &CalcData, b: u8) {
        let input_value_current = d.input_value.load(SeqCst);
        d.input_value.store(input_value_current * 10f64 + b as f64, SeqCst);
    }
    fn swap(d: &CalcData) {
        let old_input_value = d.input_value.load(SeqCst);
        d.stored_value.store(old_input_value, SeqCst);
        d.input_value.store(0f64, SeqCst);
    }
    fn apply_function(arg: StateTransitionEffectData<Events, States, &CalcData>) {
        let old_stored_value = arg.data.stored_value.load(SeqCst);
        let old_input_value = arg.data.input_value.load(SeqCst);
        match arg.from {
            States::Adding => {
                arg.data.input_value.store(old_stored_value + old_input_value, SeqCst);
            }
            States::Subtracting => {
                arg.data.input_value.store(old_stored_value - old_input_value, SeqCst);
            }
            States::Multiplying => {
                arg.data.input_value.store(old_stored_value * old_input_value, SeqCst);
            }
            States::Dividing => {
                arg.data.input_value.store(old_stored_value / old_input_value, SeqCst);
            }
            States::Idle => {}
        }
    }
}
````