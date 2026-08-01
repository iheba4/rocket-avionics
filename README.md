# Rocket avionics

Flight software for a student sounding rocket entered in the CIAA 2026 competition. I am the solo automation lead on the electrical stack, rebuilding it rather than inheriting it, so most of what is here starts from a blank file and a heritage review of what the previous team flew.

The headline feature is the airbrake system. Everything else exists to make the airbrake decision trustworthy.

## Why airbrakes are the hard part

A sounding rocket has one shot. The motor burns, the rocket coasts, and somewhere in that coast it reaches apogee. If you want to hit a target altitude rather than whatever the motor happens to give you, the only authority you have left after burnout is drag. Airbrakes deploy into the airflow and bleed off exactly enough energy to put apogee where you want it.

So the controller has to answer a question in flight, with noisy sensors and no second attempt: given where I am and how fast I am going, how much drag do I need between now and apogee. Deploy too early and you throw away altitude you cannot recover. Deploy too late and you overshoot. The control law in `control/airbrake.cpp` predicts apogee from the current state and commands a deployment fraction from the error.

## Layout

    firmware/core/     flight state machine, configuration, shared types
    firmware/sim/      aerodynamic model and flight profile generator
    firmware/test/     unit tests for the state machine
    control/           airbrake control law and its bench harness
    docs/              heritage analysis, architecture, component selection, full report

## The state machine

`firmware/core/flight_sm.cpp` is the piece everything else hangs off. A flight is a sequence of irreversible events, and getting a transition wrong is how you deploy a parachute at Mach 0.7. The states walk from pad idle through boost, coast, apogee and descent to landed, and each transition needs more than one piece of evidence before it fires. Acceleration alone is not enough to call burnout. Altitude alone is not enough to call apogee.

The tests in `firmware/test/test_flight_sm.cpp` exist because I could not fly the thing to check. They replay synthetic profiles through the state machine and assert the transitions land where they should, including the cases built to fool it: a wobble near apogee, a sensor dropout during boost, a launch detection triggered by someone bumping the rail.

## Simulation before hardware

`firmware/sim/` generates flight profiles from an aerodynamic model so the control law can be exercised without a motor. `control/bench_airbrake.cpp` runs the controller against those profiles and reports where apogee lands with and without actuation. It does not replace a flight, but it catches the class of bug where the control law is confidently wrong.

## Status

The software is written and tested in simulation. Hardware integration and flight testing have not happened yet. The docs folder tracks component selection and the reasoning behind it, including what carried over from the previous stack and what did not.

## Building

The firmware targets an ESP32-S3 with a second microcontroller as backup. The simulation and bench code are plain C++ and build on a desktop with any recent compiler, which is deliberate. The control law should be testable without flashing anything.
