# The UNK Language Manual

*For unkle 1.0. UNK is the language Origin Systems wrote Ultima VIII:
Pagan's game logic in; unkle decompiles the game's compiled archives to UNK
and compiles UNK back into archives the original game runs.*

This manual is for people writing mods. It explains the language as unkle
compiles it, what the compiled code does in the game, and how the game's
own authors wrote it. Building and command-line use are in `README.md`; the
compiled form (archives, classes, opcodes) is in
`USECODE_VM_REFERENCE.md`.

**Where the facts come from.** Everything here was measured, not assumed:

- **the archives** -- the game's `EUSECODE.FLX` (versions 2.12 and the 1994
  release), `FUSECODE.FLX` and `GUSECODE.FLX`, which unkle decompiles and
  rebuilds byte for byte;
- **the authors' sources** -- 28 of Origin's original `.UNK` files that
  survive, about a quarter of the game's code. "The authors write..." below
  means these files;
- **the game itself** -- where a claim is about what happens at run time,
  it was tested in the original DOS game, and says so.

Every complete example in this manual (`unk` blocks) is compiled by
`check_manual.py` against the decompiled game before each release.

## Contents

1. [A first look](#1-a-first-look)
2. [Source files](#2-source-files)
3. [Classes](#3-classes)
4. [Functions and events](#4-functions-and-events)
5. [Types, variables and names](#5-types-variables-and-names)
6. [Expressions](#6-expressions)
7. [Statements](#7-statements)
8. [Calls](#8-calls)
9. [Processes](#9-processes)
10. [Searching the world: `foreach`](#10-searching-the-world-foreach)
11. [One source, three languages](#11-one-source-three-languages)
12. [Writing a mod](#12-writing-a-mod)
13. [Lessons from the game](#13-lessons-from-the-game)
14. [How the authors wrote it](#14-how-the-authors-wrote-it)

Appendices: [A. Grammar](#appendix-a-grammar) ·
[B. Built-in names](#appendix-b-built-in-names) ·
[C. The library: FREE and METHOD](#appendix-c-the-library-free-and-method) ·
[D. What the decompiler writes that you need not](#appendix-d-what-the-decompiler-writes-that-you-need-not) ·
[E. Error messages](#appendix-e-error-messages)

---

## 1. A first look

### 1.1 What usecode is

Ultima VIII's engine draws the world, moves things and runs the
interface. Almost everything a character or an object *does* -- what
Devon says when clicked, when the guards come, what a potion does -- is
**usecode**: small programs compiled for a virtual machine inside the
engine and shipped in one archive per language, `EUSECODE.FLX` for
English.

The archive holds **classes**. A class belongs to a kind of object:

- an **NPC class** (`npcclass`) to one character -- DEVON is Devon the
  fisherman;
- a **type class** (`typeclass`) to every object of one shape -- every
  barrel, every potion of a kind;
- a **module** to no object at all: a library of routines other classes
  call (FREE, METHOD, the spell modules).

The engine runs a class's code when something happens to its object -- it
is looked at, used, hit, comes on screen -- by starting the matching
**event handler** as a **process**. Processes run side by side, take turns,
wait for one another and can be stopped by type. Most of the language's
character comes from that.

### 1.2 A first change

Devon stands on the dock where the game begins. Clicking him once runs
DEVON's `look` handler, which shows his name. This patch replaces it:

```unk
process DEVON::look()
{
    if isDead()
    {
        bark("The late fisherman");
        return;
    }
    bark("Devon, who found you on the dock");
}
```

Put it in `src/custom/patch/devon.unk` and build:

```
unkle src USECODE.FLX -l E
```

`EUSECODE.FLX` now holds the original game with that one function
replaced. Copy it into the game's `USECODE` folder and click Devon.

What the patch shows:

- a function is written `process CLASS::name(...)` with its body in
  braces;
- `isDead()` and `bark(...)` are **intrinsics**, functions the engine
  provides, called here on Devon himself because nothing else is named;
- `if` takes a condition with no parentheses needed, and `return;` ends
  the handler.

### 1.3 A first class

A new class needs a declaration that lists its functions:

```unk
typeclass Greeter inherits Item
{
    process greet();
    routine double(n : Number) : Number;
}

process Greeter::greet()
{
    count : Number;

    count = double(21);
    bark("The answer is " + numToStr(count) + ".");
}

routine Greeter::double(n : Number) : Number
{
    return n * 2;
}
```

Nothing in the world has a new class's shape, so the engine never calls it
by itself; an existing class calls it (section 12.3):

```unk
// in the same mod: reach the new class from Devon
process DEVON::look()
{
    spawn GREETER::greet() -> return;
}
```

That `spawn ... -> return;` starts `greet` as its own process and lets
`look` carry on without waiting (section 8.5). `greet` runs with Devon as
its object, so Devon says the line.

---

## 2. Source files

### 2.1 Files and encoding

A source file holds one class (or one module) and is named after it:
`DEVON.unk`. Decompiled files are UTF-8; unkle also reads files in the
game's own DOS code page, CP437, telling the two apart by their bytes
(no file in either encoding has ever been misread as the other).

The game displays CP437 only. A character outside it -- a curly quote, an
emoji -- cannot be shown, and unkle refuses it with a message naming the
character:

```unk-error
// expect: has no cp437 byte
process DEVON::look()
{
    bark("“Quoted”");
}
```

### 2.2 Comments and white space

```unk-frag
// decl: x : Number;
// to the end of the line
/* across
   lines */
x = 1;   // white space, line breaks and indentation are free
```

Outside string literals, white space never changes the compiled code: a
condition may be broken across lines anywhere, braces may stand where you
like.

### 2.3 Names

Names are letters, digits and `_`, not starting with a digit, and are
case-sensitive: `count` and `Count` are different. Class names are the
exception: `Devon`, `DEVON` and `devon` name the same class, because the
archive stores class names in capitals.

Reserved words:

```
npcclass typeclass inherits process routine unkroutine constant globalflag uses
if else while for to foreach where within cells units of in container
recursive on surface switch case default return break suspend startup
exclusive spawn not and or modulo true false
```

### 2.4 Numbers

Numbers are written in decimal, or in hexadecimal with `0x`:

```unk-frag
// decl: n : Number;
n = 20;
n = -30;
n = 0x00FF;
```

The literal's **spelling decides its width** in the compiled code, which
matters only when matching original bytes:

| Written | Pushed as |
|---|---|
| `0` to `127`, `-1` to `-128` | a byte (sign-extended) |
| anything else | a word |
| `0x` with 1-2 digits | a byte -- `0xFF` is the byte `FF` |
| `0x` with 3-4 digits | a word -- `0x0001` is a word holding 1 |
| `0x` with 8 digits | a double word |
| `5:Word`, `200:Byte`, `4:Long` | the width named |

Values themselves are 16 bits (or 32 for `Long`, section 5.1); the width
only selects the instruction. New code can ignore all of this.

### 2.5 Strings

```unk-frag
// decl: s : String;
s = "Greetings, traveller. ";
s = "She said \"no\".";
s = "a backslash: \\";
```

`\"` is a quote and `\\` a backslash. `\&` produces nothing (the authors
put it before a closing quote: `"Enter quantity. \&"`). `\f` and `\g`
begin the French and German text of a multilingual literal (section 11).

**A literal may run across lines, and each line break becomes exactly one
space.** A break and the white space around it collapse into a single
space in the compiled string:

```unk-frag
bark("Why are you
    in my home?");
```

compiles to `"Why are you in my home?"`. So break a long line only where
the text has one space.

**Literals joined with `+` are one literal.** `"a" + "b"` compiles to the
single string `"ab"`; the authors use it to continue a long text:
`"... first part"` on one line, `+"second part"` on the next.

### 2.6 List literals

```unk-frag
// decl: names : String List; nums : List;
names = <<"Yes.", "No.">>;
nums = <<1, 2, 3>>;
```

Section 6.6 covers lists.

### 2.7 What a file holds

In order:

- optional `uses "name";` lines, which unkle reads and ignores;
- optional `constant NAME = value;` lines (they may also stand between
  functions);
- optional `globalflag NAME;` lines, which unkle reads and ignores (the
  global table is the authority, section 5.6);
- the class declaration, unless the file is a module;
- the functions.

The authors' files begin with lines like `uses "free";`, importing the
other source files a class relies on. unkle accepts them and ignores them:
it always knows every class, global and intrinsic, and the compiled game
keeps no trace of them, so a decompiled file has none.

```unk
uses "free";
uses "npc";

typeclass Bell inherits Item
{
    process use();
}

process Bell::use()
{
    bark("Dong.");
}
```

---

## 3. Classes

### 3.1 The three kinds

| Kind | Declared | Its object | Class numbers |
|---|---|---|---|
| NPC class | `npcclass NAME inherits Npc` | one character: class `0x400 + n` belongs to NPC number `n` | `0x0400`-`0x04FF` |
| type class | `typeclass NAME inherits Item` | every object whose shape number is the class number | the rest |
| module | no declaration | none | the rest |

The class number *is* the connection: there is no table saying which class
belongs to which object. The Avatar is NPC 1 and class `0x0401`, AVATAR;
Devon is NPC 2, class `0x0402`. A type class is run for every object whose
shape matches its number.

A module is a class with no event handlers. Its functions are called by
other classes and run on the caller's object. FREE and METHOD are the
game's general library (Appendix C); AIRSPEL and the other spell modules
hold one function per spell.

### 3.2 The declaration

```unk
typeclass Lantern inherits Item
{
    process look();
    process use();
    process flicker();
    routine brightness() : Number;
}

process Lantern::look()
{
    bark("A lantern");
}

process Lantern::use()
{
    spawn flicker() -> return;
}

process Lantern::flicker()
{
    bark("It flickers.");
}

routine Lantern::brightness() : Number
{
    return 3;
}
```

The declaration lists every function the class defines, each with its
kind (`process` or `routine`), its parameters and, when it returns one, its
result type. The functions are then defined outside it, each named
`CLASS::name`. The order of definitions is the order in the archive.

- **The name.** An existing class keeps its archive name (`DEVON`, at most
  8 letters in the shipped game). A new class's name is up to 12 ASCII
  characters and must not be an existing class's. The authors used longer
  names (`npcclass Aramina`, `typeclass TypeAirTitan`) that the archive does
  not keep; decompiled files use the archive name.
- **`inherits`.** NPC classes inherit `Npc` and type classes `Item`. The
  archive records no base class; unkle notes a mismatch (an `npcclass`
  inheriting `Item`) and otherwise treats `inherits` as documentation.
- **A module** has no declaration; its functions are written with the
  module's name (`process FREE::wait(...)`) or, as the authors did, with
  none (`process wait(...)`).

### 3.3 State: globals, not fields

A class has no fields. An object's own data is what the engine keeps for
it -- position, shape, frame, quality, quantity, status flags, hit points
-- read and written with intrinsics (`getFrame()`, `setFrame(...)`,
`getQuality()`, ...). Anything else that must last lives in a **global**
(section 5.6): `devonInRule`, `toranDead`.

The one exception is FREE's `Timer`, a two-field structure FREE uses
inside `wait` (section 5.8).

### 3.4 New classes

A new class (in a mod's `new/` folder) gets a free class number of its
kind: an NPC class one in the NPC block, a type class one outside it. It
may have event handlers, but the engine will not run them on its own --
nothing in the world has its number -- so it is reached when an existing
class calls one of its functions. A module cannot be
added; a type class with only plain functions does the same job.

---

## 4. Functions and events

### 4.1 `process` and `routine`

Every function is one of two kinds, and the kind decides how it is called:

| | `process` | `routine` |
|---|---|---|
| runs as | a new process, scheduled by the engine | part of the caller's process |
| may wait (`wait`, a call that waits, `process { }`) | yes | no |
| called with | a `spawn` instruction | a direct `call` |
| a value | read when it ends (section 8.4) | returned at once |

A caller writes both the same way -- `flicker();`, `x = brightness();` --
and unkle picks the instruction from the callee's kind. Use a `routine`
for a calculation and a `process` for anything that takes time or is
started as an activity. (`unkroutine`, which appears once in METHOD, is
compiled as a `routine`.)

### 4.2 Parameters and results

```unk
typeclass Scales inherits Item
{
    routine heavier(a : Number, b : Number) : Number;
    process weigh(what : Item);
}

routine Scales::heavier(a : Number, b : Number) : Number
{
    if a > b
    {
        return a;
    }
    return b;
}

process Scales::weigh(what : Item)
{
    if heavier(what.getQuantity(), 10) = 10
    {
        bark("Light.");
    }
}
```

Parameters are `name : Type`, separated by commas. A result type follows
the parameter list. `return value;` ends the function with that result;
`return;` ends it with none. A function that reaches its closing brace
returns.

Declare `: Boolean` for a function returning a truth value: the result is
then handed over as a byte, as the game's own `give` and `take` do. Any
other result is a word.

### 4.3 Event handlers

A process named after an engine event is that event's **handler**; the
engine starts it when the event happens to the class's object. The event
slots the authors' sources implement:

| Slot | Handler | When | Process type |
|---|---|---|---|
| `0x00` | `look()` | the object is clicked once | `0x208` (`PT_LOOK`) |
| `0x01` | `use()` | the object is double-clicked | `0x207` (`PT_USE`) |
| `0x04` | `cachein()` | the object is loaded into the game | `0x213` |
| `0x05` | `hit(r : Referent, vel : Word)` | the object hits `r` | `0x20B` |
| `0x06` | `gotHit(r : Referent, vel : Word)` | the object is hit by `r` | `0x20C` |
| `0x08` | `schedule(currentTime : Long)` | (NPCs) time to check where to be | `0x215` |
| `0x0F` | `enterFastArea()` | the object comes into the active area around the Avatar | `0x21B` |
| `0x10` | `leaveFastArea()` | the object leaves it | `0x21C` |
| `0x11` | `cast(r : Referent)` | (NPCs) the Avatar casts a spell nearby -- the shipped handlers call the guards | `0x21D` |
| `0x12` | `justMoved()` | the object has been moved | `0x22D` |
| `0x13` | `AvatarStoleSomething(r : Referent)` | the Avatar took `r`, which is the NPC's | `0x22E` |
| `0x15` | `guardianBark(n : Word)` | (AVATAR) the Guardian speaks | `0x200` (`PT_BARK`) |

There are 32 slots in all. The other names -- `anim`, `hatch`, `release`,
`equip`, `combine`, ... -- are in the names file the source folder carries
(`.unkle/names.json`, `events`), and a process with one of those names is
that slot's handler. The "process type" column is the type each handler
runs as (section 9.2); the slots the game never uses have none recorded.

**`schedule` takes a parameter** that the event list omits: the engine
passes the time, a `Long`. No handler in the game reads it, but it is
there, and a handler must declare it as the authors do:
`process schedule(currentTime : Long);`.

**A handler begins by claiming its object and type.** Before your first
statement, unkle compiles the prologue every handler in the game has: the
process is registered as type T for this object, and **if another process
of type T is already running for the object, this one ends at once**. So
one object never runs two of the same handler at a time: a second `look`
started while the first is still running ends at once. This matters most
when one class calls another's handler (section 13.1).

### 4.4 Object tags: `[[type, object]]`

A plain function has no prologue unless it asks for one, with an **object
tag**:

```unk
typeclass Chime inherits Item
{
    process ring();
}

process Chime::ring()
{
    [[PT_FX, NULL_REF]];
    safePlaySFX(MAGIC2, 100);
}
```

`[[PT_FX, NULL_REF]];` registers this process as type `PT_FX` belonging
to no object. Written as a statement it can stand anywhere; the authors put
it first. The first element is a process type (section 9.2), the second an
object: `referent` (this object), `NULL_REF` (none) or any item.

`PT_DEFAULT` is not a type of its own: it means "the type this function
would have anyway" -- the event's own type in a handler, 1 elsewhere.

A tag may also stand **before the function's name**, in the declaration or
the definition, as the authors wrote it for functions that run as an
activity of their object:

```unk
typeclass Potion2 inherits Item
{
    process [[PT_DEFAULT, referent]] glow();
}

process Potion2::glow()
{
    bark("It glows.");
}
```

The two places compile alike except for one detail of width that matters
only for matching original bytes (the tag before the name pushes its type
as a word, the statement by the literal rule of section 2.4).

A tag only registers. To end the process when another of the same type is
running, as handlers do, write `exclusive;` after it.

### 4.5 Module functions

A module's functions have no class of their own to run for, so how each
is written decides whether it has an object at all:

| Written | Object | Parameters | Called |
|---|---|---|---|
| `process WATRSPEL::cast(...)` -- the module's own name in front, as decompiled files write them | the caller's object, as `referent` | after the hidden object | `WATRSPEL::cast()` from a class; bare from the same module |
| `process cast(...)`, or a first parameter named `arg0` | none -- `referent` means nothing here | from the start of the frame | the same |
| `routine Item::toggleStatus(...)` | the object it is called on | after the hidden object | `npc.toggleStatus(INVISIBLE)` (8.3) |

The game's spell modules use the first form: AVATAR's old cheat menu
called `ERTHSPEL::...` and the spell ran on the Avatar, its animations
played with a bare `realDoAnim(...)`, and the functions it called inside
the module received the Avatar too. FREE's `wait(time)` and `random(x)` use
the second. Use the first for anything that acts on a character:

```unk
// a spell added to an existing module
process WATRSPEL::sparkle()
{
    [[544, referent]];
    realDoAnim(AS_CAST1, Npc::getDir());
    safePlaySFX(MAGIC2, 100);
}
```

```unk
// in the same mod
process DEVON::look()
{
    WATRSPEL::sparkle();
}
```

Devon casts. (unkle 1.0 compiled a new module function written the first
way as though it had no object, so its parameters and `referent` were read
from the wrong place; fixed in 1.0.1, found by the first mod built with it.)

A module process has a prologue only where it has a tag, which decompiled
modules write before the name: `process [[PT_DEFAULT, referent]]
METHOD::enterConvo()`.

---

## 5. Types, variables and names

### 5.1 Types

| Type | Bytes | Holds |
|---|---|---|
| `Number` | 2 | a signed 16-bit integer; also the default for unknown type names |
| `Word` | 2 | the same, by another name |
| `Byte` | 1 | 0-255 |
| `Boolean` | 1 | `true` (1) or `false` (0) |
| `Item` | 2 | an object in the world, by its item number; 0 is none |
| `Npc` | 2 | a character (an `Item` that is an NPC) |
| `Referent` | 2 | an item number, as a parameter type |
| `String` | 2 | a string (a handle; the text lives elsewhere) |
| `String Reference` | 2 | a string parameter passed through, as METHOD's `lookBark` takes |
| `List` | 2 | a list of words |
| `String List` | 2 | a list of strings |
| `Long`, `DWord` | 4 | a signed 32-bit integer |
| `Tick` | 4 | a timer value in 60ths of a second |
| `WorldPoint` | 5 | `x`, `y` (words) and `z` (a byte): a position |
| `Alti`, `Dir` | 1 | a height; a compass direction 0-7 |
| `Coord`, `Type`, `Frame`, `Quality`, `Quantity`, `MapNum` | 2 | words, named for what they hold |
| `Timer` | 8 | FREE's timer structure, inside FREE only (section 5.8) |

The authors also wrote `Item List` and `Number List` for lists of
words; unkle accepts them, and decompiled files say `List`.

### 5.2 Declaring variables

```unk-frag
count : Number;
a, b, c : Number;
done : Boolean = false;
here : WorldPoint;
```

A declaration is `name : Type;`, several names may share one, and an
initial value may follow `=`. Declarations may stand anywhere in a
function body -- the authors put them first, or first in a `startup`
block -- and are all hoisted: every variable belongs to the whole function
and is zero when it starts.

A variable takes space in the function's frame whether it is used or not,
so a declaration that nothing uses still moves every variable after it.
That matters only when matching original bytes (decompiled files keep such
gaps as `_pad1 : Byte[2]`).

### 5.3 This object: `referent`

`referent` is the object the function runs for -- the clicked barrel, the
character whose schedule it is, or, in a called function, the caller's
object:

```unk-frag
// decl: other : Item;
if referent = Avatar
{
    return;
}
other = referent;
```

A call with no object in front runs on `referent`: `bark("Hi")` is
`referent.bark("Hi")`. `NULL_REF` is no object (0).

### 5.4 Characters by name

Every NPC class's name is also that character's item number:

```unk-frag
Avatar.bark("I see.");
Devon.bark("Watch the nets!");
if referent = Mordea { bark("Majesty."); }
```

`Avatar` is 1, `Devon` 2. The authors also wrote `Avatar:Referent`; it
means the same. The names come from the class names, so they cannot be
mistyped silently -- an unknown name is an error.

### 5.5 Items

An `Item` variable holds an item number; methods called on it run on that
object:

```unk-frag
// decl: coin : Item; n : Number;
coin = Avatar.getEquip(EQ_BACKPACK);
if not coin
{
    return;
}
n = coin.getQuantity();
```

`x.referent` is the item number itself, the way the authors copy one item
into another: `item.referent = r;` is `item = r;`. An item tests false when
it is 0.

### 5.6 Globals

Globals are the game's flags and counters: one table of named bits shared
by every class, saved with the game. A global is read and written by name:

```unk-frag
// decl: n : Number;
if devonInRule = true
{
    toranDead = false;
}
n = devonJob;
```

Each global has a width in bits: most are one bit (`true`/`false`), some
hold small numbers (`devonJob` has two bits, 0-3). Only that many bits of
a value are stored.

The names come from the source folder's `.unkle/names.json` (`globals`),
filled from Pentagram's lists when the game was decompiled with
`--globals`; without them globals are named by position
(`global_000Fh_1bit`). An entry is keyed by its bit position:

```
"002E": [1, "pyrosMet", "unused"]
```

**To add a flag of your own**, rename one of the 59 positions marked
`unused` -- flags the game declares and never reads -- and use the new
name. The game's globals fill bits `0x000`-`0x250`; the virtual machine has
room for more, but whether the original game saves positions past the
last one it declares has not been tested. Two names for one position are
refused. Because the table lives in the source folder, not in the mod's
layer, a mod that adds a flag must ship its `names.json` change too.

A local variable with a global's name hides the global in that function.

### 5.7 Constants

```unk
constant TRIES = 3;

typeclass Persist inherits Item
{
    process nag();
}

process Persist::nag()
{
    i : Number;

    for i = 1 to TRIES
    {
        bark("Listen to me!");
        wait(60);
    }
}
```

`constant NAME = value;` names a value for the whole file. It is replaced
by its value when compiled.

unkle also knows the names the authors used for the game's own values --
process types, animations, item types, sound effects, map numbers, status
flags and more (Appendix B) -- and `true`, `false`, `TRUE`, `FALSE` and
`NULL_REF`. These win over a `constant` of the same name, so give your own
constants distinct names.

Status flags and damage types come from `.unkle/names.json` (`flags`): the
authors' names, plus ScummVM's when the game was decompiled with `--flags`.
Add a name there to use it.

### 5.8 Structures: `WorldPoint`

```unk-frag
// decl: p : WorldPoint;
p.x = Avatar.getX();
p.y = Avatar.getY();
p.z = Avatar.getZ() + 20;
if canSafeTeleport(p, MapEastTenebrae) { safeTeleport(p, MapEastTenebrae); }
```

A `WorldPoint` is `x`, `y` and `z`. It is passed to a function whole.

FREE defines one more structure, `Timer` (`startTick` and `durationTick`,
with the routines `Timer::start` and `Timer::isDone`), which its `wait`
uses. It belongs to FREE; elsewhere, use `wait`.

### 5.9 Strings and lists are values

A `String`, `List` or `String List` variable owns its value: assigning
copies it, and the compiled code frees it when it is replaced or the
function ends. You never free anything by hand.

---

## 6. Expressions

### 6.1 Operators

From loosest to tightest binding:

| Level | Operators | |
|---|---|---|
| 1 | `+=` `-=` `^=` | in expression position: list append, subtract, union (6.6) |
| 2 | `or` | |
| 3 | `and` | |
| 4 | `=` `!=` `<` `>` `<=` `>=` | comparisons; `=` is also assignment |
| 5 | `+` `-` | add; strings join; lists append |
| 6 | `*` `/` `modulo` `&` `\|` `<<` `>>` | multiply, divide, remainder, bit and/or, shifts |
| unary | `not` `-` `~` `&` | negate, bit not, address of |

Operators of one level group from the left: `a - b - c` is `(a - b) - c`.

Two rules differ from most languages, and both bite:

- **`not` takes everything up to the next `or`.** `a and not b and c`
  means `a and not (b and c)`. Write `a and (not b) and c` for the other
  reading. (This is how the game's own compiler read it; CYRRUS's source
  depends on it.)
- **`&` and `|` bind like `*`**: tighter than `+`, `-` and comparisons.
  `x & 4 = 0` is `(x & 4) = 0`, as you would hope, but `a + b & 0xFF` is
  `a + (b & 0xFF)`, and `a | b + 1` is `(a | b) + 1`. Bracket mixed
  arithmetic.

There is no `%`; the remainder is `modulo`. There is no `&&`, `||` or `!`:
write `and`, `or`, `not`.

### 6.2 Arithmetic

Arithmetic is on signed 16-bit words, or on 32 bits when either side is a
`Long`, `DWord` or `Tick`. Division truncates. `x modulo 8` is the
remainder.

### 6.3 Comparison and truth

`=` compares (and, as a statement, assigns: section 7.1). Comparisons give
1 or 0. Anything non-zero is true where a condition is expected, so `if
count` and `if not backpack` are ordinary.

Strings compare with `=` and `!=` by content: `if answer = "bye"`.

`true` and `false` are 1 and 0. The authors write them for one-bit
globals, `Boolean` variables and flag tests: `if (getStatus() & INVISIBLE)
= false`.

### 6.4 Strings

`+` joins strings. A number is turned into text with the `numToStr`
intrinsic:

```unk-frag
// decl: line : String; day : Number;
line = "Day " + numToStr(day) + " of my exile.";
line = getName() + ", go away!";
```

### 6.5 Bits

`&`, `|` and `~` work on bits. The status flags are the common case:

```unk-frag
if (getStatus() & INVISIBLE) = false
{
    orStatus(INVISIBLE);
}
andStatus(~OWNED);
```

### 6.6 Lists

A list holds words (`List`) or strings (`String List`):

```unk-frag
// decl: options : String List; picks : List; n : Number;
options = <<"Yes.", "No.">>;
options = options + <<"Maybe.">>;
options ^= <<"Goodbye.">>;
options -= <<"No.">>;
picks = <<3, 5, 8>>;
n = picks[1];
if 5 in picks { n = 5; }
```

- `+` appends: `list + <<x>>`.
- `list ^= <<...>>;` adds what the list does not already hold (a union);
  `list -= <<...>>;` removes elements. The authors build conversation
  choices this way.
- `list[i]` is element `i`, **counting from 1**.
- `x in list` tests membership.
- `foreach x in list { ... }` walks a list (section 7.7).

### 6.7 Calls in expressions

A call that returns a value can stand in an expression: `if isDead()`,
`n = urandom(10) + 1`. A `process` called for its value runs to its end
first; the expression continues when it has (section 8.4).

### 6.8 Width casts

`expr : Type` gives a value an explicit width: `100:Long` passes a
double word where a word would otherwise be pushed. The common use is a
`Long` or `Tick` argument (`timer.start(120:Tick)`) and, in decompiled
text, keeping an original instruction's width (`[[1:Word, ...]]`).

---

## 7. Statements

### 7.1 Assignment

```unk-frag
// decl: n : Number; s : String;
n = 5;
n += 2;
n -= 1;
n++;
n--;
s = s + "!";
```

`=` at the start of a statement assigns. `x += v` and `x -= v` are
`x = x + v` and `x = x - v`; `*=` and `/=` also exist. `x++` and `x--` add
and subtract one.

### 7.2 `if`

```unk-frag
// decl: n : Number;
if n > 10
{
    bark("Many.");
}
else if n > 1
{
    bark("Some.");
}
else
{
    bark("Few.");
}
```

The braces are required; parentheses around the condition are not. An
empty `else { }` is allowed and is not always idle -- decompiled code
needs some of them to reproduce the original bytes (Appendix D).

### 7.3 `switch`

```unk-frag
// decl: answer : String;
switch answer
{
    case "name"
    {
        bark("I am Devon.");
    }
    case "job", "work"
    {
        bark("I fish.");
    }
    default
    {
        bark("What?");
    }
}
```

A case may list several values. There is no fall-through and no `break`
inside a case. It is the same as an `if` / `else if` chain comparing the
value with each case in order, and compiles to exactly that, so use
whichever reads better; the authors used both.

### 7.4 `while`

```unk-frag
// decl: n : Number;
n = 25;
while (not getTarget()) and n
{
    wait(1);
    n--;
}
```

### 7.5 `for`

```unk-frag
// decl: i : Number;
for i = 1 to 5
{
    bark(numToStr(i));
}
```

`for v = first to last` runs with `v` from `first` up to and including
`last`, adding one after each pass. The limit is compared each time.

### 7.6 `break`

`break;` leaves the innermost `while`, `for` or `foreach`. There is no
`continue`.

### 7.7 `foreach` over a list

```unk-frag
// decl: who : String; names : String List;
names = <<"Devon", "Mordea">>;
foreach who in names
{
    bark(who);
}
```

`foreach` also searches the world for items; that form has a section of
its own (section 10).

### 7.8 `return`

`return;` ends the function; `return value;` ends it with a result. In a
process, `return` ends the process.

### 7.9 Calls as statements

A call on its own is a statement: `bark("Hello.");`. How a call to a
process waits is section 8.5.

### 7.10 `startup`, `process` and `suspend`

A process may be split into a part that runs once and a part that repeats:

```unk
typeclass Pacer inherits Item
{
    process patrol();
}

process Pacer::patrol()
{
    [[PT_ACTIVITY, referent]];
    startup
    {
        steps : Number;

        steps = 0;
    }
    process
    {
        steps++;
        if steps > 100
        {
            return;
        }
        wait(30);
    }
}
```

- `startup { }` runs once, when the process starts. Declarations are
  usually put here.
- `process { }` **repeats forever**: each pass begins by yielding to the
  other processes, runs the body, and goes round again. It ends only by
  `return` (or by being stopped from outside, section 9.3).
- `suspend;` inside `process { }` yields and starts the next pass at once,
  like a `continue` for the endless loop. Outside a `process { }` block it
  just yields.

This is how the game writes every activity that keeps going -- an NPC
pacing, a door waiting for its key, `wait` counting its time. A `process
{ }` block's first action is a yield, and that matters for chains (section
8.6).

### 7.11 `exclusive`

`exclusive;` ends the process at once if another process of its type is
already running for its object (the check every handler makes in its
prologue, section 4.3). Use it after an object tag to make a plain process
behave like a handler.

---

## 8. Calls

### 8.1 What a name resolves to

A call `name(...)` with nothing in front is looked up in this order:

1. a function of the class being compiled;
2. an intrinsic of that name;
3. a library function: FREE, then METHOD, then the other classes with
   named library functions (CALGUARD's `callTheGuards`, DOOR_NS's
   `doorDriver`, ...).

So `wait(30)` is FREE's `wait` and `bark("Hi")` the engine's `bark`.
`CLASS::name(...)` names a function exactly, in any class:
`CALGUARD::callTheGuards(0)`, `ARMITAGE::converse()`.

Two refinements, both from the authors' own code:

- **An explicit object prefers the intrinsic.** `x.look()` is the engine's
  `look` on `x` even inside a class that has its own `look`.
- **The argument count decides between a class's function and an
  intrinsic of the same name.** METHOD defines a three-argument
  `receiveHit` that calls the engine's four-argument one by the same name.

A few library functions are known to replace the intrinsic they share a
name with (FREE's `scrollTo` over the camera's); unkle carries that list.

### 8.2 Intrinsics

Intrinsics are the engine's built-in functions: about 250 of them, from
`bark` and `getX` to `teleport` and `playSFX`. The source folder's
`.unkle/names.json` lists them (`intrinsics`), one signature per number,
from the ScummVM header given when the game was decompiled:

```
process Item::bark(char* str)
word Item::getX()
Kernel::resetRef(uword, ProcessType)
strptr process Item::ask(uword slist)
```

unkle reads from each signature:

- **the name and the number of arguments** -- a call must match one;
- **`Item::` or `Npc::` in front**: called on an object (`referent` if
  none is written). Where `Item::` and `Npc::` intrinsics share a name and
  arity, the declared type of the object decides: an `Npc` variable calls
  the `Npc::` one;
- **`process` in front**: the intrinsic runs as a process, and a call
  waits for it like a call to a usecode process (8.5); `strptr` means it
  gives back a string;
- **a pointer parameter** (`word*`, `uword*`, `WorldPoint*`): the
  intrinsic writes the result there, and you pass the variable itself;
- **a namespace** (`Kernel::`, `Camera::`): written with it,
  `Camera::setCenterOn(0)`.

This manual names the intrinsics the authors' code uses, by the names they
used, but does not list them all: the table is ScummVM's work, under its
own licence, and you have it in `names.json`.

### 8.3 Calling on an object

```unk-frag
// decl: npc : Npc; it : Item;
npc = Mordea;
npc.bark("Majesty?");
it = Avatar.getEquip(EQ_BACKPACK);
Avatar.toggleStatus(OWNED);
npc.turnToFace(getDirToItem(Avatar), false, 10);
```

`x.f(...)` calls `f` on `x`: an intrinsic, a METHOD routine written for an
object (`toggleStatus`, `turnToFace`, `realDoAnim`, ...), or a usecode
process, which then runs with `x` as its `referent`.

### 8.4 Values from calls

A `routine` returns at once. A `process` called for a value -- `if
canSafeTeleport(p, map)` -- is started, the caller waits for it to end,
and the value is what the process returned. The same holds for a
`process` intrinsic (`ask`).

### 8.5 Waiting, chains and fire-and-forget

How a call to a process is **written** decides whether the caller waits.
All three forms were tested in the original game:

| Written | The caller |
|---|---|
| `f();` | waits until `f` ends |
| `spawn f() -> return;` | carries on at once; `f` runs alongside |
| `a() -> spawn b() -> return;` | carries on at once; `b` runs after `a` ends |

```unk
process DEVON::look()
{
    Avatar.bark("Hello?");
    spawn safePlaySFX(MAGIC2, 100) -> return;
    bark("Mending nets, as you can see.");
}
```

`bark` runs as a process for as long as its speech bubble shows, so the
plain call waits: Devon answers only when the Avatar's bubble is gone. The
sound is started and left playing (`safePlaySFX` itself would wait for it
to finish), and Devon speaks while it plays. Conversations rely on the
first rule: one plain `bark` after another shows the lines in turn.

The `-> return` belongs to the started process, not to the caller: the
caller does **not** return. The authors write every fire-and-forget call
this way.

### 8.6 Chains

```unk
process DEVON::look()
{
    spawn turnToFace(getDirToItem(Avatar), false, 10) ->
    spawn realDoAnim(AS_FIDGET, getDir()) -> return;
}
```

A chain runs its links one after another, as a unit the caller does not
wait for. Devon turns to the Avatar, then fidgets.

**A link waits for the one before it only once it first yields.** The
compiled chain starts the *later* link first and then ties it to the
earlier one, and a started process runs at once until its first yield. So
a later link that acts straight away acts at the same time as the first.
The game's chained functions put their work in `process { }`, which begins
by yielding -- METHOD's `realDoAnim` is

```unk-frag
// decl: animSet, dir : Number;
startup { }
process
{
    killProcess(referent, PT_ANIM);
    doAnim(animSet, dir, NO_ANIM_LIMIT, false);
    return;
}
```

and so do `turnToFace` and `wait`. Write a function you mean to chain the
same way. (Measured in the game: a second link that barked at once spoke
together with the first; moved into `process { }`, it spoke after it.)

### 8.7 Inline blocks

A chain may end in a block of code that runs, as its own process, when the
chain before it has ended:

```unk
process DEVON::look()
{
    realDoAnim(AS_FIDGET, getDir()) -> [[PT_DEFAULT, 0, 0]]
    {
        bark("There, that's better.");
    };
}
```

The block's tag (`[[type, object, flags]]`; the game's usual one is
`[[1, 0, 0]]`, the same thing written in numbers) gives it its process
type. The block runs later than the function around it, inside that
function's frame:

- use `referent` and the function's variables **by reference**, as the
  game's 20 blocks do (`npc1.setDead()`);
- reading a variable's value, a parameter, or assigning a variable inside
  the block is untested in the game; unkle builds it with a note.

---

## 9. Processes

### 9.1 What runs when

Every handler, every process called with `spawn`, every chain link and
every inline block is a process. The engine runs one at a time, each until
it yields -- by `wait`, by a call that waits, by `suspend`, or at the top
of each pass of `process { }` -- and then the next. A routine never
yields; a long loop in a routine holds up the whole game.

### 9.2 Process types

Every process has a **type** and an **object**, and the game uses the pair
to find and stop processes. The types the authors named:

| Name | Value | Used for |
|---|---|---|
| `PT_ANIM` | `0x0F0` | animations |
| `PT_BARK` | `0x200` | speech bubbles |
| `PT_FX` | `0x202` | sound and visual effects |
| `PT_PATHFIND` | `0x204` | walking somewhere |
| `PT_ACTIVITY` | `0x205` | an NPC's current activity |
| `PT_USE` | `0x207` | a `use` handler |
| `PT_LOOK` | `0x208` | a `look` handler |
| `PT_ANIM_UNK` | `0x20A` | the `anim` handler; `turnToFace` |
| `PT_DEFAULT` | -- | the function's own type (4.4) |

A handler's type is its event's (table in 4.3). A plain function registers
no type unless it has a tag (4.4); `PT_DEFAULT` there is 1.

### 9.3 Stopping processes

```unk-frag
killProcess(referent, PT_BARK);
Kernel::resetRef(referent, PT_ACTIVITY);
if Kernel::getNumProcesses(referent, PT_LOOK) > 2 { return; }
```

`killProcess(item, type)` (FREE) stops every process of that type for
that item; it is the engine's `resetRef`. The authors stop an NPC's
activity, speech, animation and walking before a conversation
(`enterConvo` does all four), and `leaveFastArea` handlers commonly stop
everything as the object goes out of range. `getNumProcesses` counts them.

### 9.4 Waiting

`wait(n)` (FREE) waits `n` 60ths of a second. Inside it is a `process { }`
loop testing a `Timer`, so it also yields at once, which makes it a good
first link in a chain.

### 9.5 Conversations

The game's conversations follow one pattern, built from METHOD's
`enterConvo` and `exitConvo` and the `ask` intrinsic:

```unk
typeclass Talker inherits Item
{
    process talk();
}

process Talker::talk()
{
    answer : String;
    choices : String List;

    enterConvo();
    bark("What do you want?");
    choices = <<"Who are you?", "Goodbye.">>;
    while answer != "Goodbye."
    {
        answer = ask(choices);
        if answer = "Who are you?"
        {
            bark("Nobody you know.");
        }
    }
    exitConvo();
}
```

`enterConvo` stops what the NPC is doing, holds the Avatar still and turns
the two to face each other; `ask(list)` shows the choices and returns the
one picked; `exitConvo` lets everyone move again.

---

## 10. Searching the world: `foreach`

```unk
typeclass Finder inherits Item
{
    process look();
}

process Finder::look()
{
    it : Item;
    n : Number;

    n = 0;
    foreach it where (type = TypeBarrel) within 10 cells of referent
    {
        n++;
    }
    bark(numToStr(n) + " barrels nearby.");
}
```

`foreach item where (condition) <search> { ... }` runs the body once for
each item the search finds, with `item` set to it. The searches:

| Search | Finds |
|---|---|
| `within N cells of X` | items within N cells of item X (a cell is 32 world units) |
| `within N units of X` | the same, N in world units |
| `in container X` | items directly inside container X |
| `on surface of X` | items lying on X |

`recursive` -- at the end, or right after `foreach` -- also searches
inside containers: `foreach recursive it where family = CONTAINER_FAMILY
within 30 cells of Avatar`.

The condition is optional, and tests only these fields of the item:

| Field | |
|---|---|
| `type` | its shape number (`TypeBarrel`, `TypePathMarker`, ...) |
| `frame` | its frame within the shape |
| `quality` | its quality |
| `family` | its family (`CONTAINER_FAMILY`, ...) |

with `=` or `<`, joined by `and` and `or`, and the set test `type in {a,
b, c}` or `frame in {...}` (the game uses up to five values for `type`
and six for `frame`). The compared
value may be any expression: `frame = (19 - counter)`. Anything else in a
`where` clause is an error; test it inside the body instead.

`break;` ends the search early.

---

## 11. One source, three languages

The game shipped in English, French and German, one archive each, all
compiled from the same sources. unkle reads and writes sources the same
way.

### 11.1 Separators

```unk-frag
bark("Why are you in my home?
    \f Pourquoi es-tu dans ma maison?
    \g Was habt Ihr in meinem Haus zu suchen?");
```

In a literal, `\f` begins the French text and `\g` the German. Each
language's build keeps only its own part. The space after the separator
belongs to the next language's text, which is why the game's French and
German strings begin with a space; the line break before a separator
becomes the space that ends the English.

A literal with no separator is the same in every language, which is how
names are written once. A literal with `\f` but no `\g` gives German the
English text.

### 11.2 `#if` blocks

Where the languages differ in more than a literal -- a number spliced into
the middle of a sentence, or different code -- the statement is written
once per language:

```unk-frag
// decl: day : Number;
#if English
{
    bark("I have been in Pagan for " + numToStr(day) + " days.");
}
#if French
{
    bark("Cela fait " + numToStr(day) + " jours que je suis à Pagan.");
}
#if German
{
    bark("Ich bin seit " + numToStr(day) + " Tagen in Pagan.");
}
```

Each language's build keeps its own block's contents in place (a block is
not a scope). There is no `#else` or `#endif`; the authors always wrote all
three blocks, repeating one where two languages agree.

### 11.3 Building

A multilingual source is compiled with the languages named:
`unkle src USECODE.FLX -l E,F,G`. A mod written this way serves every
language with one file; an English-only mod is fine too, and builds into
all three with English text.

A literal joined with `+` is joined before the language is picked, so a
long multilingual text may be continued with `+"` on the next line, even
where a separator starts the new part.

---

## 12. Writing a mod

`README.md` covers the folders and commands; this section covers what to
write in them.

### 12.1 Layers

A mod is a folder -- a **layer** -- with up to three parts:

- `new/` -- classes the game does not have, one per file;
- `override/` -- a whole existing class, replacing the decompiled file
  (same name, same kind);
- `patch/` -- just the functions that change, for any classes:
  - a function with an existing name replaces it;
  - a new name adds the function at the end of its class;
  - `remove CLASS::name;` on a line of its own removes one;
  - `rename CLASS::old new;` renames one and every call to it.

Patched functions are written exactly as in a decompiled file:
`process CLASS::name(...)` at the start of a line, its closing `}` at the
start of a line.

### 12.2 What a change may do

A patched or overridden class is compiled whole, and may add, remove,
rename and re-sign its functions and add or remove event handlers. Where a
change would break a caller in another class -- a removed function still
called, a changed parameter list with old callers -- unkle refuses the
build and names the caller. Calls to a function another class adds work
in the same build.

### 12.3 Reaching new code

The engine runs a class's handlers only for its objects, so new code runs
when existing code calls it. Give the new class a plain function, and
patch an existing handler to call it:

```unk
typeclass Visitor inherits Item
{
    process look();
    process introduce();
}

process Visitor::look()
{
    spawn introduce() -> return;
}

process Visitor::introduce()
{
    bark("A visitor from afar.");
}
```

```unk
// in the same mod
process DEVON::use()
{
    spawn VISITOR::introduce() -> return;
}
```

The called function runs with the caller's object as `referent` -- here,
Devon speaks.

**Call plain functions, not handlers.** A handler's prologue ends it at
once if a process of its type already runs for its object (4.3), and the
caller's own handler is such a process. unkle refuses a call into another
class's handler of the same type:

```unk-error
// expect: exclu
typeclass Visitor inherits Item
{
    process look();
}

process Visitor::look()
{
    bark("A visitor from afar.");
}
```

```unk-error
// in the same mod
process DEVON::look()
{
    spawn VISITOR::look() -> return;
}
```

### 12.4 Checking animations

An NPC's shape has its own set of animations, and some are drawn facing
only some directions. Played in a direction it is not drawn in, an
animation shows blank frames -- the character vanishes. Give unkle the
game's data files and it checks every animation the changed classes play
(`README.md`, "Checking animations").

### 12.5 After the build

Every build writes `OUT.FLX.identity.json` beside the archive. Keep the two
together: decompiling the archive later reads it and gives back your
function names, so a mod survives being decompiled again.

---

## 13. Lessons from the game

Each of these was learned by a mod that built and round-tripped perfectly
and then did the wrong thing in the game.

### 13.1 A handler called from a handler ends itself

The first mod tried here made Devon's `look` start a new class's `look`.
Every check passed; in the game, clicking Devon showed nothing. Both
handlers ran as (Devon, `PT_LOOK`), so the second ended in its prologue
before saying anything. The fix is the rule in 12.3: give the new class a
plain function and call that.

### 13.2 Chains run together unless the later link yields

A two-link chain whose second link barked at once printed both lines
together. Put a chained function's work in `process { }` (8.6).

### 13.3 An animation can make a character vanish

Devon's animation 27 exists, but is drawn facing one direction only.
Played facing the way he stood, it made him disappear for good. The game
itself only ever plays it on him facing that direction, late in the story.
An animation the game plays on a character is evidence for that character
**in that situation** only. Check animations against the game's files
(12.4).

### 13.4 New classes are not in the world

A new type class has a shape number nothing in the world uses, so its
handlers never run by themselves. That is what makes adding classes safe,
and it is why new code is reached by calls.

### 13.5 A new module function must have an object to act on

The first mod built with unkle added four spells to WATRSPEL. Every check
passed; in the game three of them crashed, and none showed the Avatar's
casting pose. The new functions were called without an object but laid
out as though they had one, so every parameter, and `referent`, was read
from the wrong slot. Fixed in the compiler (4.5). The rule to remember: a
module function acts on whatever object its caller passes, and the game's
callers pass the character.

### 13.6 Smaller traps

- `not` takes everything to the next `or` (6.1).
- `&` and `|` bind as tightly as `*`: bracket them inside `+` and `-`
  (6.1).
- A line break inside a literal is a space (2.5).
- Lists count from 1 (6.6).
- A routine never yields: loops that wait belong in a process.

---

## 14. How the authors wrote it

The decompiler writes code the way the authors' sources do, wherever the
compiled bytes allow, and new code reads best in the same style. Measured
on their 28 files:

- **Library calls are bare**: `wait(30)`, `canSafeTeleport(p, map)`, not
  `FREE::wait`.
- **Member calls have no type**: `pathEgg.getFrame()`.
- **Calls to METHOD routines on an object are written on it**:
  `Avatar.toggleStatus(OWNED)`, bare `toggleStatus(INVISIBLE)` for this one.
- **Numbers are decimal** (1,050 against 7 hex); a bit mask written as a
  number is hex (`& 0x00FF`).
- **Flags are named**: `getStatus() & INVISIBLE`, `orStatus(OWNED)`,
  `DMG_FIRE | DMG_BLUNT`.
- **Truth values are `true`/`false`**; `doAnim`'s last argument is written
  `FALSE`.
- **Animations are named**: `realDoAnim(AS_CAST1, getDir())`.
- **`x++` and `x--` for counters; `x += v` for numbers; `s = s + "..."`
  for strings.**
- **Few parentheses**: `if fram = 0`, `if not bool`, but `(a = 1) and (b =
  2)` with each side bracketed.
- **Declarations of one type share a line**: `counter, offX, offY :
  Number;`, at the top of the function or its `startup` block.
- **A `for` loop names its start**: `for i = 1 to 5`.
- **Every fire-and-forget call is `spawn f() -> return;`.**
- **`turnToFace`'s second argument** is `false` when turning to face
  someone and `0` for a direction (it chooses the way round: `RANDOM`,
  `LEFT` or anything else for right).
- **A long condition** breaks after `and`/`or` at about 80 columns, the
  continuation indented, `{` at the end of its last line.
- **A long literal** breaks at a space near column 80, the continuation
  indented, separators starting new lines.
- **`switch` and `if`/`else if` chains** are both used, for strings,
  numbers and named values alike.

---

## Appendix A. Grammar

```
unit        = { uses | constant | globalflag | classdecl | function }
uses        = "uses" STRING ";"
constant    = "constant" NAME "=" expr ";"
globalflag  = "globalflag" NAME ";"
classdecl   = ("npcclass" | "typeclass") NAME [ "inherits" NAME ]
              "{" { declfn } "}"
declfn      = kind [ tag ] NAME params [ ":" type ] ";"
kind        = "process" | "routine" | "unkroutine"
function    = kind [ tag ] [ NAME "::" ] NAME params [ ":" type ] block
params      = "(" [ NAME ":" type { "," NAME ":" type } ] ")"
type        = NAME [ "List" | "Reference" ] | NAME "[" NUMBER "]"
tag         = "[[" expr "," expr "]]"
block       = "{" { stmt } "}"

stmt        = decl | tag ";" | assign ";" | call-stmt
            | "if" expr block [ "else" ( block | if-stmt ) ]
            | "switch" expr "{" { "case" expr { "," expr } block }
                                [ "default" block ] "}"
            | "while" expr block
            | "for" NAME [ "=" expr ] "to" expr block
            | "foreach" NAME "in" NAME block
            | "foreach" [ "recursive" ] NAME [ "where" cond ] search
                        [ "where" ( cond | "(" ")" ) ] block
            | "startup" block | "process" block
            | "return" [ expr ] ";" | "break" ";" | "suspend" ";"
            | "exclusive" ";"
            | NAME ( "++" | "--" ) ";"
            | NAME ( "^=" | "-=" ) expr ";"
            | lang-block
decl        = NAME { "," NAME } ":" type [ "=" expr ] ";"
assign      = target ( "=" | "+=" | "-=" | "*=" | "/=" ) expr
call-stmt   = call [ chain ] ";"
chain       = "->" ( "return" | call [ chain ] | tag3 block [ chain ] )
tag3        = "[[" expr "," expr [ "," expr ] "]]"
search      = "within" expr ( "cells" | "units" ) "of" expr [ "recursive" ]
            | "in" "container" expr [ "recursive" ]
            | "on" "surface" "of" expr
lang-block  = "#if" ( "English" | "French" | "German" ) block

expr        = (see section 6.1 for precedence)
primary     = NUMBER | STRING | NAME | "(" expr ")" | "<<" [ expr { "," expr } ] ">>"
            | call | primary "." NAME | primary "[" expr "]"
            | primary ":" type
call        = [ "spawn" ] [ primary "." ] [ NAME "::" ] NAME "(" [ expr { "," expr } ] ")"
cond        = field ( "=" | "<" ) expr | field "in" "{" expr { "," expr } "}"
            | cond "and" cond | cond "or" cond | "(" cond ")"
field       = "type" | "frame" | "quality" | "family"
```

---

## Appendix B. Built-in names

unkle knows these names without any file. They are the authors' own, each
paired with its value by aligning their sources with the compiled game.
Status flags and damage types live in `.unkle/names.json` instead, and
globals and intrinsics always do.

**Values**

```
true 1 · false 0 · TRUE 1 · FALSE 0 · NULL_REF 0 · NO_ANIM_LIMIT 10000
differentMap -1 (FREE's getDistance on another map)
```

**Directions** (`getDir()`, `getDirToItem()`)

```
north 0 · northeast 1 · east 2 · southeast 3 · south 4 · southwest 5 · west 6 · northwest 7
RANDOM 0 · LEFT 1 · RIGHT 2      (turnToFace's turning side)
```

**Time** -- FREE's `getPeriod()` (four game hours each), days and months

```
bloodwatch 0 · firstebb 1 · daytide 2 · threemoons 3 · lastebb 4 · eventide 5
guarday 0 · earthday 1 · waterday 2 · airday 3 · fireday 4 · blackday 5
stonemark 0 · skyrock 1 · windbreak 2 · stormpeak 3 · firefall 4 · darkflame 5
```

**Process types**

```
PT_ANIM 0x0F0 · PT_BARK 0x200 · PT_FX 0x202 · PT_PATHFIND 0x204 · PT_ACTIVITY 0x205
PT_USE 0x207 · PT_LOOK 0x208 · PT_ANIM_UNK 0x20A · PT_DEFAULT 1
```

**Status flags and damage** (the authors' names, in `names.json`)

```
OWNED 0x04 · CONTAINED 0x08 · INVISIBLE 0x10 · VIRTUAL 0x80
CONTAINER_GUMP_OPEN 0x0100 · HANGING 0x1000
DMG_HAND 0x01 · DMG_BLUNT 0x04 · DMG_FIRE 0x08 · DMG_FULL 0x80
```

**Animations** (`realDoAnim`, `doAnim`, `getLastAnimSet`)

```
AS_WALK 0 · AS_RUN 1 · AS_STAND 2 · AS_VERT_LEAP 3 · AS_GET_UP 4
AS_DRAW_WEAPON 5 · AS_SHEATH_WEAPON 6 · AS_SWING_WEAPON 7 · AS_FALL_DOWN 14
AS_CLIMB_2 19 · AS_CLIMB_3 20 · AS_CLIMB_4 21 · AS_CLIMB_5 22
AS_ENTER_CAST 27 · AS_EXIT_CAST 28 · AS_CAST1 29 · AS_CAST2 30 · AS_CAST3 31
AS_LOOK_LEFT 32 · AS_LOOK_RIGHT 33 · AS_FIDGET 47 · AS_SIT_DOWN 51
AS_SIT_UP 52 · AS_TALK 53
```

**Item types** (shape numbers)

```
TypeKey 82 · TypeDoorNS 68 · TypeDoorEW 69 · TypeTallDoorNS 340 · TypeTallDoorEW 342
TypeBarrel 115 · TypeBottle 180 · TypeTankard 178 · TypeFood 537 · TypeMorefood 538
TypePlateWithFood 284 · TypeBarCounter 160 · TypeWoodThrone 223 · TypeCusion 812
TypeBook 613 · TypeGrimoire 288 · TypePathMarker 174 · TypeUnkEgg 73 · TypeDummy 731
TypeObsidianCoin 143 · TypeBlackrockFragments 442 · TypeStalagmytes 66
TypeArmour 523 · TypeArmGuards 530 · TypeHelmets 531 · TypeLeggings 532
TypeMagArmor 841 · TypeTorwinsRing 832 · TypeAirFocus 396 · TypeFireItem 397
TypePentagramBase 124 · TypeMagicYellowCircle 202 · TypeMagicSparkle 587
TypeFireField 400 · TypeFireGlobe 703 · TypeFireblast 480 · TypeLightning 511
TypeExplosion 578 · TypeFrictionGrenade 592 · TypeWallSwitch 505 · TypeLever 672
TypeDemon 96 · TypeBlackStag 152 · TypeWoundedTorax 209 · TypeMiniAirTitan 481
TypeSalkind 404 · TypeSorcerer 200
```

**Families, slots and others**

```
QUAL_FAMILY 1 · QUAN_FAMILY 2 · CONTAINER_FAMILY 6 · REAGENT_FAMILY 9
FIXED_TYPE 0 · LAND_SURFACE_TYPE 3 · EQ_BACKPACK 6 · Gump_MainMenu 1
ACT_LOITER 0 · ACT_COMBAT 1 · AL_GOOD 2 · AL_CHAOTIC 8
Symbol 1 · Wand 2 · Rod 3 · Staff 4 · Talisman 5        (METHOD's focusCheck)
Necromancer 0 · Theurgist 1 · Tempest 2 · Sorcerer 4 · Thaumaturge 5   (isMage)
```

**Sound effects** (`playSFX`, `safePlaySFX`)

```
LTNING1A 6 · TELEPORT 7 · AROWFLY1 9 · MIXSPELL 15 · EXPLODE3 27 · EXPLODE1 31
IMPACT1A 41 · MAGIC1 54 · MAGIC2 57 · MAGIC3 58 · MAGIC9 59 · MONSTER9 69
THUNDR1A 91 · THUNDR2A 94 · FALSCRM1 99 · WIND3 115 · WIND4 116 · SMOKPUF1 124
WHOA1A 146 · EXPLODE2 158 · SPELLGO2 189 · SPELLGO4 191 · CHEATER 217 · TROODLE 218
```

**Maps** (`getMap()`, `teleport`)

```
MapProgPlayland 1 · MapDocks 3 · MapFishermansReef 4 · MapEastRoad 5
MapShepherdValley 7 · MapThePlateau 8 · MapDaemonsCrag 12 · MapBrimstoneCliffs 13
MapObsidianFortress 14 · MapRoadTreasure 15 · MapCarthaxLake 16 · MapStoneCove 21
MapLavaRiverBefore 24 · MapLavaRiverAfter 25 · MapCarthaxII 26
MapCARTHAXbreakingground 28 · MapHallOfTheMountainKing 31 · MapArgentrockIsle 37
MapWestTenebrae 39 · MapCentralTenebrae 40 · MapEastTenebrae 41 · MapPlaneOfFire 44
MapPlaneOfWater 45 · MapPlaneOfAir 46 · MapPlaneOfEarth47 47 · MapUpperCatacombsII 49
MapUpperCatacombsIII 50 · MapLowerCatacombsI 51 · MapShrine 54 · MapLowerCatacombsII 57
MapAncientNecs 62 · MapGhostRoom 63
```

**Per-NPC schedule values**, valid where the authors used them -- compared
with or assigned to the global named:

```
araminaSchedule   at_home 1 · chow_down 3
mordeaSchedule    in_bed 1 · on_throne 2 · chow_down 3
cyrrusSchedule    at_home 1 · at_monastery 2 · in_woods 3
darionSchedule    at_home 1 · east_side_patrol 3 · west_side_patrol 4
SalkindSchedule   at_home 1 · at_court 2
acolyteKilled     baneDead 1 · vardionDead 2
```

---

## Appendix C. The library: FREE and METHOD

The functions below are the game's shared library, called bare from any
class (8.1). Names and parameters are the authors' where their sources
survive; descriptions are from the authors' code. The kind matters: a
`process` call waits (8.5).

**FREE**

| Function | Kind | |
|---|---|---|
| `random(x)` | routine | a random number from 0 to x - 1 (`urandom`) |
| `randRange(a, b)` | routine | a random number from a to b inclusive |
| `abs(x)` | routine | the absolute value |
| `wait(time)` | process | wait `time` 60ths of a second |
| `askYesNo()` | process | the Avatar picks Yes (true) or No (false) |
| `getPeriod()` | routine | the time of day, 0-5 (`bloodwatch` ... `eventide`) |
| `getDistance(a, b)` | routine | the distance between two items; `differentMap` on different maps |
| `give(r, type, frame, quality, quantity) : Boolean` | routine | put new items into container r (for the Avatar, the backpack); false if r is not a container |
| `take(r, type, frame, quality, quantity) : Boolean` | routine | take such items out of r |
| `createNear(point, type, frame, quality, quantity)` | routine | create an item near a point |
| `safePlaySFX(sound, priority)` | process | play a sound and wait for it to end |
| `killProcess(r, ptype)` | routine | stop r's processes of a type |
| `scrollTo(x, y, z, vel)`, `scrollBack(vel)` | process | move the camera away and back |
| `unkCloseAllGumps()` | routine | close the open container windows near the Avatar; true if it closed any |

**METHOD** -- called on an object (`npc.f(...)`, or bare for this one)

| Function | Kind | |
|---|---|---|
| `turnToFace(newDir, side, speed)` | process | turn step by step to face `newDir` (`side`: 0 / `false` the short way, `LEFT`, else right) |
| `realDoAnim(animSet, dir)` | process | play an animation (stops the current one) |
| `enterConvo()`, `exitConvo()`, `enterConvo2()` | process | begin and end a conversation (9.5) |
| `ask(choices, tries)` | process | keep asking until an answer is picked or `tries` attempts pass (the one-argument `ask` is the intrinsic) |
| `toggleStatus(flag)` | routine | turn a status flag on if off, off if on |
| `receiveHit(sender, dir, damage)` | routine | take a hit with `DMG_HAND` damage |
| `timedBark(time, text)` | process | show a speech bubble for `time` 60ths of a second |
| `lookBark(text)` | process | the bark a `look` handler shows, unless several are already showing |
| `moveItemTo(x, y, z, vel)`, `legal_moveItemTo(...)` | process | slide an item to a position |
| `safeTeleport(point, map) : Boolean`, `canSafeTeleport(...)` | process | teleport an NPC where the Avatar will not see it happen (or test whether it can) |
| `pace(frame1, frame2)` | process | walk between two path markers |
| `wander()` | process | wander about |
| `downUp(dir)` | process | fall down, get up and stand, facing `dir` |
| `isInWorld()` | process | not hidden away (z not above 251) |
| `isMage()` | routine | which school of magic the NPC follows, or `false` |
| `focusCheck(item)` | routine | which kind of focus an item is (`Wand`, `Rod`, ...), or `false` |
| `boom(selfDestruct)`, `candleAnim()` | process | an explosion; a candle's flicker |

**Others** -- `CALGUARD::callTheGuards(r)` summons the town guards;
`DOOR_NS::doorDriver(...)` opens and closes a door; `AIRSPEL`'s functions are the
air spells (`healingTouch`, `airWalk`, `hearTruth`, ...).

---

## Appendix D. What the decompiler writes that you need not

Decompiled source is UNK, but it must reproduce the original bytes exactly,
and some of what it writes exists only for that:

| Written | Why | In new code |
|---|---|---|
| `num1`, `str1`, `item1`, `arg1` | the bytes keep no variable names | name them |
| `sub_02A8` | a private function's name is not in the bytes; it is its offset | name it |
| `_pad1 : Byte[2]` | an unused variable the authors declared | omit |
| `else { }` | an empty `else` still compiles to a jump the original has | omit |
| `5:Word`, `0x00FF` | the original used a word where the value fits a byte | write the number |
| `[[1:Word, referent]]` before a name | an object tag's word-wide type | `PT_DEFAULT` |
| `within N units of` | a search whose range the original gave in world units | `cells` |
| `where ()` | the order of a surface search's operands | omit |
| `[[t, o, 5]]` on an inline block | the third byte of its header, which varies | `[[1, 0, 0]]` |
| a module function written `CLASS::name` | the class it lives in | bare name |
| `twice(referent, 4)` | an object passed to a class routine, shown explicitly | `twice(4)` |
| a top-level copy of an inline block | the block, for reference only; not compiled | -- |

**Layout annotations** may appear in very old decompiled text and should
not be written: `@block XXXX`, `@latch`, `@after`, labels and `goto`,
`detached`, `?[...]`. The shipped game's 1,598 functions need none of them.

---

## Appendix E. Error messages

| Message | Meaning |
|---|---|
| `SyntaxError: unexpected 'X' at line L, col C` | the text cannot be read there; for a multilingual source, the line is the master file's |
| `unknown identifier X` | not a variable, global, constant or character name |
| `cannot assign to Name(id='X')` | an assignment to a name that is not a variable or global |
| `unknown intrinsic X` | no intrinsic, library or class function by that name with that many arguments |
| `unknown call target CLASS::X` | no such function in that class |
| `X is a method of this unit but did not resolve` | the class name in a definition does not match the declaration |
| `calls X, which no class defines` | a call to a function nothing in the build has |
| `calls X, which this class no longer defines` | a removed or renamed function is still called |
| `... begins with process.exclude on the same (object, type N) ...` | a call into another class's handler of the same process type (12.3) |
| `character 'X' (U+....) has no cp437 byte` | a character the game cannot display (2.1) |

A failed build writes no archive.
