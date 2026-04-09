# Xenon Module Loader

A multi-phase module loading system for Bloxd.io worlds. Modules are stored in Code Blocks at fixed coordinates and loaded automatically on server start. The loader intercepts all Bloxd.io callbacks and dispatches them to registered module handlers.

---

## Writing a Module

A module is JavaScript stored in a Code Block at one of the 22 reserved slots. The loader reads, compiles, and executes it automatically. You just write handler functions and top-level variables — the loader handles the rest.

```js
var COINS = {};

onPlayerJoin = function(pid) {
    COINS[pid] = 0;
};

onPlayerLeave = function(pid) {
    delete COINS[pid];
};

onPlayerKilledOtherPlayer = function(attacker, killed) {
    COINS[attacker] = (COINS[attacker] || 0) + 10;
    api.sendMessage(attacker, "+10 coins!");
    return "keepInventory";
};
```

### How the loader processes your module

1. **PREP** — Strips `var`/`let`/`const` from top-level declarations. `var COINS` becomes `this.COINS`, scoped to your module and not polluting globals.
2. **COMPILE** — Wraps your code in `with(shared){...}` so shared state and `mango`/`xenon` are directly accessible by name.
3. **EXEC** — Runs the compiled code. Your handler assignments are captured.
4. **REGISTER** — Handler functions are extracted and added to the dispatch table.

### Handler naming

The loader detects handlers by matching function names against known hook names. Any function whose name starts with a hook name followed by `_` is registered:

```js
onPlayerJoin = function(pid) { ... }
onPlayerJoin_setup = function(pid) { ... }
onPlayerJoin_coins = function(pid) { ... }
```

All three register as `onPlayerJoin` handlers. The suffix after `_` is just for your own organization. Multiple handlers for the same hook in the same module all run.

### Handler priority

Add a `priority` export to control ordering across modules. Higher runs first. Default is `0`.

```js
priority = 10;

onPlayerJoin = function(pid) {
    // this runs before modules with priority < 10
};
```

### Top-level variables

Top-level `var` declarations are scoped to your module — they won't collide with other modules even if they use the same name.

```js
var DATA = {};   // becomes this.DATA — safe, module-local
```

Do not use `globalThis.X =` to declare module state. Use `shared.X =` if you want something accessible from other modules.

---

## Hooks

### Void hooks

Return values are ignored. All registered handlers always run.

| Hook | Args |
|---|---|
| `tick` | `ms` |
| `onPlayerJoin` | `pid, fromReset` |
| `onPlayerLeave` | `pid, shuttingDown` |
| `onPlayerKilled` | `pid` |
| `onEntityKilled` | `eid` |
| `onPlayerFinishChargingItem` | `pid, used, itemName, duration` |
| `onPlayerBoughtShopItem` | `pid, categoryKey, itemKey, item, userInput` |
| `onPlayerSelectInventorySlot` | `pid, slotIndex` |

### Intercept hooks

Return values matter. The first non-void return wins. Returning `false` short-circuits immediately — remaining handlers are skipped and the action is blocked.

| Hook | Args | Return to block |
|---|---|---|
| `onPlayerChat` | `pid, message, channel` | `false` |
| `onPlayerAttemptOpenChest` | `pid, x, y, z, isMoonstone, isIron` | `"preventOpen"` |
| `onAttemptKillPlayer` | `killed, attacker` | `"preventDeath"` |
| `onPlayerDamagingOtherPlayer` | `attacker, damaged, dmg, item, bodyPart, dmgDbId` | `"preventDamage"` or number |
| `onMobDamagingPlayer` | `mob, pid, dmg, item` | `"preventDamage"` or number |
| `onPlayerDamagingMob` | `pid, mobId, dmg, item, dmgDbId` | `"preventDamage"` or number |
| `onPlayerKilledMob` | `pid, mobId, dmg, item` | `"preventDrop"` |
| `onPlayerKilledOtherPlayer` | `attacker, killed, dmg, item` | `"keepInventory"` |
| `onMobKilledPlayer` | `mob, pid, dmg, item` | `"keepInventory"` |
| `playerCommand` | `pid, command` | `false` |
| `onPlayerMoveItemOutOfInventory` | `pid, itemName, amount, fromIdx, type` | `"preventChange"` |
| `onPlayerMoveInvenItem` | `pid, ...` | `"preventChange"` |
| `onPlayerMoveItemIntoIdxs` | `pid, ...` | `"preventChange"` |
| `onPlayerSwapInvenSlots` | `pid, ...` | `"preventChange"` |
| `onPlayerMoveInvenItemWithAmt` | `pid, ...` | `"preventChange"` |
| `onPlayerClick` | `pid, ...` | `false` |
| `onPlayerAttemptAltAction` | `pid, ...` | `false` |

---

## Shared State

Modules run inside `with(shared){...}`. Anything you put on `shared` is accessible by name in every module without prefixing.

```js
// moduleA — publish something
shared.RANKS = { "PlayerName": "Admin" };

// moduleB — consume it
onPlayerJoin = function(pid) {
    var name = api.getEntityName(pid);
    var rank = RANKS[name] || "Member";  // works directly — no shared. prefix needed
};
```

After all modules load, everything in `shared` is also copied to `globalThis`, so it's accessible from Press-to-Code boards, Code Blocks outside the module system, and World Code.

### Built-in shared values

| Name | Description |
|---|---|
| `mango` | Timer and utility object |
| `xenon` | Extended utilities |
| `KEEP_INVENTORY_ON_KILL` | Boolean from loader config |

---

## exec()

`exec` lets you inject and run code at runtime without placing a Code Block. Useful for map building, live debugging, and temporary features.

```js
exec(code, name, verbose)
```

| Arg | Type | Description |
|---|---|---|
| `code` | string | JavaScript source to run |
| `name` | string | Base name for this exec. Auto-increments: `shop_0`, `shop_1`, etc. |
| `verbose` | boolean | If true, sends load confirmation to admins |

The loader queues execs and processes them across ticks, same as boot modules.

### Context variables

Every exec automatically has `myId`, `thisPos`, and `ownerDbId` available by name — no hardcoding needed.

```js
exec(
  'api.sendMessage(myId, "Hello from exec! pos: " + JSON.stringify(thisPos));',
  "test",
  true
);
```

### Hooks in execs

Execs can register handlers exactly like modules:

```js
exec(
  'onPlayerJoin = function(pid) { api.sendMessage(pid, "Welcome!"); };',
  "welcome",
  true
);
```

### Self-disposing execs

Use `dispose("name")` inside exec code to clean itself up. The loader automatically rewrites it to target the correct versioned key.

```js
exec(
  'mango.delay(function() { dispose("tempBuff"); }, 30000);' +
  'onPlayerJoin = function(pid) { api.applyEffect(pid, "Speed", 5000); };',
  "tempBuff",
  true
);
```

---

## dispose()

Removes a previously exec'd module by name. Cleans up its handlers, shared state keys, global keys, and timers.

```js
dispose(name)
dispose(name, verbose)
```

`dispose("shop")` removes all `shop_*` versions. `dispose("shop_2")` removes only that specific one.

Boot modules (loaded from Code Blocks at startup) cannot be disposed — only exec'd code can be cleaned up.

---

## mango

`mango` is the loader's timer and utility object. Available in all modules and execs by name.

### mango.delay(fn, ms) → handle

Runs `fn` once after `ms` milliseconds.

```js
mango.delay(function() {
    api.broadcastMessage("Event starting!");
}, 10000);
```

### mango.interval(fn, ms) → handle

Runs `fn` every `ms` milliseconds. Return `false` from the callback to stop it.

```js
var h = mango.interval(function() {
    var ids = api.getPlayerIds();
    for (var i = 0; i < ids.length; i++) {
        api.applyEffect(ids[i], "Health Regen", 3000);
    }
}, 5000);
```

### mango.cancel(handle) → boolean

Cancels a `delay` or `interval` using the handle returned when it was created. Returns `true` if found and removed.

```js
var h = mango.interval(fn, 1000);
mango.cancel(h);
```

### mango.once(fn) → fn

Wraps a function so it only executes on the first call. All subsequent calls are no-ops.

```js
var init = mango.once(function() {
    api.broadcastMessage("Server ready!");
});
```

### mango.guard(fn) → fn

Wraps a function in a try/catch. Errors are logged to admins via the error system instead of crashing.

```js
var safe = mango.guard(function() {
    doRiskyThing();
});
```

### mango.require(fn)

Runs `fn` immediately if the loader is already finished, otherwise queues it to run once loading completes.

```js
mango.require(function() {
    // safe to access shared state from other modules here
    COINS["_Xenon_"] = 9999;
});
```

### mango.every(n) → boolean

Returns `true` every Nth call. Internal counter increments each call. Useful inside `tick` to throttle work.

```js
tick = function(ms) {
    if (mango.every(20)) {
        // runs every 20 ticks (~1 second)
        checkLeaderboard();
    }
};
```

---

## xenon

`xenon` is the extended utility object. Available everywhere by name after the loader finishes.

### xenon.isAdmin(pid) → boolean

Returns `true` if the player is in the admin list.

```js
playerCommand = function(pid, cmd) {
    if (!xenon.isAdmin(pid)) return false;
    // admin-only logic
};
```

### xenon.onFunction(name, fn, priority)

Intercepts a shared function. Your `fn` runs every time the original is called, receiving the same arguments. Return `false` to block the original from running. Return anything else or nothing to let it proceed.

Multiple listeners on the same function run in priority order (higher first), same as hook dispatch.

The function must already be on `shared` to be wrapped. If called before the loader finishes, it automatically queues until ready — no `mango.require` needed.

```js
// assume shared.giveCoins exists from another module
xenon.onFunction("giveCoins", function(pid, amt) {
    api.sendMessage(pid, "You received " + amt + " coins!");
});

// block the function conditionally
xenon.onFunction("giveCoins", function(pid, amt) {
    if (isEventLocked) return false;
}, 10);
```

Anti-recursion is built in — if `giveCoins` calls itself internally, the listener only fires once.

Internal loader functions (`mango`, `xenon`, `exec`, `dispose`, etc.) cannot be wrapped.

### xenon.formattedTime(tzOffset, hemisphere) → object

Returns a complete breakdown of the current real-world time. `tzOffset` is a UTC offset in hours (e.g. `-7` for PDT). `hemisphere` is `"N"` or `"S"` and affects season calculation.

```js
var t = xenon.formattedTime(-5, "N");
```

| Field | Type | Description |
|---|---|---|
| `hour` | number | 0–23 |
| `hour12` | number | 1–12 |
| `minute` | number | 0–59 |
| `second` | number | 0–59 |
| `ms` | number | 0–999 |
| `ampm` | string | `"AM"` or `"PM"` |
| `year` | number | e.g. `2026` |
| `month` | number | 1–12 |
| `day` | number | 1–31 |
| `weekday` | number | 0 (Sun) – 6 (Sat) |
| `weekdayName` | string | `"Monday"` etc. |
| `monthName` | string | `"April"` etc. |
| `yearDay` | number | 1–366, day of year |
| `week` | number | 1–53, ISO week number |
| `isLeapYear` | boolean | |
| `daysInMonth` | number | Days in current month |
| `daysLeftInYear` | number | |
| `daysSinceEpoch` | number | Days since Jan 1 1970 |
| `isWeekend` | boolean | Sat or Sun |
| `isWeekday` | boolean | Mon–Fri |
| `season` | string | `"Spring"`, `"Summer"`, `"Autumn"`, `"Winter"` |
| `seasonDay` | number | Day within current season |
| `daysLeftInSeason` | number | |
| `dayProgress` | number | 0.0–1.0, progress through current day |
| `seasonProgress` | number | 0.0–1.0, progress through current season |
| `yearProgress` | number | 0.0–1.0, progress through current year |

Seasons use meteorological boundaries (Dec/Jan/Feb = Winter, Mar/Apr/May = Spring, Jun/Jul/Aug = Summer, Sep/Oct/Nov = Autumn). Southern hemisphere flips them.

```js
tick = function(ms) {
    if (!mango.every(200)) return;
    var t = xenon.formattedTime(-5, "N");
    // drive a day/night cycle
    var brightness = Math.sin(t.dayProgress * Math.PI);
};
```

---

## Reserved names

Do not define these names at the top level of a module — the loader will warn you with a COLLISION message:

`ADMIN_NAMES`, `KEEP_INVENTORY_ON_KILL`, `MODULES_PER_TICK_MIN`, `MODULES_PER_TICK_MAX`, `MODULES_PLAYER_MAX`, `WARMUP_COOLDOWN_MS`, `MODULE_RETRY_MS`, `MODULE_MAX_ERRORS`, `OP_NONE`, `OP_PREP`, `OP_COMPILE`, `OP_EXEC`, `OP_REGISTER`, `VOID_HOOKS`, `INTERCEPT_HOOKS`, `mods`, `pub`, `ST`, `mango`, `xenon`, `keyOf`, `notify`, `runVoid`, `runIntercept`, `Loaded`, `cL`, `exec`, `dispose`

---

## Error handling

If a module throws during any load phase it is retried after 1 second. After 10 failures the module is permanently disabled for the session. Admins see:

- A message with the error, phase name, and line number
- A direction arrow pointing at the offending Code Block

Hook-level errors (thrown inside a handler during gameplay) are caught, logged, and throttled — one report per coord+hook pair per 30 seconds.

---

## Examples

### Coins module

```js
var COINS = {};

onPlayerJoin = function(pid) {
    COINS[pid] = 0;
};

onPlayerLeave = function(pid) {
    delete COINS[pid];
};

onPlayerKilledOtherPlayer = function(attacker, killed) {
    COINS[attacker] = (COINS[attacker] || 0) + 10;
    api.sendMessage(attacker, [
        { str: "+10 ", style: { color: "#fbbf24", fontWeight: "bold" } },
        { str: "coins" }
    ]);
    return "keepInventory";
};

playerCommand = function(pid, cmd) {
    if (cmd === "coins") {
        api.sendMessage(pid, "You have " + (COINS[pid] || 0) + " coins.");
        return false;
    }
};

shared.COINS = COINS;
shared.addCoins = function(pid, amt) {
    COINS[pid] = (COINS[pid] || 0) + amt;
};
```

### Intercepting a shared function from another module

```js
// runs after coins module loads
xenon.onFunction("addCoins", function(pid, amt) {
    if (amt > 1000) {
        api.sendMessage(pid, "Large deposit: " + amt + " coins!");
    }
});
```

### Temporary exec-based buff

```js
exec(
    'onPlayerJoin = function(pid) {' +
    '    api.applyEffect(pid, "Speed", 999999);' +
    '};' +
    'mango.delay(function() { dispose("speedBuff"); }, 60000);',
    "speedBuff",
    true
);
```

### Day/night cycle driven by real time

```js
var LAST_PERIOD = "";

tick = function(ms) {
    if (!mango.every(100)) return;
    var t = xenon.formattedTime(-5, "N");
    var period = t.hour >= 6 && t.hour < 20 ? "day" : "night";
    if (period === LAST_PERIOD) return;
    LAST_PERIOD = period;
    if (period === "night") {
        api.broadcastMessage("Night has fallen.");
    } else {
        api.broadcastMessage("A new day begins.");
    }
};
```
