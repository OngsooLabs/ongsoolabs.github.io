# OSL Deterministic Core

A zero-dependency, zero-allocation deterministic RNG suite for Unity.

OSL Deterministic Core gives gameplay programmers a reliable foundation for any system
that requires reproducible randomness: lockstep multiplayer, replay recording, server-authoritative
simulations, automated QA, save-state rollback, and procedural content generation.

---

## Table of Contents

1. [Overview](#overview)
2. [Requirements](#requirements)
3. [Installation](#installation)
4. [Quick Start](#quick-start)
5. [Core API](#core-api)
6. [Shuffling](#shuffling)
7. [Gacha / Weighted Draws](#gacha--weighted-draws)
8. [Pity System](#pity-system)
9. [Save & Restore State](#save--restore-state)
10. [Full Documentation](#full-documentation)

---

## Overview

**Key highlights:**

- **Deterministic by construction** — same seed → same byte-for-byte sequence on every platform, every Unity version, every CPU.
- **Zero allocations on hot paths** — RNG advance, dice, gacha draw, and shuffle never allocate. State save/load uses `Span<byte>`.
- **Zero third-party dependencies** — Pure .NET Standard 2.1. No native plugins.
- **Integer-first probability** — `RollPermille`, `RollProbability`, and weighted gacha use integer comparisons to avoid floating-point drift across platforms.
- **Pluggable** — `IRandomEngine` and `IPitySystem` are full extension points.
- **Production tested** — Ships with EditMode tests covering determinism, distribution, state round-trips, and pity semantics.

---

## Requirements

- Unity 2021.3 LTS or newer
- Scripting runtime: .NET Standard 2.1

---

## Installation

**Option A — Package Manager (Git URL)**

1. Open **Window > Package Manager**.
2. Click **+** > **Add package from git URL…**
3. Enter the repository URL pointing to `Assets/OSL/Deterministic`.

**Option B — Asset Store / .unitypackage**

1. Import the asset into your project.
2. Verify that the `Assets/OSL/Deterministic/` folder is present.
3. *(Optional)* Import the **Lockstep Demo** sample from the Package Manager Samples section.

---

## Quick Start

```csharp
using OSL.Deterministic.Runtime.Core;

// Seed-only: identical seed produces identical sequences.
var rng = new OSLRandom(seed: 0xC0FFEE);

// Seed + stream id: derive independent sub-RNGs from one master seed.
var combatRng = new OSLRandom(new PCG32(seed: 0xC0FFEE, stream: 1));
var lootRng   = new OSLRandom(new PCG32(seed: 0xC0FFEE, stream: 2));
```

> **Never** call `UnityEngine.Random` or `System.Random` inside deterministic code.
> Their results vary between platforms and Unity versions.

---

## Core API

```csharp
uint  u  = rng.NextUInt();           // [0, 2^32)
uint  u2 = rng.NextUInt(100u);       // [0, 100)  unbiased
int   i  = rng.NextInt(10, 20);      // [10, 20)
float f  = rng.NextFloat();          // [0, 1)
bool  b  = rng.NextBool();
bool  c  = rng.Chance(0.15f);        // 15% true
int   r  = rng.Range(1, 7);          // d6
```

---

## Shuffling

```csharp
using OSL.Deterministic.Runtime.Shuffling;

int[] deck = new int[52];
for (int i = 0; i < deck.Length; i++) deck[i] = i;

OSLDeckShuffler.Shuffle(deck, rng);            // in-place, zero alloc
OSLDeckShuffler.Shuffle(deck, rng, 0, 10);     // shuffle only the first 10 elements
```

---

## Gacha / Weighted Draws

```csharp
using OSL.Deterministic.Runtime.Gacha;

var table = new OSLGachaTable<GachaEntry<string>>(new []
{
    new GachaEntry<string>(60, "common",   "C"),
    new GachaEntry<string>(30, "uncommon", "U"),
    new GachaEntry<string>(10, "rare",     "R"),
});

var result = table.Draw(rng);   // returns one entry, weighted by first parameter
```

---

## Pity System

The pity system guarantees a rare drop within a configurable number of consecutive non-rare draws.

```csharp
using OSL.Deterministic.Runtime.Pity;

var pity = new SimpleHardPity(threshold: 90);
// Pass the pity tracker alongside your gacha table for guaranteed distribution.
```

---

## Save & Restore State

```csharp
Span<byte> buf = stackalloc byte[rng.StateSize];  // 16 bytes for PCG32
rng.SaveState(buf);

// Later — same process, same machine, or a different machine entirely.
rng.LoadState(buf);
```

Use this to:

- Implement rollback netcode (snapshot RNG alongside simulation state).
- Persist mid-run state across save files.
- Reproduce a bug from a player's session by serializing the engine state.

---

## Full Documentation

For the complete API reference and the Lockstep Demo walkthrough, see the generated pages under
[OSL Integration / Assets / OSL / Deterministic](/OSL%20Integration/Assets/OSL/Deterministic/README.html).
