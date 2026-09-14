# The cabinet

What the game asks its hardware for, what answers, and the
one thing that is assumed rather than emulated.

## The cabinet

A Lindbergh game does not talk to hardware directly; it calls SEGA's `amLib`,
which is statically linked into the binary. Every one of those calls failed
here, and the game stopped on **Error 11 — JVS I/O board is not connected to
main board** before it drew anything at all.

The game narrates all of this itself through `_sDebug::putConsole`, which goes
to a debug console the cabinet has and this does not. Binding it
(`LINDBERGH_CONSOLE=1`) is what turned the rest from guesswork into reading:

```
_sArcadeManager: amLibInit ret=-1
 SEGA BaseBD not available
_sInterfaceJvsManager: amJvsInit ret=-1
```

What answers now:

* **The base board** — `amLibInit`, `amLibIsBasebdAvailable`, `amJvsInit`, `amDongleInit`, `amDongleUpdate` and their predicates. `amJvsCheckInit` is a predicate, not a status code; answering it with the library's success value of 0 turned into "−5 JVS node(s) found".
* **A JVS I/O board**, at the transport. `amJvsSendRequest` and `amJvsRecvAcknowledge` are the only two functions replaced; the frames are real JVS (`E0 01 02 10 13` — sync, node, count, READ ID, checksum) so all 60 of the game's own packet builders and parsers run unmodified above them. It reports board identity, command and JVS revisions, and a feature list of two players, twelve buttons each, two coin slots and eight analog channels. Nothing is pressed, inserted or aimed: an attract mode wants a board that answers, not one that plays.
* **The battery-backed store**, at the four wrapper functions under the record layer, persisted to `lindbergh_nvram.bin`. Record layout, duplicate copies and CRCs are the game's own code and work untouched. Read takes the offset first and the buffer second; write takes them the other way round.

## What is assumed rather than emulated

The backup records are blank and the game cannot initialise them, because its
repair path writes through an EEPROM on an I2C bus that is not emulated. The
records then fail their CRC and the arcade framework latches **Error 15 — Game
Program Not Found** over everything.

A real cabinet ships with that store already written, so the runtime makes the
same statement: it suppresses that one error code and says so on the way past.

```
[sega] backup records are blank and cannot be initialised without the EEPROM
       bus; treating the bookkeeping as good
```

Every other error still reaches the game untouched. Emulating the EEPROM bus so
the game writes its own defaults is the next piece of work here.
