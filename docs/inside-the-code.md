# Inside the Code

The full Apex_AI codebase is **12,000+ lines of JavaScript** and lives in a private repository. These are real, unedited excerpts that show how the supervisor architecture actually works — the parts I'm proudest of.

---

## 1. The Reflex Layer: Panic Swimmer & Lava Flee

Pathfinding libraries are great at walking and terrible at not drowning. This handler runs on **every physics tick (20 Hz)** and sits *below* the AI in the control stack — when it fires, it takes the controller away from whatever the AI was doing. It's the software equivalent of a hardware interrupt.

```js
const inLava  = (headBlock && headBlock.name.includes('lava')) || (feetBlock && feetBlock.name.includes('lava'));
const inWater = (headBlock && (headBlock.name === 'water' || headBlock.name === 'flowing_water'));

if (inLava) {
    bot.setControlState('jump', true);
    bot.setControlState('back', true);
    bot.setControlState('sprint', true);
    isFleeingLava = true;
} else if (isFleeingLava) {
    bot.clearControlStates();
    isFleeingLava = false;
} else if (inWater) {
    bot.setControlState('jump', true);
    isDrowning = true;
} else if (isDrowning) {
    bot.setControlState('jump', false);
    isDrowning = false;
}
```

Every other system in the bot checks these flags before acting — combat, eating, mining all yield to the reflex layer:

```js
bot.on('physicTick', () => {
    if (!bot.entity || isFleeingLava || isDrowning) return;
    // ... combat reflex, emergency heal, etc.
});
```

---

## 2. The "Lunchbox" Rule: Prerequisite Enforcement

The AI can *decide* to tunnel to Y=16 for iron, but the supervisor won't *let* it leave the surface without supplies. If any prerequisite is missing, the requested action is silently replaced with the action that fixes the gap — and the chain re-runs until the checklist passes.

```js
else if (action === 'MINE_TO') {
    const targetY = parseInt(target);
    const p = bot.entity.position;

    // Leaving the surface for a deep mine? Check the lunchbox first.
    if (p.y > 50 && targetY < 50) {
        const inv = getInventory();
        const totalWood = inv.logs + Math.floor(inv.planks / 4);
        if (totalWood < 10 || inv.furnaces === 0 || (inv.tables === 0 && !env.table) || inv.bestPickaxe === 'none') {
            if (totalWood < 10) return await runAction('GATHER:WOOD', env);
            if (inv.tables === 0 && !env.table) return await runAction('CRAFT:CRAFTING_TABLE', env);
            if (inv.furnaces === 0) {
                if (inv.toolStone >= 8) return await runAction('CRAFT:FURNACE', env);
                return await runAction('GATHER:STONE', env);
            }
            if (inv.bestPickaxe === 'none') return await runAction('CRAFT:WOODEN_PICKAXE', env);
        }
    }
    // ... proceed with tunneling
```

---

## 3. The LLM Tiebreaker: Prompt Design

The deterministic planner handles ~95% of decisions for free. Only when it's genuinely stuck does the bot phone the LLM — and the prompt is engineered to be **cheap, fast, and machine-parseable**: compressed world state in, exactly one action token out.

```js
async function consultLLM(inv, env, stuckInfo) {
    const currentY = Math.floor(bot.entity.position.y);
    const visible = Object.keys(env).filter(k => env[k]).join(',') || 'nothing';

    const prompt = `You are Apex_AI. Planner is stuck.

State: Y=${currentY}, Tool=${inv.bestPickaxe}, Armor=${inv.equippedArmor.length}/4, Food=${bot.food}/20
Resources: logs=${inv.logs} stone=${inv.toolStone} iron=${inv.ironIngots} diamond=${inv.diamonds} obsidian=${inv.obsidian}
Visible: ${visible}
Phase: portalBuilt=${hasCelebrated}, portalLit=${portalLit}, endPortal=${endPortalActive}, dragonDead=${dragonDefeated}

Stuck: ${stuckInfo}

Actions: [GATHER:WOOD|STONE|IRON|DIAMOND|OBSIDIAN] [MINE_TO:70|16|-58] [CRAFT:...] [SMELT:IRON] [HUNT:...] [BUILD:PORTAL] ...

Pick one action. Reason in 1 short sentence, then output [ACTION] at the end.`;

    const response = await openai.chat.completions.create({
        model: "gpt-5.1",
        messages: [
            { role: "system", content: "You are a Minecraft strategy advisor. Be decisive." },
            { role: "user", content: prompt }
        ],
        // gpt-5 is a reasoning model: skip extended reasoning and bound
        // the output — we want the fastest possible single-action answer.
        reasoning_effort: "minimal",
        max_completion_tokens: 400
    });

    const full = response.choices[0].message.content;
    const matches = full.match(/\[(.*?)\]/g);
    const raw = matches ? matches[matches.length - 1] : '[EXPLORE]';
    const action = raw.replace(/\[|\]/g, '').toUpperCase();
    // ...
}
```

Note the failure path: if the API is unreachable, the bot doesn't crash or freeze — it defaults to `EXPLORE` and keeps playing.

---

## 4. Never Trust the Model: The Action Sanitizer

Whatever the LLM answers gets validated against the bot's **real** inventory before execution. If the model hallucinates equipping armor it doesn't own, or building a portal without obsidian, the supervisor swaps in a corrective action instead:

```js
function sanitizeLLMAction(llmAction, inv) {
    const parts = llmAction.split(':');
    const verb = parts[0];
    const target = parts.slice(1).join(':');

    if (verb === 'EQUIP' && !inv.ownedArmor.includes(target.toLowerCase())) {
        return { action: 'EXPLORE', overridden: true, reason: 'LLM picked EQUIP for unowned item.' };
    }
    if (verb === 'BUILD' && target === 'PORTAL' && inv.obsidian < 10) {
        return { action: 'GATHER:OBSIDIAN', overridden: true, reason: 'LLM picked BUILD:PORTAL without obsidian.' };
    }
    return { action: llmAction, overridden: false };
}
```

---

## 5. The Greed Cut-Off & Tech-Tree Fallback Chain

Two guardrails in one handler. First, the hard cap: at 24 raw iron, mining commands are overridden and the bot is forced to go smelt. Second, the fallback chain: every gathering target validates its tool prerequisite and **recursively downgrades** to the action that unblocks it — ask for diamonds with a stone pickaxe and you'll be sent to get iron first.

```js
else if (action === 'GATHER') {
    // Greed cut-off: enough iron banked — stop mining, start smelting.
    if (target === 'IRON' && currentInv.rawIron >= 24) {
        return await runAction('PLACE:FURNACE', env);
    }

    // Tech-tree enforcement: wrong tool for the target? Recurse downward.
    if (target === 'STONE'    && !bestToolName)                                return await runAction('GATHER:WOOD', env);
    if (target === 'IRON'     && (!bestToolName || bestToolName === 'wooden_pickaxe')) return await runAction('GATHER:STONE', env);
    if (target === 'DIAMOND'  && bestToolName !== 'iron_pickaxe'
                              && bestToolName !== 'diamond_pickaxe')           return await runAction('GATHER:IRON', env);
    if (target === 'OBSIDIAN' && bestToolName !== 'diamond_pickaxe')           return await runAction('GATHER:DIAMOND', env);
    // ...
```

---

## 6. The Ingredient Cascade

When a craft needs sticks and there are none, the code walks backward down the recipe tree on its own: logs → planks → sticks. The AI never has to know the intermediate steps existed.

```js
async function ensureSticks() {
    if (invCount('stick') >= 1) return true;
    if (plankTotal() < 2) {
        const log = bot.inventory.items().find(i => i.name.endsWith('_log'));
        if (log) {
            const pd = mcData.itemsByName[log.name.replace('_log', '_planks')];
            const pr = pd && bot.recipesFor(pd.id, null, 1, null)[0];
            if (pr) { try { await bot.craft(pr, 1, null); } catch (e) {} }
        }
    }
    if (plankTotal() < 2) return false;
    const sd = mcData.itemsByName['stick'];
    const sr = sd && bot.recipesFor(sd.id, null, 1, null)[0];
    if (sr) { try { await bot.craft(sr, 1, null); } catch (e) {} }
    return invCount('stick') >= 1;
}
```

---

## 7. Mid-Flight Replanning

A 60-block tunnel dig can take minutes — plenty of time for the world to change. During long actions, a watcher re-runs the deterministic planner every 2 seconds. If a *different* high-confidence plan emerges, it kills the current pathfinding goal mid-flight rather than finishing a stale decision:

```js
const watcher = setInterval(() => {
    const curInv = getInventory();
    const curPlan = plan(curInv, freshEnv);
    const newVerb = curPlan.action.split(':')[0];
    if (curPlan.action !== actionString &&
        curPlan.confidence === 'high' &&
        abortableVerbs.includes(newVerb)) {
        abortReason = `${curPlan.action} — ${curPlan.reason}`;
        console.log(`🛑 Aborting [${actionString}] mid-flight: planner now wants [${curPlan.action}]`);
        bot.pathfinder.setGoal(null);
    }
}, 2000);
```

---

## 8. The Streamer Persona (Cost-Optimized Commentary)

Strategy and personality run on **separate models**. GPT-5.1 makes decisions; a cheap `gpt-4o-mini` layer provides live trash-talk with rate-limiting, idle banter, and a one-flag fallback to free templated lines:

```js
const PERSONA = {
    name: 'Apex',
    voice: "You are Apex, an AI speedrunner livestreaming Minecraft. You are cocky, " +
           "dramatic and funny, with the energy of a Twitch gremlin. You hype your wins, " +
           "rage-tilt at deaths, trash-talk mobs, and talk straight to chat. Keep it PG-13."
};

const commentaryConfig = {
    useLLM: true,             // false = templated lines only (zero API cost)
    model: 'gpt-4o-mini',     // cheap/fast chat model
    cooldownMs: 9000,         // minimum gap between spoken lines
    idleBanterMs: 45000,      // drop an idle line if nothing has happened for this long
    verbosePlanner: false     // true = also chat the dry planner reasoning (noisy on stream)
};
```

The bot also persists lifetime stats (`deaths`, `mobKills`, `blocksMined`, `diamondsFound`...) across sessions — so it can brag accurately.

---

*Want to see more? The full source is private, but I'm happy to walk through it — reach out via my GitHub profile.*
