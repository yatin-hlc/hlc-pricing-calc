# HLC Pricing Calculator — Developer Rules

Single-file app: `index.html`. Hosted on GitHub Pages.
Firebase Realtime Database: `half-light-coffee-calc-default-rtdb.firebaseio.com`
All data lives under `hlc/` in Firebase.

---

## MANDATORY DATA SAFETY RULES — check before every commit

### Never replace an entire Firebase collection
These patterns are FORBIDDEN:
```js
db.ref('hlc/logs').set(logs)           // ❌ wipes all logs
db.ref('hlc/skus').set(skus)           // ❌ wipes all SKUs
db.ref('hlc/cropEntries').set(...)     // ❌ wipes all crop entries
db.ref('hlc/producers').set(...)       // ❌ wipes all producers
saveData('logs', logs)                 // ❌ same — saveData does .set()
saveData('skus', skus)                 // ❌ same
```

These are the ONLY allowed patterns for user-generated records:
```js
// Save one item
db.ref('hlc/logs/' + log.id).set(log)
db.ref('hlc/skus/' + sku.id).set(sku)
db.ref('hlc/cropEntries/' + entry.id).set(entry)
db.ref('hlc/producers/' + producer.id).set(producer)

// Delete one item
db.ref('hlc/logs/' + id).remove()
db.ref('hlc/skus/' + id).remove()
db.ref('hlc/cropEntries/' + id).remove()
db.ref('hlc/producers/' + id).remove()
```

`saveData()` is safe ONLY for flat config keys: `settings`, `settings250g`,
`pkgBreakdown1kg`, `pkgBreakdown250g`, `bulkTierDiscounts`, `bulkTierDiscounts250`.

---

### Never call Firebase writes during initialisation
- `onAuthStateChanged` must `await initFromCloud()` before any writes
- The overlay must hide AFTER `initFromCloud()` completes — not before
- `cloudReady` flag must be `true` before any `saveData()` call fires
- `attachFirebaseListeners()` must only run once (guarded by `listenersAttached`)
- `onAuthStateChanged` full init must only run once (guarded by `appInitialised`)

---

### Pre-commit checklist — verify before every `git push`

1. Does any new code call `saveData()` with a key in `['logs','skus','cropEntries','producers']`?
   → If yes, STOP. Rewrite as individual `db.ref(.../id).set()` instead.

2. Does any new code call `db.ref('hlc/logs').set(...)` or `.remove()` on a collection path?
   → If yes, STOP. Only individual item paths are allowed.

3. Does any new code run before `await initFromCloud()` completes and write to Firebase?
   → If yes, STOP. Writes must happen only after cloudReady = true.

4. Does any new event listener or callback call `attachFirebaseListeners()` or `initFromCloud()`?
   → If yes, verify the `listenersAttached` / `appInitialised` guards still work.

5. Is there any `logs = []` or `skus = []` assignment outside of a Firebase listener or explicit user delete action?
   → If yes, STOP. Collections should only be emptied when Firebase confirms they are empty.

---

## Architecture summary

| Collection | Firebase path | Write pattern |
|---|---|---|
| History logs | `hlc/logs/[id]` | Individual `.set()` / `.remove()` |
| Coffee SKUs | `hlc/skus/[id]` | Individual `.set()` / `.remove()` |
| Crop entries | `hlc/cropEntries/[id]` | Individual `.set()` / `.remove()` |
| Producers | `hlc/producers/[id]` | Individual `.set()` / `.remove()` |
| Green Varietals | `hlc/greenVarietals/[id]` | Individual `.set()` / `.remove()` |
| Green Varietal Prices | `hlc/greenVarietalPrices/[id]` | Individual `.set()` / `.remove()` |
| Settings 1kg | `hlc/settings` | `saveData()` full object replace ✅ |
| Settings 250g | `hlc/settings250g` | `saveData()` full object replace ✅ |
| Pkg breakdown | `hlc/pkgBreakdown1kg/250g` | `saveData()` full object replace ✅ |
| Bulk discounts | `hlc/bulkTierDiscounts/250` | `saveData()` full object replace ✅ |

## Known safe functions
- `renderLogs()` — read-only, no Firebase writes
- `renderSkuTable()` — read-only
- `calculate()` — read-only
- `loadSettings()` — read-only
- `updateSkuSelect()` — read-only
