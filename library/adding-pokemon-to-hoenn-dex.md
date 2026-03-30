# Adding a Pokémon Line to the Hoenn Pokédex

If you've added a new Pokémon to the game (encounters, trainers, etc.) but it doesn't show up in the Pokédex, this guide explains exactly what to fix and why. We'll use the Froakie line as the running example.

---

## Background: How the Regional Dex Works

The Hoenn Pokédex is a *regional* dex — a curated subset of the full national dex. The game maintains a separate numbered list of just the Pokémon that belong to the Hoenn region. When you see or catch a Pokémon, the game checks whether it belongs to this regional list to decide whether to register it.

This regional list is maintained in two parallel data structures that must always stay perfectly in sync:

1. **`HoennDexOrder` enum** — defines what *slots* exist in the regional dex, and in what order. Each entry is just a named number. The game uses these numbers as indices.
2. **`sHoennToNationalOrder` array** — maps each of those regional slot numbers to the corresponding national dex number, so the game knows which species data to load.

If a Pokémon is missing from either one, the dex won't work correctly for it. If the two get out of sync (different number of entries, or entries in a different order), the whole dex will silently show the wrong Pokémon for wrong numbers.

---

## The Two Files You Need to Edit

### File 1: `include/constants/pokedex.h`

This file contains the `HoennDexOrder` enum. Every Pokémon in the regional dex needs an entry here. The entries are just named constants — `HOENN_DEX_FROAKIE`, `HOENN_DEX_FROGADIER`, etc. — whose numeric values are automatically assigned in sequence by the C compiler.

This is also where `HOENN_DEX_COUNT` gets its value:

```c
#define HOENN_DEX_COUNT (HOENN_DEX_DEOXYS + 1)
```

`HOENN_DEX_DEOXYS` is an enum value, not a hardcoded number. Its actual numeric value is determined by how many entries come before it in the enum. This means you must add new entries **before** `HOENN_DEX_DEOXYS` so that Deoxys's value shifts upward and `HOENN_DEX_COUNT` includes your new Pokémon. Entries added after Deoxys would be outside the count and the game would ignore them.

### File 2: `src/pokemon.c`

This file contains the `sHoennToNationalOrder` array. Its size is `HOENN_DEX_COUNT - 1`, so it must contain exactly as many entries as the enum has (excluding the `HOENN_DEX_NONE` zero value).

Each entry uses the `HOENN_TO_NATIONAL` macro, which expands a species name into the correct `NATIONAL_DEX_` constant. The entries must be in the **exact same order** as the enum in `pokedex.h`.

---

## Step-by-Step Instructions

### Step 1 — Find the family flag name

All Pokémon in this codebase are grouped into families and gated behind a `P_FAMILY_` flag. You need to know your Pokémon's flag name before writing any code.

Search for it in `include/config/species_enabled.h`:

```
grep P_FAMILY_FROAKIE include/config/species_enabled.h
```

You'll find something like:

```c
#define P_FAMILY_FROAKIE    P_GEN_6_POKEMON
```

The flag name is `P_FAMILY_FROAKIE`. Make a note of it.

### Step 2 — Find all the species names in the line

Check `include/constants/pokedex.h` to confirm the exact national dex names for every Pokémon in the evolution line:

```
grep NATIONAL_DEX_FROAKIE include/constants/pokedex.h
```

For Froakie the line is: `FROAKIE` → `FROGADIER` → `GRENINJA`.

Note: size variants and alternate forms (like Pumpkaboo's sizes) are handled as *forms* of a single species and only need one dex entry each — only add entries for distinct Pokédex numbers.

### Step 3 — Add to the `HoennDexOrder` enum

Open `include/constants/pokedex.h` and find the end of the `HoennDexOrder` enum. You're looking for this section near the bottom:

```c
    HOENN_DEX_JIRACHI,
    HOENN_DEX_DEOXYS,
};
```

Add your new entries **between** `HOENN_DEX_JIRACHI` and `HOENN_DEX_DEOXYS`, wrapped in your family flag:

```c
    HOENN_DEX_JIRACHI,
#if P_FAMILY_FROAKIE
    HOENN_DEX_FROAKIE,
    HOENN_DEX_FROGADIER,
    HOENN_DEX_GRENINJA,
#endif //P_FAMILY_FROAKIE
    HOENN_DEX_DEOXYS,
};
```

The `#if` / `#endif` guard means the entries are only compiled in when the family flag is enabled — matching how every other Pokémon in the codebase is handled.

### Step 4 — Add to the `sHoennToNationalOrder` array

Open `src/pokemon.c` and find the same region at the end of `sHoennToNationalOrder`:

```c
    HOENN_TO_NATIONAL(JIRACHI),
    HOENN_TO_NATIONAL(DEOXYS),
};
```

Add your entries in the **identical position and order** as you did in Step 3:

```c
    HOENN_TO_NATIONAL(JIRACHI),
#if P_FAMILY_FROAKIE
    HOENN_TO_NATIONAL(FROAKIE),
    HOENN_TO_NATIONAL(FROGADIER),
    HOENN_TO_NATIONAL(GRENINJA),
#endif //P_FAMILY_FROAKIE
    HOENN_TO_NATIONAL(DEOXYS),
};
```

The guard, the order, and the species names must match Step 3 exactly.

---

## The Golden Rule

> **The enum in `pokedex.h` and the array in `pokemon.c` must always have the same entries in the same order.**

The array is indexed by the enum values. If they differ — even by one entry, or one entry in the wrong position — the game will map regional dex numbers to the wrong Pokémon silently, with no compiler error to warn you.

---

## Quick Reference: Adding Multiple Lines at Once

If you're adding several lines in one session, stack them between `HOENN_DEX_JIRACHI` / `HOENN_TO_NATIONAL(JIRACHI)` and `HOENN_DEX_DEOXYS` / `HOENN_TO_NATIONAL(DEOXYS)`:

**`include/constants/pokedex.h`:**
```c
    HOENN_DEX_JIRACHI,
#if P_FAMILY_FROAKIE
    HOENN_DEX_FROAKIE,
    HOENN_DEX_FROGADIER,
    HOENN_DEX_GRENINJA,
#endif //P_FAMILY_FROAKIE
#if P_FAMILY_SQUIRTLE
    HOENN_DEX_SQUIRTLE,
    HOENN_DEX_WARTORTLE,
    HOENN_DEX_BLASTOISE,
#endif //P_FAMILY_SQUIRTLE
    HOENN_DEX_DEOXYS,
};
```

**`src/pokemon.c`:**
```c
    HOENN_TO_NATIONAL(JIRACHI),
#if P_FAMILY_FROAKIE
    HOENN_TO_NATIONAL(FROAKIE),
    HOENN_TO_NATIONAL(FROGADIER),
    HOENN_TO_NATIONAL(GRENINJA),
#endif //P_FAMILY_FROAKIE
#if P_FAMILY_SQUIRTLE
    HOENN_TO_NATIONAL(SQUIRTLE),
    HOENN_TO_NATIONAL(WARTORTLE),
    HOENN_TO_NATIONAL(BLASTOISE),
#endif //P_FAMILY_SQUIRTLE
    HOENN_TO_NATIONAL(DEOXYS),
};
```

The order of the blocks relative to each other doesn't matter, as long as it's the same in both files.

---

## Why Doesn't Just Adding the Pokémon to the Game Work?

When the game registers a Pokédex sighting or catch, it calls `NationalToHoennOrder()` to convert the Pokémon's national dex number into a Hoenn dex number. If the Pokémon isn't in `sHoennToNationalOrder`, that function loops through the entire array without finding a match and returns `0` — which is `HOENN_DEX_NONE`. A regional dex number of zero means "not in the regional dex", so nothing gets registered and the Pokémon never appears in the dex UI, even though it exists in the game perfectly fine.

That's the entire reason for these two edits: you're giving the game a regional dex number to associate with the species.

## Questions

- How do regional forms work? Are they just new pokemon?

## Pokemon to add to Pokedex
- Galarian Zigzagoon
