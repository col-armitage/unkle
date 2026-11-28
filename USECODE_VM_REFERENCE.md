# Ultima VIII Usecode: VM and Bytecode Reference

*For unkle 1.0. The compiled form of UNK: the archive, the class layout, the
instruction set, and how each source construct compiles. The language
itself is `UNK_MANUAL.md`.*

Every statement here was measured against the shipped archives -- English
V2.12 and 1994, French and German V2.12, German 1994 -- which unkle
decompiles and rebuilds byte for byte, and against the 28 surviving
original sources. Where something is not known, this document says so.

## Contents

1. [The archives](#1-the-archives)
2. [The Flex container](#2-the-flex-container)
3. [Class layout](#3-class-layout)
4. [Execution model](#4-execution-model)
5. [Instruction set](#5-instruction-set)
6. [Calling conventions](#6-calling-conventions)
7. [How source constructs compile](#7-how-source-constructs-compile)
8. [The search-loop predicate language](#8-the-search-loop-predicate-language)
9. [Names the bytes do not carry](#9-names-the-bytes-do-not-carry)
10. [Open questions](#10-open-questions)

---

## 1. The archives

One archive per language and release:

| Archive | Release | Bytes | Classes |
|---|---|---|---|
| `EUSECODE.FLX` | English, 1994 | 1,209,018 | 516 |
| `EUSECODE.FLX` | English, patch 2.12 | 1,251,108 | 516 |
| `FUSECODE.FLX` | French, 2.12-era | 1,300,957 | 516 |
| `GUSECODE.FLX` | German, 2.12-era | 1,378,604 | 516 |
| `GUSECODE.FLX` | German, 1994 | 1,335,445 | 516 |

The languages of one release share their code exactly -- every class has
the same functions, events and inline blocks -- and differ in their
strings and in the 11 functions whose code the translations changed. The
two releases differ in code: 10 classes have different functions (FREE
grew from 7,871 to 14,532 bytes), 107 event functions changed, and the
global flag table shifted (section 9.3).

Each archive carries one language's text, in the DOS code page 437. No
archive contains a language separator or a directive: the original
toolchain built each language from shared multilingual sources.

---

## 2. The Flex container

`*USECODE.FLX` is a Flex archive, the game's general container of numbered
objects.

### 2.1 Header and directory

```
offset   0 - 79    comment, padded with 0x1A (empty in the usecode archives)
offset  80 - 83    uint32  (not decoded; 6682 in V2.12)
offset  84 - 87    uint32  entry count (1433)
offset  88 - 91    uint32  1
offset  92 - 127   reserved
offset 128 - ...   directory: count x (uint32 offset, uint32 length)
```

All values little-endian. An entry of length 0 is empty.

### 2.2 Entries 0 and 1

**Entry 0** holds 8,475 bytes in V2.12 whose meaning is not decoded; unkle
copies it unchanged.

**Entry 1** is the class-name table: a 4-byte header, then one 13-byte
slot per class id, the name in ASCII, zero-padded. Shipped names are at
most 8 characters; one slot (class `0x0237`) reads `MORDEABE`, not cleanly
terminated.

### 2.3 Class entries

```
class_id = entry index - 2
```

So entry 2 is class 0. Of 1,431 class slots, 516 are used. The class id
carries the class's kind (section 3.6).

---

## 3. Class layout

### 3.1 Structure

```
bytes   0 -  11    header: three uint32
bytes  12 - 139    event table: 32 x uint32, offsets of the handlers
bytes 140 - ...    code
```

**Offsets** inside a class -- event table entries, call and spawn targets,
jump arithmetic -- count from byte 12, so the event table occupies
`0x0000`-`0x007F` and code begins at `0x0080`. A class's first function is
almost always at `0x0080`.

### 3.2 The header

| Field | Meaning | Measured |
|---|---|---|
| `dword[1]` | the class's size in bytes | 516 of 516 |
| `dword[2]` | `size + 84 x function_count + 2` | 482 of 516 |
| `dword[0]` | `dword[2] + 2 + 80 x k`, with `k` always an integer | `k` unexplained |

`k` matched no count tried (opcodes, spawns, process intrinsics, frame
sizes). The engine does not read it; unkle records it per class and keeps
it. FLAGS, a declaration-only class of 148 bytes, carries header values
that would imply 483 functions.

### 3.3 The event table

32 slots, the same for every class. Slot *n* holds the offset of the
handler for event *n*, or 0. The slots the authors' sources implement:

| Slot | Event | Process type |
|---|---|---|
| `0x00` | `look()` | `0x208` |
| `0x01` | `use()` | `0x207` |
| `0x02` | `anim()` | `0x20A` |
| `0x04` | `cachein()` | `0x213` |
| `0x05` | `hit(uword, word)` | `0x20B` |
| `0x06` | `gotHit(uword, word)` | `0x20C` |
| `0x07` | `hatch()` | `0x210` |
| `0x08` | `schedule(ulong)` | `0x215` |
| `0x09` | `release()` | `0x20D` |
| `0x0C` | `combine()` | `0x214` |
| `0x0F` | `enterFastArea()` | `0x21B` |
| `0x10` | `leaveFastArea()` | `0x21C` |
| `0x11` | `cast(uword)` | `0x21D` |
| `0x12` | `justMoved()` | `0x22D` |
| `0x13` | `AvatarStoleSomething(uword)` | `0x22E` |
| `0x15` | `guardianBark(word)` | `0x200` |

The process type is the one each event's handlers register in their
prologue (section 4.3): one value per event across the whole archive,
except a single `cast` outlier in 35. The full list of 32 names is
ScummVM's and travels in the source folder's `names.json`.

`schedule` receives a 4-byte parameter its event-table name omits: callers
push the game time as a dword (`calli` with 8 argument bytes: 4 for `this`,
4 for the time). No handler reads it.

### 3.4 Functions not in the table

Most code is reached only by calls: walking the event tables reaches 923
functions covering 45% of the bytes; following every `call`, `spawn` and
`spawn.inline` target closes over **1,598 functions in V2.12** (1,555 in the
1994 release) and the whole of the code. A function's name is in the bytes
only if it is an event handler; the rest are named by the decompiler by
offset (`sub_02A8`) unless their names were recovered (section 9).

Functions are laid out contiguously from `0x0080` in source order. The 20
inline blocks per archive sit inside their parent functions.

### 3.5 Class identity

The class id is the only identity a class has: there is no id inside the
class. The name table maps ids to names. A new class needs a free id in
the right block, a directory slot and a name-table entry, nothing else.

### 3.6 Class kind is the id block

The bytecode records no class kind and no base class. The engine maps NPC
number *n* to class `0x0400 + n`, and runs a usecode egg's events on class
`0x047F +` its egg type (the egg's quality). Both land in `0x0400`-`0x04FF`:
the game's NPC classes are `0x0401`-`0x043D` and its egg classes
`0x0480`-`0x04C9`, so unkle reads `0x047F` and above as an egg class
(`eggclass`) and the rest of the block as an NPC class. Egg type 47 and 48
eggs have no class: GLOBSWAP (`0x04B0`) finds them as templates, which
pins the offset at `0x047F`. A class with no event handlers at all is a
module; anything else is a type class, run for objects whose shape is its
id. Checked against every class whose source declares its kind: all agree.
V2.12: 46 NPC classes, 60 egg classes, 368 type classes, 42 modules. An
egg class receives only `hatch` (48 of the 60 implement it, and no other
class does), `enterFastArea` and `leaveFastArea`.

---

## 4. Execution model

### 4.1 Processes

All usecode runs in processes, scheduled cooperatively: a process runs
until it executes `suspend` (`0x53`) or ends, then the next runs. A
process starts when the engine dispatches an event, or when usecode
executes `spawn` (`0x57`), `spawn.inline` (`0x58`) or calls a process
intrinsic. A newly spawned process **runs immediately**, until its first
yield, before control returns to the spawner -- the reading every test in
the game has agreed with (7.9).

`implies` (`0x54`) makes one process wait for another to end. `ret` ends a
process.

### 4.2 Stack and frame

The stack holds 16-bit words; dwords occupy two. A function's frame:

```
BP+0A ...   parameters (methods and processes)
BP+06       `this`: a 4-byte pointer to the object's item number
BP+00..05   the call's linkage
BP-01 ...   locals, allocated downward in declaration order
```

A free routine -- one with no class in its name -- has no `this`; its
parameters start at `BP+06`. Parameters advance by their own width (a
`WorldPoint` takes 5 bytes, a dword 4) and are at least 2 bytes each.
Locals pack by size: `Boolean`, `Byte`, `Alti`, `Dir` take 1 byte, words
2, dwords 4, `WorldPoint` 5. `init N` (`0x5A`) zeroes the locals.

The bare name `referent` compiles to `push.local.dword [BP+06]; push
indirect 02h`: the object's item number, read through `this`.

### 4.3 The prologue

An event handler begins:

```
push word  <type>          the event's process type (3.3)
push       referent        the object
set.info                   0x77: register (object, type)
process.exclude            0x78: end if another process has them
```

`process.exclude` ends the new process at once if another process with the
same object and type is running. A plain function has no prologue unless
its source gives an object tag, which compiles to the first three
instructions; an explicit `exclusive;` adds the fourth.

### 4.4 Heap values

Strings and lists live on heaps; locals and the stack hold 2-byte handles.
Pushing a string or list local pushes a copy (`0x41`-`0x43`); the compiled
code frees heap locals when a function ends (`0x62`-`0x64`) and temporaries
after a call (`0x65`-`0x67`).

### 4.5 Globals

Globals are one bit-addressed table shared by all classes, saved with the
game. `push.global` / `pop.global` carry a bit offset and a width. V2.12's
504 named globals fill bits `0x000`-`0x250`.

---

## 5. Instruction set

Opcodes are one byte, followed by fixed operands, little-endian. `bp` and
`sp` operands are signed bytes. A jump's target is relative to the address
after its operand. Decoding the whole of every archive with this table
gives no unknown opcode and no decode error.

### 5.1 Stack and locals

| Op | Mnemonic | Operands | Effect |
|---|---|---|---|
| `00` | pop byte | bp | pop a word, store its low byte at `[BP+bp]` |
| `01` | pop word | bp | |
| `02` | pop dword | bp | |
| `03` | pop huge | bp, size | pop `size` bytes (a `WorldPoint`: 5) |
| `08` | pop result | -- | pop a dword into the process result |
| `09` | assign element | bp, elem size, is-slist | store into a list element |
| `0A` | push byte | b | sign-extended to a word |
| `0B` | push word | w | |
| `0C` | push dword | d | |
| `0D` | push string | len, bytes, 0 | a string literal |
| `0E` | create list | elem size, count | from `count` stacked values |
| `12` | pop temp | -- | pop a word into the return register |
| `13` | pop temp32 | -- | dword form (not seen executed) |
| `3E` | push local byte | bp | unsigned |
| `3F` | push local word | bp | |
| `40` | push local dword | bp | |
| `41` | push local string | bp | a copy |
| `42` | push local list | bp, elem size | a copy |
| `43` | push local slist | bp | a copy, strings copied too |
| `44` | push element | elem size, is-slist | element at a stacked index of a stacked list (index from 1) |
| `45` | push huge | bp, size | push `size` bytes |
| `4B` | push address | bp | 4-byte pointer to `[BP+bp]` |
| `4C` | push indirect | size | dereference a stacked pointer |
| `4D` | pop indirect | size | store through a stacked pointer |
| `4E` | push global | bit offset (word), width | |
| `4F` | pop global | bit offset (word), width | |
| `59` | push pid | -- | the current process id |
| `5D` | push retval byte | -- | |
| `5E` | push retval word | -- | |
| `5F` | push retval dword | -- | |
| `60` | word to dword | -- | sign-extend |
| `61` | dword to word | -- | truncate |
| `6D` | push result | -- | the process result register (a dword) |
| `6E` | add sp | delta | discard argument bytes |
| `6F` | push address sp | sp | 4-byte pointer into the stack |

### 5.2 Arithmetic, comparison, logic

16-bit forms at even opcodes, 32-bit forms one above.

| Op | | Op | |
|---|---|---|---|
| `14`/`15` | add | `1C`/`1D` | subtract |
| `16` | concatenate strings | `1E`/`1F` | multiply |
| `17` | list append | `20`/`21` | divide |
| `19` | string-list union | `22`/`23` | modulo |
| `1A` | string-list subtract | `1B` | list subtract (not used by U8) |
| `24`/`25` | equal | `26` | string equal |
| `28`/`29` | less | `2A`/`2B` | less or equal |
| `2C`/`2D` | greater | `2E`/`2F` | greater or equal |
| `30`/`31` | not | `32`/`33` | and |
| `34`/`35` | or | `36`/`37` | not equal |
| `38` | in list (elem size, is-slist) | `39` | bit and |
| `3A` | bit or | `3B` | bit not |
| `3C` | shift left | `3D` | shift right |

### 5.3 Control

| Op | Mnemonic | Operands | Effect |
|---|---|---|---|
| `0F` | calli | arg bytes, intrinsic (word) | call an intrinsic; result in the return register |
| `11` | call | class (word), offset (word) | call usecode in the same process |
| `50` | ret | -- | return; ends a process at its outermost frame |
| `51` | jne | rel (word) | pop; jump if false |
| `52` | jmp | rel (word) | |
| `53` | suspend | -- | yield |
| `54` | implies | `01 01` | pop two pids: the second waits for the first |
| `57` | spawn | arg bytes, this size, class, offset | start a process; pid in the return register |
| `58` | spawn.inline | class, offset, delta, this size, flags | start the block at `offset + delta` |
| `5A` | init | size | zero `size` bytes of locals |
| `5B`/`5C` | debug line | line [, name] | absent from the shipped archives |
| `62`-`64` | free string / slist / list | bp | release a local |
| `65`-`67` | free string / list / slist | sp | release a temporary |
| `69` | string to pointer | bp | a local string as `char*` |
| `6B` | string to pointer | -- | a stacked string as `char*` |
| `6C` | parameter copy | bp, kind (1 string, 2 slist, 3 list) | give a spawned process its own copy |
| `70` | loop init | bp, script size, search type | start a world search (section 8) |
| `73` | loop next | -- | push 1 and set the loop variable, or push 0 |
| `74` | loop script | byte | one byte of the search predicate |
| `75` | foreach list | bp, elem size, rel | next element of a list, or jump out |
| `76` | foreach slist | bp, elem size, rel | the same for a string list |
| `77` | set info | -- | pop (object, type): register the process |
| `78` | process exclude | -- | end if another process has the same pair |
| `79`/`7A` | end | -- | after a function's last `ret`; never executed |

`jne`, `jmp`, `foreach` offsets wrap modulo 65536: ORLOK's 40,141-byte
`use` jumps forward 35,295 bytes.

---

## 6. Calling conventions

### 6.1 Intrinsics: `calli`

Arguments are pushed **right to left**, so the first argument is on top.
A method intrinsic (`Item::...`, `Npc::...`) then gets `this`, a 4-byte
pointer to the object's item number:

```
push byte 01h          item 1, the Avatar
push byte 06h          EQ_BACKPACK
push.addr.sp -2        pointer to the item number pushed first
calli 06h 00B2h        6 argument bytes: 4 + 2
add sp -08h            the 6, plus the item number
```

`Avatar.getEquip(EQ_BACKPACK)`. The receiver's forms:

| Receiver | Pushed |
|---|---|
| this object | `push.local.dword [BP+06]` (the `this` pointer itself) |
| a local | `push.addr.bp <local>` |
| a value (an NPC number, a call result) | the value first, then `push.addr.sp` to it |

A `char*` parameter takes a 4-byte pointer: a string local is converted
with `0x69` and not freed; any other string value is converted with `0x6B`
and freed with `free.string.sp` after the call. A `word*`, `uword*` or
`WorldPoint*` parameter takes the address of the variable passed.

The result is read with `push.retval.byte/word/dword`, the width fixed per
intrinsic. A result the source discards is usually not pushed; a few
intrinsics push and drop it anyway (`push.retval.word; pop.temp`) -- 104 of
106 intrinsics behave the same at every site, `bark` and `urandom` do
both, and 6 sites in all are irreducible.

### 6.2 Usecode routines: `call`

`call class offset` runs the target in the same process. A routine written
with a class (`routine Item::toggleStatus(...)`) receives `this` after the
arguments, like a method intrinsic. The caller frees the heap arguments the
callee uses -- read from how the callee touches its parameter slots -- and
one further case that is not identified (section 10).

### 6.3 Usecode processes: `spawn`

```
push.pid                     only when the caller waits
<arguments, right to left>
<this>
spawn  argbytes thissize class offset
add sp -<argbytes>
push.retval.word             the new pid
implies 01 01                only when the caller waits
pop.temp
suspend                      only when the caller waits
```

**Three encodings**, chosen by how the call is written, all confirmed in
the game:

| Source | Bytes | The caller |
|---|---|---|
| `f();` | push pid ... implies, suspend | waits for `f` |
| `spawn f() -> return;` | no push pid, no implies | carries on |
| `a() -> spawn b() -> ...` | `b` spawned first, then `a`, one implies per link | carries on; `b` waits for `a` |

A value-returning process read in an expression is awaited and then read
with `push.result` (`0x6D`), narrowed with `0x61`.

The `-> return` of the fire-and-forget form emits nothing in the caller.

A heap argument handed to a spawned process is copied for it with
`param.copy` (`0x6C`).

### 6.4 Arguments to direct calls

`push.local.dword [BP+06]` -- the `this` pointer, two instruction bytes,
four stack bytes -- passes the current object to a routine that reads its
`this`. A routine that never reads `BP+06` still receives one.

### 6.5 Free offsets

A `free.*.sp` operand counts the bytes above the value freed: `4` for a
receiver if one was pushed, plus the widths of the arguments before it.
`Plaque::read(16, "...")` frees at `sp=6`.

---

## 7. How source constructs compile

### 7.1 Literals

A literal's width is in its spelling, not its value:

```
0x0001   0b 01 00   push word          255   0b ff 00   push word
0x01     0a 01      push byte          0xff  0a ff      push byte
1        0a 01      push byte          127   0a 7f      push byte
-30      0a e2      push byte          128   0b 80 00   push word
```

A bare decimal 0-127 or -1 to -128 is a byte (sign-extended by the VM);
anything else a word. Hex: 1-2 digits a byte, 3-4 a word, 8 a dword.
`N:Word`, `N:Byte` and `N:Long` force the width. A named constant
compiles as a bare decimal of its value.

### 7.2 `if` and `else`

```
<cond> ; jne else_l
<then> ; jmp end_l           the merge jump: emitted whenever there is an else
else_l: <else>
end_l:
```

The merge jump is emitted **even when unreachable**: after a `then` that
ends in `return`, and over an empty `else { }` (a jump of 0). So `if A {
...; return; } else { ... }` and `if A { ...; return; } ...` are the same
program and different bytes, and the archive states which the author
wrote: V2.12 has 1,460 returning `if`s with no merge jump and 733 with
one. One exception is confirmed against original source: a returning arm
at the tail of an enclosing construct gets no merge jump (XAVIER's
`use`). An empty `else` (417 sites) has no other spelling.

At the end of a function, a merge jump is `jmp` to the epilogue, the same
bytes as `return;`: two consecutive jumps there are a `return;` plus a
merge.

### 7.3 Loops

`while c { b }`: `top: c; jne end; b; jmp top; end:`.

`for v = a to n { b }`: `v = a;` then the `while (v <= n)` form with
`v = 1 + v` at the bottom -- the constant first.

`break` jumps to the loop's end. A list `foreach` is `0x75`/`0x76` at the
top, its exit offset in the operand.

### 7.4 `switch`

No multi-way instruction: a `switch` compiles exactly as an `if` / `else
if` chain comparing the subject with each case. The two cannot be told
apart in the bytes.

### 7.5 Increments and compound assignment

`x++` compiles as `x = 1 + x` (the constant pushed first); `x--` as `x = x
- 1`; `x += v` as `x = x + v`. So a target-first increment in the bytes is
`x += 1` or `x = x + 1` in source.

### 7.6 Strings and lists

`"a" + "b"` between two literals is folded at compile time. `s + t`
concatenates (`0x16`). `list ^= <<x>>` is `push literal; create list;
push list; slist.union; free old; store`; `list -= <<x>>` pushes the list
first (`0x1A`). `list + <<x>>` appends (`0x17`).

### 7.7 Object tags

A tag statement `[[T, O]];` compiles to `push T; push O; set.info`, `T`
pushed by the literal rule (`PT_DEFAULT` = 1, a byte). A tag before a
function's name is the same with `T` always a word. An event handler's own
prologue (4.3) takes precedence.

### 7.8 `startup` and `process`

`startup { }` compiles inline; its declarations join the function's.
`process { }` is an endless loop that yields first:

```
top:  suspend
      <body>
      jmp top
```

175 backward jumps in V2.12 land on a `suspend`; 168 of them end their
function. `suspend;` inside the body compiles to a jump back to `top` (the
yield is there already), elsewhere to `0x53`.

### 7.9 Calls and chains

A chain `a() -> spawn b() -> spawn c() -> return;` spawns `c`, then `b`,
then `a`, each but the first followed by an `implies` tying it to the one
spawned before, with one `pop.temp` closing the chain. A process runs as
soon as it is spawned, until it yields, so a link that acts before
yielding acts at once; the game's chained functions begin with `startup {
} process { ...` and yield first (confirmed in the game).

### 7.10 Inline blocks

`expr -> [[T, O, F]] { body };` lays `body` out inside the parent function
and starts it with `spawn.inline class offset delta thissize F`, where
`offset` is the parent's start and `delta` the block's distance from it.
The block runs in the parent's frame, with `init 00`. 20 blocks per
archive, each reached by exactly one `spawn.inline` in its parent and by
nothing else. `F` is 0, 2 or 5 in the shipped game, and nothing measured
predicts which.

### 7.11 Text

A string is pushed with `0x0D` as CP437 bytes. A source literal's line
break and surrounding white space compile to one space; `\&` to nothing;
`\f` and `\g` divide the languages, each build keeping its own part.

---

## 8. The search-loop predicate language

`foreach item where (...) within N cells of X` compiles to a predicate
program built byte by byte with `loop.script` (`0x74`), its operands pushed
as ordinary words between the bytes, then `loop.init` (`0x70`) with the
script's size, then a loop on `loop.next` (`0x73`).

**Grammar** -- prefix notation:

```
script  := '$' [ expr ]
expr    := '&' expr expr                 and
         | '+' expr expr                 or
         | cmp field operand '%'         comparison
         | operand... LETTER             set membership
cmp     := '=' | '<'
field   := '@' type | '`' frame | '*' quality | ':' family
```

`$` alone matches everything. For `and`/`or` the right-hand term comes
first: `(type = A) and (frame = B)` compiles to `$ & = ` B % = @ A %`. A set
test pushes its N operands and then a letter: uppercase `A`-`E` test
`type` against 1-5 values, lowercase test `frame`. The script's sixteen
bytes are all known.

**Search types** (`loop.init`'s third operand):

| Type | Search | `loop.init` pops |
|---|---|---|
| 2 | area | the centre item and the range |
| 3 | area, recursive | |
| 4 | container | the container and `0xFFFF` |
| 5 | container, recursive | |
| 6 | surface | the item and `0xFFFF`, in either order |

An area range in cells compiles as `cells * 0x20`; 12 of 344 area searches
push the range directly (written `within N units`). A surface search
pushes its two operands in either order -- 24 of V2.12's 28 cursor first --
with nothing in the bytes to predict it; the decompiler records the order
by where it writes the `where` clause.

---

## 9. Names the bytes do not carry

### 9.1 What survives

The bytes carry class names (entry 1), event handlers' slots, intrinsic
numbers, global bit offsets, and every literal. They do not carry local
variable names, private function names, comments, `uses` imports, class
base names, or any distinction between spellings that compile alike
(`switch` and `if` chains; `#if` blocks and separators; `x += 1` and `x =
x + 1`).

### 9.2 Name tables

unkle names what the bytes number from four tables in the source folder's
`names.json` and two built in:

| Table | From | Keyed by |
|---|---|---|
| intrinsics | ScummVM's `convert_usecode_u8.h`, supplied by the user | intrinsic number |
| events | the same file | event slot |
| globals | Pentagram's `u8e2.12` lists, supplied by the user | bit offset |
| flags | the authors' names, plus ScummVM's `item.h`/`weapon_info.h` | value |
| constants | built in: the authors' names, each value measured | value |
| library functions | built in: recovered from the authors' sources | entry-point index |

Library function names (FREE's `wait`, METHOD's `turnToFace`, 91 in all)
are keyed by the function's position among its class's entry points, not
by offset, so they survive edits that move code; an edited archive also
carries an identity map with every function's name.

### 9.3 Globals across releases

The Pentagram lists describe V2.12. The 1994 release's globals sit at
offsets shifted by a step function:

```
[0x0000, 0x0001) +0    [0x000C, 0x00DE) +2
[0x0001, 0x000C) +1    [0x00DE, ...)    +4
```

derived from 197 globals in byte-identical functions of both releases,
with no conflict. unkle applies it when an archive's globals resolve
better that way.

---

## 10. Open questions

- **The header's `k`** (3.2): what it counts.
- **Entry 0** (2.2): what its 8,475 bytes are.
- **`implies`**: modelled as "the second pid waits for the first"; the
  game agrees in every case tested, but the exact semantics are not
  pinned down in any reimplementation either.
- **A third free rule** (6.2): 20 direct calls free an argument twice,
  more than either the callee's use or the argument's type explains
  (PENT+215E).
- **Discarded results** (6.1): six sites keep a result their intrinsic
  otherwise drops, inside functions that drop it elsewhere.
- **An inline block's flags byte** (7.10).
- **The original compiler's rule for joining a `+` chain that carries
  language separators**: no authored chain does; unkle joins before
  choosing the language.
