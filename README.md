# sonicAES
Some code inspired by a facebook conversation - a hormonal robot which
uses sounds, such as beeps and whistles, to indicate its status. See
the "notes" textfile for more information.

## Synth system
This is a jackd client implementing a blocking monosynth (since animal
vocalisations are generally monophonic).
It works by parsing commands from the user, which typically create
and edit SynthDef objects (synthesizer definitions), which contain 
GenDefs (generator definitions) and the links between them. 
The GenDefs describe things like oscillators, envelope generators etc.
and their parameters.

Issuing a note command causes a Synth (containing Gens) to be created
and added to the note command queue, along with which frequency to play.
When the audio thread has finished playing a note, it pulls a new command
off the end of the queue. Certain gens (typically envelope generators)
are nominated as "done monitors", and these all have to be in the "done"
state for the note to have finished.

The projected usage here is that the main program will send UDP messages to the synth, which it will ignore if it's currently making a sound, or queue up - it may be necessary
for complex beep sequences to be made up of multiple queued commands.

An example of some commands:
```
# start a new synth
newsynth foo

# create an env generator called env1, and make it a "done monitor" - the
# sound is finished when all donemons are complete.
# Envs use a special syntax without the "g" keyword, but their params
# can still be modified with "p" etc. Note that "done" is a separate command,
# which makes the current generator a donemon.
#
# We need to be careful with the order in which gens are created: put
# first
# those gens which are earlier in the directed graph which forms the
# synth patch. Here, the envelope generator feeds into the 
# oscillator, so we add it before the oscillator.

env         env1        0 0, 0.01 1, 0.7 0;         done

# create a sin osc called sin1 with the default params

g sin       sin1

# create a sin osc called sin2 with a freq mult of 1.01. The "p" keyword
# sets parameters in the current synth, which are key-value pairs
# separated by commas and terminated with a semicolon: "k1 v1, k2 v2;".
# There's only one here, and it's the last thing on the line so we
# don't need the terminator (that's only necessary if we have another
# command on the same line).

g sin       sin2        p freq 1.01

# create a mixer called "mix" and make it the output, and set the
# amplitudes of its outputs to 0.5

g mix2      mix         p ampa 0.5,ampb 0.5;        out

# link env1 to the amp input of sin1

env1 ->     sin1:amp

# link env1 to the amp input of sin2

env1 ->     sin2:amp

# link the sine oscillators to the mixer

sin1 ->     mix:a
sin2 ->     mix:b

# play a note
: 440;


# this creates a new synth, but here we have no comments and
# we've compressed stuff.

newsynth bar
g sin sin1 p pm 19,amp 0.1; out
g sin sin2 p freq 1.7 # phase modulator
env e 0 0, 0.05 1, 0.07 0.7, 0.12 0; done
env emod 0 0, 0.02 1, 0.07 0.2, 0.08 0.01, 0.1 0;

sin2->sin1:pm
e -> sin1:amp
emod -> sin2:amp

# selects it and plays notes on it using the note language
s bar


# 400 is a note. 
# semicolon pops off the stack, and builds a new note of that
# and adds it. The 400 is left on the stack.

:400 d ;

# Let's look at this.
# cmd       stack after and effect
# -100      400 -100
# 100       400 -100 100
# r         400 (random from -100 to 100, call it x)
# +         400+x
# d         400+x,400+x (ie. duplicate)
# ,         pop and play node
#
# 

:100 100 r+d, 100 100 r+d;


:-100 100 r+d, -100 100 r+d;
:-100 100 r+d, -100 100 r+d;
:-100 100 r+d, -100 100 r+d;
:-100 100 r+d, -100 100 r+d;

# this demonstrates how to modify a synthdef with "m"
# Change the sin1 PM and amp values:
#   m sin1 p pm 20,amp 0.1;
# Change the time of the final stage of the envelope:
#   m e p t3 0.2;

```

