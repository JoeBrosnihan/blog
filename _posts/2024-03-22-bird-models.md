---
layout: post
title:  "Building a UGC Economy in BIRD"
date:   2024-03-22 19:55:00 -0700
---

We built a UGC platform in BIRD on Roblox.

<iframe width="560" height="315" src="https://www.youtube.com/embed/Amwp6iZ377I" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

We did this because the our game has a community of young people excited to create things, and an audience of players excited to play with new creations. We connected these creators to the audience so we, the small team of first-party developers, would no longer be the content bottleneck.

So far, it's worked out incredibly well. BIRD's daily revenue has doubled because players want to pay to insert other creators' models, and we've paid 300k R$ (3k USD) to our creators.

Roblox doesn't support self-serve "modding" well enough for most developers to add it to their experiences. We spent ~4 months implementing this system. If Roblox improves platform support for this, this could take future developers a couple days. I will describe some challenges we faced building this system I hope become easier for future devs.

- Inserting Models requires obscure workarounds
- Scripts cannot be sandboxed
- Asset system and Toolbox must be rebuilt from scratch
- Creators must be manually paid regularly

### Inserting Models
We have to use an obscure workaround for inserting Model Assets, which forces the models to conform to a BIRD-specific format.
- We cannot insert arbitrary Models from the Toolbox. Players must make them for BIRD.
- Over 50% of our creators who attempt to make a Model fail because their format is wrong.

We first tried using Roblox's function [InsertService:LoadAsset(id)](https://create.roblox.com/docs/reference/engine/classes/InsertService#LoadAsset) to insert player Model assets. We found this errors if you try to insert a Model made by anyone other than the experience creator. Other developers have worked around this:
- F3X in-game building tools import player models through HTTPService
- Various admin-command tools use `require(assetId)` to load command code
We discovered HD Admin has not only code in the asset it requires, but also spawn-able Tools and NPC models parented under its ModuleScript Instance. BIRD uses this mechanism.

Players must put what they want to insert as children of a specific ModuleScript we give them. The ModuleScript must be named "MainModule" and it must contain a specific snippet of code.
```
// MainModule source
return script:GetChildren()
```
Then, the player publishes this as an Asset on Roblox and pastes the ID into BIRD. BIRD gets the actual models the player made like this:
```
// BIRD code
local playersModels = require(pastedAssetId)()
for i, model in playersModels do
	model.Parent = workspace
end
```
Note - Because we're forced to get the player models by requiring a ModuleScript by assetId, there's no opportunity for us to strip player scripts. A player could insert whatever code they want on the line before `return script:GetChildren()` and we'd have no way of detecting or preventing that code from running.

We've done our best to simplify this through in-game Video tutorials, error messages, examples on GitHub, and even community-made tutorials. This was a lot of work we could have avoided if we could just use `LoadAsset`.

### Sandboxing Scripts
Players can include Scripts in their Models. We've seen them make insanely creative NPCs (a robot you can command to chase and kill other players), Vehicles (mining drills, helicopters), and even minigames (a Battle Royale game mode).

Sandboxing these scripts simply is not feasible. The Script.Source property isn't accessible at runtime, so we cannot parse script contents in any way to block malicious activity like wiping our DataStores.

In fact, we think someone tried to do that. A player created a model titled "BIRD Data Destroy." Interestingly, before the player was able to insert it in the game, they got a 1 day ban and the Model asset was deleted. Our hypothesis is some Roblox automation detected malicious code when they published their asset to the Creator Store.

BIRD happens to not store any persistent data in the first place, we can tolerate players wiping our DataStores. Other Roblox experiences cannot. It's unlikely they'd ever allow player script execution without guarantees to limit script capabilities.

### Rebuilding the Toolbox
It took us (two programmers working part-time) 4 months to rebuild features Roblox has built but not yet exposed to developers
- A query-able database of known BIRD-format assets, since there's no lua API for searching Roblox's catalog of Model Assets
- Asset management flows - creation, deletion, updating name/description/public-visibility.
- The entire Toolbox UI. Browsable grid of thumbnails, expanded asset details, search by keyword/creator, usage statistics

If the Studio Toolbox were open source and all its dependent APIs worked in-game, we'd have just dropped that in with light modification, in a day or two. We'd probably make the background pink to fit our game and that's it.

### Manual Payouts
Every weekend, I manually pay out our creators through the Group.

On Sunday, I run a script that prints out the list of creators and amounts due. I paste these values into the website for a One Time group payout. This is tedious and error prone. It can take up to an hour now there are 30+ players making content every week and I need to carefully follow a few steps for each player, including giving them a cosmetic "Model Maker" role in the group.

There is a period after joining a group in which a player is "too new" to receive any payouts. (1 week?) In the worst case, a creator may wait 2 weeks to get any payout for their creation.
