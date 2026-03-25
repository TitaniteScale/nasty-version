# How the Autorun Toggle Works — An Explainer

If you're new to C or coming back to coding after a break, this doc walks through *exactly* why the three small changes we made were enough to implement autorun. No assumed knowledge — we'll build up from scratch.

---

## Background: What Even Is a Flag?

In Pokémon Emerald's codebase, a **flag** is just a single bit stored in the save data. Think of it like a light switch: it's either **on** (1) or **off** (0). The game uses hundreds of these to track things like "has the player received the Running Shoes?" or "has this NPC been talked to?"

```c
FlagGet(FLAG_SYS_B_DASH)   // returns TRUE if the flag is on, FALSE if off
FlagSet(FLAG_SYS_B_DASH)   // turns the flag on
FlagClear(FLAG_SYS_B_DASH) // turns the flag off
FlagToggle(FLAG_SYS_B_DASH) // flips it — on becomes off, off becomes on
```

They live in `include/constants/flags.h` and are just `#define`s — named aliases for numbers. The name `FLAG_SYS_B_DASH` is easier to read than a raw number, but under the hood it's all just numbers pointing to a bit in memory.

---

## Background: `newKeys` vs `heldKeys`

Every single frame (the GBA runs at 60 frames per second), the game reads the physical buttons. It gives you two different views of that data:

- **`heldKeys`** — a bitmask of every button that is currently held down. If you're holding B right now, `heldKeys & B_BUTTON` is true. It's true every frame for as long as you hold it.
- **`newKeys`** — a bitmask of only the buttons that were *just* pressed **this frame** — i.e. they weren't held last frame but are held now. If you're holding B and have been for a while, `newKeys & B_BUTTON` is false. It only fires on the very first frame of the press.

This distinction is crucial. The original running shoes used `heldKeys` because you needed to *hold* B. Our toggle needs `newKeys` because we want to respond to a *tap* — one press, not a sustained hold.

A "bitmask" just means each button corresponds to a specific bit in a 16-bit number. The `&` operator checks if a particular bit is set:

```c
// B_BUTTON is a constant like 0x0002 (binary: 0000000000000010)
// If bit 1 is set in heldKeys, the & returns nonzero (truthy)
if (heldKeys & B_BUTTON) { /* B is held */ }
```

---

## The Call Chain: How a Button Press Reaches the Run Logic

When you press a direction on the d-pad, the game goes through a chain of function calls. Understanding this chain is the key to understanding where we made our changes and why.

```
CB1_Overworld()
  └─ DoCB1_Overworld(newKeys, heldKeys)
       └─ PlayerStep(direction, newKeys, heldKeys)
            └─ MovePlayerAvatarUsingKeypadInput(direction, newKeys, heldKeys)
                 └─ MovePlayerNotOnBike(direction, heldKeys)   ← newKeys dropped here!
                      └─ PlayerNotOnBikeMoving(direction, heldKeys)
                           └─ [run or walk decision happens here]
```

Notice that `newKeys` is passed all the way down to `MovePlayerAvatarUsingKeypadInput`, but then `MovePlayerNotOnBike` only accepts `heldKeys`. The original code never needed `newKeys` down that deep because all it cared about was whether B was *currently held* — so it was fine to drop it.

We needed `newKeys` to detect a tap, so we placed our toggle logic at `MovePlayerAvatarUsingKeypadInput`, which is the last point in the chain where `newKeys` is still available.

---

## Change 1: Defining the Flag

**File:** `include/constants/flags.h`

**Before:**
```c
#define FLAG_UNUSED_0x8E3   (SYSTEM_FLAGS + 0x83) // Unused Flag
```

**After:**
```c
#define FLAG_SYS_AUTORUN    (SYSTEM_FLAGS + 0x83) // Autorun toggle (B-button tap)
```

This is the simplest change. We didn't create any new memory or new logic here — we just gave a name to a bit that already existed but was unused. A `#define` in C is a preprocessor directive: before your code even compiles, the compiler does a find-and-replace of every occurrence of `FLAG_SYS_AUTORUN` with `(SYSTEM_FLAGS + 0x83)`.

We chose an unused slot so we don't accidentally overwrite something the game is already using. The `SYSTEM_FLAGS` base address plus the offset `0x83` points to a specific bit in the save file.

---

## Change 2: The Toggle Logic

**File:** `src/field_player_avatar.c` — inside `MovePlayerAvatarUsingKeypadInput`

**Before:**
```c
static void MovePlayerAvatarUsingKeypadInput(enum Direction direction, u16 newKeys, u16 heldKeys)
{
    if (gPlayerAvatar.flags & (PLAYER_AVATAR_FLAG_MACH_BIKE | PLAYER_AVATAR_FLAG_ACRO_BIKE))
        MovePlayerOnBike(direction, newKeys, heldKeys);
    else
        MovePlayerNotOnBike(direction, heldKeys);
}
```

**After:**
```c
static void MovePlayerAvatarUsingKeypadInput(enum Direction direction, u16 newKeys, u16 heldKeys)
{
    if (gPlayerAvatar.flags & (PLAYER_AVATAR_FLAG_MACH_BIKE | PLAYER_AVATAR_FLAG_ACRO_BIKE))
        MovePlayerOnBike(direction, newKeys, heldKeys);
    else
    {
        if ((newKeys & B_BUTTON) && FlagGet(FLAG_SYS_B_DASH))
            FlagToggle(FLAG_SYS_AUTORUN);
        MovePlayerNotOnBike(direction, heldKeys);
    }
}
```

Let's break down the new `if` line piece by piece:

```c
if ((newKeys & B_BUTTON) && FlagGet(FLAG_SYS_B_DASH))
    FlagToggle(FLAG_SYS_AUTORUN);
```

- **`newKeys & B_BUTTON`** — "Was B just tapped this frame?" (not held, just newly pressed)
- **`&&`** — both conditions must be true (logical AND)
- **`FlagGet(FLAG_SYS_B_DASH)`** — "Does the player have the Running Shoes?" This is the flag Mom sets when she gives them to you in Littleroot Town. This gate means B-tapping does nothing new until you've earned the shoes.
- **`FlagToggle(FLAG_SYS_AUTORUN)`** — flips our new flag. On → off, off → on.

Notice this whole block only runs in the `else` branch — which is when the player is **not on a bike**. If you're on the Mach Bike or Acro Bike, it goes into `MovePlayerOnBike` instead and this code never runs. That's the existing structure working in our favor; we didn't have to add any extra bike checks ourselves.

The toggle runs *before* `MovePlayerNotOnBike(direction, heldKeys)` is called. This means the flag is always up to date before the movement logic checks it.

---

## Change 3: Using the Flag to Decide Whether to Run

**File:** `src/field_player_avatar.c` — inside `PlayerNotOnBikeMoving`

**Before:**
```c
if (!(gPlayerAvatar.flags & PLAYER_AVATAR_FLAG_UNDERWATER)
 && (heldKeys & B_BUTTON)
 && FlagGet(FLAG_SYS_B_DASH)
 && IsRunningDisallowed(gObjectEvents[gPlayerAvatar.objectEventId].currentMetatileBehavior) == 0
 && !FollowerNPCComingThroughDoor()
 && (I_ORAS_DOWSING_FLAG == 0 || (I_ORAS_DOWSING_FLAG != 0 && !FlagGet(I_ORAS_DOWSING_FLAG))))
```

**After:**
```c
if (!(gPlayerAvatar.flags & PLAYER_AVATAR_FLAG_UNDERWATER)
 && FlagGet(FLAG_SYS_AUTORUN)
 && FlagGet(FLAG_SYS_B_DASH)
 && IsRunningDisallowed(gObjectEvents[gPlayerAvatar.objectEventId].currentMetatileBehavior) == 0
 && !FollowerNPCComingThroughDoor()
 && (I_ORAS_DOWSING_FLAG == 0 || (I_ORAS_DOWSING_FLAG != 0 && !FlagGet(I_ORAS_DOWSING_FLAG))))
```

The only line that changed is the second one: `(heldKeys & B_BUTTON)` became `FlagGet(FLAG_SYS_AUTORUN)`.

Here's why this is the right place. This big `if` statement is the gatekeeper for running. It only runs `PlayerRun()` if **all** of these are true:

| Condition | What it checks |
|---|---|
| `!(flags & PLAYER_AVATAR_FLAG_UNDERWATER)` | Player is not diving underwater |
| `FlagGet(FLAG_SYS_AUTORUN)` | *(our change)* Autorun is toggled on |
| `FlagGet(FLAG_SYS_B_DASH)` | Player has the Running Shoes |
| `IsRunningDisallowed(...) == 0` | This tile allows running (e.g. not indoors) |
| `!FollowerNPCComingThroughDoor()` | No follower NPC is mid-door-animation |
| ORAS dowsing check | Dowsing isn't active (expansion feature) |

By swapping out `heldKeys & B_BUTTON` for `FlagGet(FLAG_SYS_AUTORUN)`, we changed the question from *"is B physically held right now?"* to *"is the autorun flag switched on?"*. All the other safety checks — indoors, underwater, follower NPCs — remain completely untouched. They continue to work for free.

This is why you automatically slow down in buildings even with autorun on: `IsRunningDisallowed()` returns true indoors, which makes the whole condition false, so the game falls through to the walk logic instead. The flag stays set; the tile just overrides it temporarily.

---

## Putting It All Together

Here's the full flow with our changes in place:

1. **Player taps B** on the overworld (not in a menu, not on a bike).
2. `DoCB1_Overworld` reads `gMain.newKeys` and `gMain.heldKeys` from hardware and passes them down.
3. `MovePlayerAvatarUsingKeypadInput` sees `newKeys & B_BUTTON` is true and `FLAG_SYS_B_DASH` is set → calls `FlagToggle(FLAG_SYS_AUTORUN)`. The flag is now on.
4. Every subsequent frame, the player walks a step → `PlayerNotOnBikeMoving` checks `FlagGet(FLAG_SYS_AUTORUN)` → it's on → player runs (assuming the tile allows it).
5. **Player taps B again** → same toggle fires → flag is now off → player walks.
6. **Player enters a building** → `IsRunningDisallowed()` returns true → player slows to a walk even though `FLAG_SYS_AUTORUN` is still on. Steps out of the building → runs again automatically.

Three changes. One new flag. One toggle on a tap. One condition swap. That's all it took.

---

## Why This Approach Is Safe

A concern you might have: *"what if B is pressed in a menu or during a battle — will it accidentally toggle autorun?"*

It won't, for a couple of reasons:

- **Menus and battles have their own input loops.** `DoCB1_Overworld` (and therefore the entire call chain above) only runs when `CB2_Overworld` is the active callback — meaning the overworld is in control. Menus, battles, and cutscenes swap out the callback, so this code simply doesn't execute.
- **`ProcessPlayerFieldInput` gets first dibs.** In `DoCB1_Overworld`, `ProcessPlayerFieldInput` runs before `PlayerStep`. If B triggers something like a dive/emerge action, it returns `TRUE` and `PlayerStep` is never called — so the toggle never fires.

The game's existing architecture naturally sandboxes our change to exactly the right context.