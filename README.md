# unkle -- the UNK Language Engine

*Version 1.0.1*

`unkle` compiles and decompiles the scripts of **Ultima VIII: Pagan** -- the
usecode archives `EUSECODE.FLX` (English), `FUSECODE.FLX` (French) and
`GUSECODE.FLX` (German). Usecode is written in UNK, the language Origin's
developers used; `unkle` decompiles an archive into UNK source that compiles
back to the **same bytes**, and compiles your edited source, or a mod built
on top of it, into an archive the original game runs.

It is for developers writing mods and custom code for Ultima 8. The
language is described in **`UNK_MANUAL.md`**, beside this file, with a
first mod in its first section; the compiled form is in
`USECODE_VM_REFERENCE.md`.

- one file, no installation, no dependencies: Python 3.10 or later
- every archive tested decompiles and compiles back byte-for-byte: the
  English, French and German archives of version 2.12, and the 1994
  English and German archives (the 1994 French one has not been tested)
- one source folder can hold all three languages and build each of them
- your changes live in **custom layers** next to the decompiled source, so
  the source stays as decompiled and one mod builds for every language

## Contents

1. [Getting started](#getting-started)
2. [The name files](#the-name-files)
3. [Decompiling](#decompiling)
4. [Compiling](#compiling)
5. [The source folder](#the-source-folder)
6. [Custom layers](#custom-layers)
7. [Multilanguage sources](#multilanguage-sources)
8. [Checking animations against the game's data](#checking-animations-against-the-games-data)
9. [Reference: all options](#reference-all-options)
10. [About the project](#about-the-project)
11. [Licence](#licence)

## Getting started

```
python3 unkle -h
```

On Linux and macOS `./unkle` works as well. Then, with the name files
described next:

```
# 1. decompile the game's English archive into ./src
python3 unkle -d EUSECODE.FLX -o ./src --intrinsics convert_usecode_u8.h \
    --globals u8e2.12_used.txt,u8e2.12_unused.txt --flags item.h,weapon_info.h

# 2. compile it back -- before any change, the result is identical
python3 unkle ./src USECODE.FLX -l E --verify EUSECODE.FLX

# 3. put your changes in ./src/custom (see Custom layers), build, and copy
#    EUSECODE.FLX into the game's USECODE folder
python3 unkle ./src USECODE.FLX -l E
```

**Keep a copy of the original archives.** `unkle` never writes over its
input, but the game's files are what you will compare against.

## The name files

The archives carry no names: an intrinsic is a number, a global is a bit
offset. `unkle` takes the names from three files that are **not included**
with it -- they belong to the ScummVM and Pentagram projects and are under
the GPL, while `unkle` is under the Apache License 2.0. Download them
yourself; `unkle` reads them, it does not copy them into itself.

| Option | Files | Where to get them | Licence |
|---|---|---|---|
| `--intrinsics` | `convert_usecode_u8.h` | ScummVM: `engines/ultima/ultima8/convert/u8/convert_usecode_u8.h` (github.com/scummvm/scummvm) | GPL-3.0+ |
| `--globals` | `u8e2.12_used.txt`, `u8e2.12_unused.txt` | Pentagram: `docs/scratch/globals/` (github.com/pentagram-u8/pentagram) | GPL-2.0 |
| `--flags` | `item.h`, `weapon_info.h` | ScummVM: `engines/ultima/ultima8/world/` | GPL-3.0+ |

- `--intrinsics` is **required to decompile**: it names the intrinsics, gives
  their parameter lists, and names the 32 events.
- `--globals` is optional. Without it, global flags get placeholder names
  such as `global_0123h_1bit`.
- `--flags` is optional. Without it, status and damage flags are named only
  where the game's authors named them (`INVISIBLE`, `DMG_FIRE`); with it,
  ScummVM's names fill the rest (`FLG_FLIPPED`). `--flags none` uses the
  authors' names only.

Decompiling stores the tables it read in the source folder
(`.unkle/names.json`, with each file's checksum), so **compiling needs none
of these files**. Passing them to the compiler replaces the stored tables;
an intrinsic whose parameters changed needs `--allow-signature-change`, a
table of another size (a modified executable) `--allow-table-size`.

If you publish a source folder, note that `.unkle/names.json` holds names
taken from those GPL files.

## Decompiling

```
unkle -d ARCHIVE.FLX -o OUTPUT_DIR --intrinsics FILE [options]
```

Writes the source folder `OUTPUT_DIR`, created if it does not exist. A
folder that is not empty is refused unless you pass `--force`, which
replaces the `npcs/`, `types/`, `modules/` and `.unkle/` it holds and leaves
anything else (your `custom/` layer, say) alone. Source files are UTF-8; `--encoding cp437` writes the game's own
DOS encoding instead. Either is read back automatically.

With `-ml`, the other languages of the same build are decompiled too and
merged with the first into one source (see
[Multilanguage sources](#multilanguage-sources)):

```
unkle -d ./input/EUSECODE.FLX -ml ./input/FUSECODE.FLX,./input/GUSECODE.FLX \
    -o ./new-src/ --intrinsics convert_usecode_u8.h
```

An archive's language is the first letter of its file name. For other
names, write `L=PATH`: `-ml F=french.flx,G=german.flx`. English must be one
of the archives. The archives must come from the same release of the game:
2.12 with 2.12, 1994 with 1994. A mixed set is refused, naming the classes
that differ.

A decompile takes one and a half to two minutes per language; with `-ml`
the languages are decompiled side by side.

**Decompiling an archive unkle built.** Every compile writes
`OUTPUT.FLX.identity.json` beside the archive: which function is which,
under the names your source gave them. `unkle -d` reads it when it sits
beside the archive, and the folder it writes compiles back to that archive
exactly, names included. Keep the two files together.

## Compiling

```
unkle SOURCE_DIR OUTPUT.FLX [-l LANGS] [options]
```

The language letter goes in front of the file name. Name the output
`USECODE.FLX` and you get the names the game expects:

```
unkle ./src USECODE.FLX -l E          -> EUSECODE.FLX
unkle ./src out/USECODE.FLX -l E,F,G  -> out/EUSECODE.FLX, out/FUSECODE.FLX,
                                         out/GUSECODE.FLX
unkle ./src MYUSECODE.FLX -l G        -> GMYUSECODE.FLX
```

`-l` is required for a multilanguage source. A single-language source
builds its own language, and `-l` may be left out.

**Everything it needs is in the source folder** (`.unkle/`, written when
it was decompiled): no original archive, no name files. `--base FLX` builds
against an original archive instead of that stored data, one per language.

`--verify FLX` compares the result with an archive class by class and exits
with status 1 on any difference. Before you change anything, verifying
against the archive you decompiled must pass.

What a compile reports:

- classes whose layout changed are listed (`RELAID`), with the size of
  each changed function
- a class that fails to compile stops the build: no archive is written,
  and the error names the class and the line of its file:
  `FAIL  DEVON: SyntaxError: unexpected ';' at line 19, col 23`
- calls to a function another class removed or renamed are errors, named
- notes on animations, when the game's data files are given

## The source folder

```
src/
  npcs/          one .unk per NPC class (npcclass)
  types/         one .unk per item class (typeclass)
  modules/       one .unk per module: classes with no events, holding
                 routines other classes call (FREE, METHOD, the spells)
  custom/        your custom layer, if you keep it here (optional)
  .unkle/        what unkle needs to compile without the original archive:
    names.json     the name tables (see The name files)
    E/             per language:
      archive.json   the archive's slot layout
      base-facts.json  facts about the original classes -- entry points,
                       calls, signatures, checksums; no code
      raw/           the archive's slots that are not classes, byte-for-byte
      identity.json  present when the archive was one unkle built
    F/ G/          the same, for a multilanguage source
```

Every `.unk` file starts with

```
// Auto-decompiled by unkle
// class 0x0596 (AIRSPEL) -- 11 processes
```

What the decompiled source cannot give back: the bytes carry no local
variable names (`num1`, `str1`), no names for private functions (`sub_02A8`,
named by offset), no unused locals (they show as `_pad1 : Byte[2]`), no
comments, and no `uses` lines (which unkle accepts and ignores in files
you write). A class's declared name is its archive name,
and its base class is a guess (`Npc` for NPC classes, else `Item`). None of
this affects the bytes. A file with no class declaration is a module.

Edit the `.unk` files directly if you like, but a custom layer keeps your
changes apart from the decompiled text and builds for every language.

## Custom layers

A layer is a folder with up to three subfolders. `SOURCE_DIR/custom` is
used when it exists; `--custom DIR` adds layers (repeatable; later layers
win).

```
mymod/
  new/        *.unk  classes the game does not have
  override/   *.unk  a WHOLE existing class: same name, same kind
  patch/      *.unk  only the functions that change, any classes per file
```

In `patch/`:

- a function with an existing name **replaces** it; a new name is **added**
- `remove CLASS::name;` removes a function
- `rename CLASS::old new;` renames it everywhere it is called

A new class gets a free class number and slot. The game only runs a
class's event handlers for objects of its shape, so reach new code from an
existing class: a patched function that calls it.

`--show-overlay DIR` writes each class a layer changed, as it is compiled.

Things `unkle` refuses, with the reason: a class that fails to compile, a
call into another class's event handler of the same process type (the
engine would end it before it runs), and a call to a function that no
longer exists.

## Multilanguage sources

`unkle -d ... -ml ...` writes one source whose text carries every language:

```
bark("Why are you in my home?
    \f Pourquoi es-tu dans ma maison?
    \g Was habt Ihr in meinem Haus zu suchen?");
```

`\f` starts the French text and `\g` the German, as in the game's original
sources. Where two languages differ in more than text, the statement is
written once per language:

```
#if English
{
    bark(arg1 + ", the late Sorcerer");
}
#if French
{
    ...
```

A literal may run across lines: a line break inside a literal compiles to
**one space**, so a break can only stand where the text has a space. Where
a separator is glued to text on both sides, the literal is ended and the
next one starts with the separator, joined with `+`, which compiles to the
same single string:

```
    eat them too!"
    +"\fCe que me disent les poissons ...
```

A source merged from two languages (`-ml` with one other archive) builds
those two. Its `#if` blocks still name all three languages, and the third
language's block repeats the English one; `-l` with that language is
refused, as the source has no build data for it.

`unkle ./src USECODE.FLX -l E,F,G` builds all three from it. A custom
layer can be written the same way, so one patch serves every language.

## Checking animations against the game's data

A function that plays an animation an NPC's shape does not have -- or has
drawn blank in the direction it faces -- makes the NPC vanish in the game.
Give the game's data files and `unkle` notes it for the classes you
changed:

```
unkle ./src USECODE.FLX -l E --anim-dat STATIC/ANIM.DAT \
    --itemcach STATIC/ITEMCACH.DAT --shapes STATIC/U8SHAPES.FLX
```

Notes only, never errors; findings the original game already has are not
shown. The files are read as given, so a mod's own data files can be checked
too.

## Reference: all options

```
unkle [options] SOURCE_DIR OUTPUT.FLX           compile
unkle -d [options] ARCHIVE.FLX -o OUTPUT_DIR    decompile
```

| Option | Mode | |
|---|---|---|
| `-h`, `--help` | both | show the help and exit |
| `--version` | both | show the version |
| `-q`, `--quiet` | both | errors only, no progress |
| `-l`, `--language LANGS` | compile | `E`, `F`, `G`, or a list: `E,F,G` |
| `--custom DIR` | compile | a custom layer; repeatable, later wins |
| `--base [L=]FLX` | compile | build against an original archive; repeatable, one per language |
| `--verify [L=]FLX` | compile | compare with an archive; exit 1 on difference; repeatable |
| `--no-identity` | compile | do not write `OUTPUT.FLX.identity.json` |
| `--show-overlay DIR` | compile | write the classes a layer changed |
| `--ignore-unknown` | compile | build despite `.unk` files the manifest does not list |
| `--anim-dat FILE` | compile | check animations against `ANIM.DAT` |
| `--itemcach FILE` | compile | actors' starting shapes, with `--anim-dat` |
| `--shapes FILE` | compile | the frames, with `--anim-dat` |
| `-d`, `--decompile` | decompile | decompile instead of compile |
| `-o`, `--output DIR` | decompile | the source folder to write (required) |
| `-ml`, `--multi-language FLX[,FLX]` | decompile | the other languages, merged into one source |
| `--encoding utf-8\|cp437` | decompile | encoding of the files written (default `utf-8`) |
| `--force` | decompile | write into a folder that is not empty |
| `--intrinsics FILE` | both | `convert_usecode_u8.h`; required to decompile |
| `--globals USED,UNUSED` | both | the two `u8e2.12` files |
| `--flags FILE[,FILE]` | both | `item.h,weapon_info.h`, or `none` |
| `--allow-signature-change` | both | accept changed intrinsic parameters |
| `--allow-table-size` | both | accept a different number of intrinsics |

Archives in `-ml`, `--base` and `--verify` take their language from the
file name's first letter, or from `L=` (`G=german.flx`).

`unkle` reads no environment variables. Exit status: 0 success, 1 the
build or a check failed, 2 incorrect usage (the help is printed).

## About the project

### History

In January 2025, Dominus Dragon released a remarkable file from The Origin
Museum's archives: a snapshot of the work-in-progress code for *The Lost
Vale*, the expansion that was never released. It answered many questions
the community had long had about the expansion (see the findings on the
Ultima Codex:
https://ultimacodex.com/2025/02/a-look-at-the-usecode-for-ultima-8-the-lost-vale/).

The package also held something else of great value: several examples of
UNK, the language the game's scripts were written in, not only for *The
Lost Vale* but for the base game as well.

A great deal of effort had already gone into Pentagram and ScummVM to
reverse-engineer the original game and build a compatible virtual machine.
That raised a question: could these pieces of original source be used to
build a decompiler, turning the game's compiled usecode back into UNK?
Since some of the code was for the base game, the compiled form could in
principle be mapped back to the source. That seemed possible, if very
difficult and time-consuming.

To test the idea, I made a small proof of concept. ARAMINA.UNK was one of
the few original source files we had, and I obtained its disassembly with
Pentagram's `disasm` tool. Could those two artifacts be used to turn the
disassembly back into UNK?

They could. But anyone familiar with these artifacts knows how hard that
is: the idea was sound, but the work needed another breakthrough. So I
shelved the project for a while.

A few months later, AI agents and tools arrived and changed how this kind
of work could be approached. Could they be used here? That question is how
this project began.

### What this project is

- **A decompiler and a compiler.** `unkle` decompiles the retail game's
  usecode archives (`EUSECODE.FLX`, `FUSECODE.FLX`, `GUSECODE.FLX`) back
  into UNK source code, and compiles UNK source into archives the game runs.
- **A common baseline for modders and other projects.** UNK code written
  for one mod should work in another, as long as both use the same (or
  compatible) intrinsics and globals.
- **A historical study.** It documents and recreates the UNK language with
  preservation in mind, aiming to be as authentic as possible. Byte-for-byte
  compatibility with the retail release was the guiding principle.

### What this project is not

- Affiliated in any way with Origin Systems, Electronic Arts or any other
  company.
- A source of anyone's copyrighted code or data. It contains none of the
  game's code or data files. The original sample sources were used for
  analysis only and are not redistributed here.
- A full decompilation of the game. For that, see Kitrinx's decomp:
  https://github.com/Kitrinx/Ultima8_Decomp

### Known issues

- **ScummVM does not recognise modified archives.** ScummVM identifies the
  game by checking its files' MD5 sums, so a modified usecode archive stops
  the game from being recognised. To run a mod in ScummVM, you will likely
  need to add your modified file to its detection tables
  (`detection_tables.h`). The original DOS game runs modified archives as
  they are.

### Acknowledgements

- Origin Systems and the original Ultima VIII development team, for
  creating the game.
- Mad Hatter, for extensive work documenting the game's file formats.
- The Pentagram and ScummVM developers, for their work describing the
  intrinsics, globals and flags that this project depends on.
- Dominus Dragon, for releasing the files that started this project.
- theGreyWanderer, for help with some pieces and the Titan Toolkit
  (https://github.com/theGreyWanderer-uc/tgwUltima).
- Kitrinx, for help with decompilation and valuable insights into the
  usecode's structure.
- Claude and Anthropic, for the legwork that made this possible.
- UDIC and its Discord community, for their patience and support.

## Licence

`unkle` is licensed under the Apache License, Version 2.0.

The name files are not part of `unkle` and keep their own licences
(ScummVM: GPL-3.0 or later; Pentagram: GPL-2.0). Ultima VIII and its data
files are the property of their owners; `unkle` contains none of the
game's code or data files, and the archives it reads are your own copies.
