This is a more involved alternative to the [Running shoes](https://github.com/pret/pokecrystal/wiki/Running-Shoes) tutorial with added player animations! Note that you'll need to create sprites for this to be meaningfully different than the other option. I'll provide some guidance as to how to do that at the end.

I recommend reading this until the end before starting to decide if this is worth your effort. :)

## Contents
1. [Adding placeholder sprite assets][sec-1]
2. [Loading the sprites into ROM][sec-2]
3. [Contextualizing the assets and adding player run state][sec-3]
4. [Creating logic for run and walk state transitions with sprite transitions][sec-4]
5. [Triggering and un-triggering player state transitions][sec-5]
6. [Tips for creating running sprites][sec-6]

## 1. Adding placeholder sprite assets
To start, we'll want to create a second set of sprites for both Chris and Kris for their run animations. Character sprites can be found under *gfx/sprites/*. It's better to pick someting that's obviously different than the "normal" sprites so let's go ahead and copy the "bike" sprites over:

```bash
cp gfx/sprites/chris_bike.png gfx/sprites/chris_run.png
cp gfx/sprites/kris_bike.png gfx/sprites/kris_run.png
```

If you've compiled pokecrystal before, you might notice `.2bpp` files along with each `.png`. [2BPP](https://www.huderlem.com/demos/gameboy2bpp.html) is just the graphics format that's used on gameboys. The PNG assets are converted to 2BPP automatically during `make` via `rgbgfx -o`. Note that the PNG are the sources so it's important (and much easier) to edit those files rather than their 2BPP counterparts as those will be overwriten during compilation.

## 2. Loading the sprites into ROM
#### Let's create a new section where we'll load the assets we just created:

*gfx/sprites.asm*
```diff
EnteiSpriteGFX::               INCBIN "gfx/sprites/entei.2bpp"
 RaikouSpriteGFX::              INCBIN "gfx/sprites/raikou.2bpp"
 StandingYoungsterSpriteGFX::   INCBIN "gfx/sprites/standing_youngster.2bpp"
+
+SECTION "Sprites 3", ROMX
+
+ChrisRunSpriteGFX::            INCBIN "gfx/sprites/chris_run.2bpp"
+KrisRunSpriteGFX::             INCBIN "gfx/sprites/kris_run.2bpp"
```

#### We can now declare where in the ROM the "Sprites 3" should be loaded. In our case, we're using bank `$7e`:

*layout.link*
```diff
        "Mobile News Data"
 ROMX $7e
        "Crystal Events"
+       "Sprites 3"
 ROMX $7f
        org $7de0
        "Stadium 2 Checksums"
```

If you're curious as to how I decided on this bank, I just tried adding "Sprites 3" to all the ROMX sections bottom up, compiling between each attempt until I found a bank that could fit the new assets. :P

## 3. Contextualizing the assets and adding player run state
#### Contextualizing the assets as sprites:

*data/sprites/sprites.asm*
```diff
        overworld_sprite EnteiSpriteGFX, 4, STILL_SPRITE, PAL_OW_RED
        overworld_sprite RaikouSpriteGFX, 4, STILL_SPRITE, PAL_OW_RED
        overworld_sprite StandingYoungsterSpriteGFX, 12, STANDING_SPRITE, PAL_OW_BLUE
+       overworld_sprite ChrisRunSpriteGFX, 12, WALKING_SPRITE, PAL_OW_RED
+       overworld_sprite KrisRunSpriteGFX, 12, WALKING_SPRITE, PAL_OW_BLUE
        assert_table_length NUM_OVERWORLD_SPRITES
```

#### Creating constants for the sprites:

*constants/sprite_constants.asm*
```diff
        const SPRITE_ENTEI ; 64
        const SPRITE_RAIKOU ; 65
        const SPRITE_STANDING_YOUNGSTER ; 66
+       const SPRITE_CHRIS_RUN; 67
+       const SPRITE_KRIS_RUN; 68
 DEF NUM_OVERWORLD_SPRITES EQU const_value - 1
```
If you're wondering here how SPRITE_X_RUN connects to XRunSpriteGFX, I _think_ this is done via the order of declaration in *data/sprites/sprites.asm*

#### Creating a new player running state

*constants/wram_constants.asm*
```diff
 DEF PLAYER_SKATE     EQU 2
 DEF PLAYER_SURF      EQU 4
 DEF PLAYER_SURF_PIKA EQU 8
+DEF PLAYER_RUN       EQU 16
```
I picked 16 here because it's an order of magnitude over 8 in binary to follow existing convention.

#### And finally associating the sprites to the new run state for each Chris and Kris

*data/sprites/player_sprites.asm*
```diff
 ChrisStateSprites:
        db PLAYER_BIKE,      SPRITE_CHRIS_BIKE
        db PLAYER_SURF,      SPRITE_SURF
        db PLAYER_SURF_PIKA, SPRITE_SURFING_PIKACHU
+       db PLAYER_RUN,       SPRITE_CHRIS_RUN
        db -1 ; end

 KrisStateSprites:
        db PLAYER_BIKE,      SPRITE_KRIS_BIKE
        db PLAYER_SURF,      SPRITE_SURF
        db PLAYER_SURF_PIKA, SPRITE_SURFING_PIKACHU
+       db PLAYER_RUN,       SPRITE_KRIS_RUN
        db -1 ; end
```

#### At this stage, it's a good idea to compile and make sure everything is a-ok:

```bash
make
```

## 4. Creating logic for run and walk state transitions with sprite transitions
#### We'll need transition functions to sequentially change the player state to PLAYER_RUN and trigger a sprite refresh to load the sprites for the new state:

*engine/overworld/player_movement.asm*
```diff
        pop bc
        ret
 
+.StartRunning:
+       push bc
+       ld a, PLAYER_RUN
+       ld [wPlayerState], a
+       call UpdatePlayerSprite
+       pop bc
+       ret
+
+.StartWalking:
+       push bc
+       ld a, PLAYER_NORMAL
+       ld [wPlayerState], a
+       call UpdatePlayerSprite
+       pop bc
+       ret
+
 CheckStandingOnIce::
        ld a, [wPlayerTurningDirection]
        cp 0
```

#### Next, adding a function to check that B is held down and another to check if the player is standing still:

*engine/overworld/player_movement.asm*
```diff
        cp PLAYER_SKATE
        ret
 
+.CheckBHeldDown:
+       ldh a, [hJoypadDown]
+       and B_BUTTON
+       cp B_BUTTON
+       ret
+
+.CheckStandingStill
+       ld a, [wWalkingDirection]
+       cp STANDING
+       ret
+
 .CheckWalkable:
 ; Return 0 if tile a is land. Otherwise, return carry.
```

#### At this point, we've the following inventory of helpers:
1. `.StartRunning` which changes the player state to `PLAYER_RUN` and triggers a sprite refresh
2. `.StartWalking` which changes the palyer state to `PLAYER_NORMAL` and triggers a sprite refresh
3. `.CheckBHeldDown` which checks if... B is held down
4. `.CheckStandingStill` which checks if the player is standing still

## 5. Triggering and un-triggering player state transitions
Almost there!

#### Let's create functions that'll conditionally trigger our `.StartRunning` and `.StartWalking` transition functions and set the appropriate movement speeds for the player:

*engine/overworld/player_movement.asm*
```diff
+.shouldrun
+       ld a, [wPlayerState]
+       cp PLAYER_RUN
+       call nz, .StartRunning
+       jr .fast
+
+.shouldwalk
+       ld a, [wPlayerState]
+       cp PLAYER_NORMAL
+       call nz, .StartWalking
+       jr .walk
+
 .fast
        ld a, STEP_BIKE
        call .DoStep
```

Calling `.shouldrun` and `.shouldwalk` instead of the transition functions directly ensures that we only trigger transitions when the player is actually transiting from walk to run or vice versa. For instance, calling `.StartRunning` when the player is already running would cause the sprite to be reloaded via `UpdatePlayerSprite` but that's not the case with `.shouldrun`. Without this intermediate step, noticeable lag is introduced at each step by the continous reloading.

#### Next, we introduce a function to coordinate everything we've created above:

*engine/overworld/player_movement.asm*
```diff
DoPlayerMovement::
        scf
        ret
 
+.HandleWalkAndRun
+       call .CheckStandingStill
+       jr z, .shouldwalk
+       call .CheckBHeldDown
+       jr z, .shouldfast
+       jr .shouldwalk
```

Here, we ensure the player returns to the walking state whenever they're standing still and appropriatly jumps to the running state when B is held down or the walking state otherwise.

#### Finally, we can trigger all of this whenever the player takes a step:

*engine/overworld/player_movement.asm*
```diff
        call CheckIceTile
        jr nc, .ice
 
+       call .BikeCheck
+       jr nz, .HandleWalkAndRun
+
 ; Downhill riding is slower when not moving down.
        call .BikeCheck
        jr nz, .walk
```

We use the existing `.BikeCheck` so that we only consider transiting when we're not on a bike. Not doing so leads to the player getting off the bike immediately after getting on.

#### Compile and try it out!
```bash
make
```

If everything went well, the player will now jump onto a bike whenever B is held down while they're moving in the overworld. Time to add some sprites :)

## 6. Tips for creating running sprites
- You probably don't want to create running sprites from scratch. Instead, use the walking variants as reference. We don't really need the previous sprites anymore now that we've confirmed that they're loading properly so just override them with the walking variants.
- You'll want to get a pixel art editor of some kind to make changes your reference sprites. I've found [Pixelorama](https://orama-interactive.itch.io/pixelorama) to work really well. Whatever you use, make sure that you're saving in lossless PNG and not JPEG.
- I've found the following are sufficient to create natural looking running animations:
  - Lowering the head and moving it forward (we don't run standing upright)
  - Exaggerating the movement of hands pulling them away from the body
  - Shifting the position of the backpack (leave it in one frame but shift it up or down in another)
  - Ensuring that the head is at the same location between frames (not doing so gives a pacman effect, which is actually kinda neat)

And that's it! Have fun! :D

[sec-1]: #1-adding-placeholder-sprite-assets
[sec-2]: #2-loading-the-sprites-into-ROM
[sec-3]: #3-contextualizing-the-assets-and-adding-player-run-state
[sec-4]: #4-creating-logic-for-run-and-walk-state-transitions-with-sprite-transitions
[sec-5]: #5-triggering-and-un-triggering-player-state-transitions
[sec-6]: #6-tips-for-creating-running-sprites

