# Working in this repo

You are the **author** here. A separate Claude in a fresh chat
will review your work before it ships. Your job is to make that
review fast and trustworthy.

## Before claiming done on any change

Run a self-audit covering at minimum:

1. **Duplicate object keys.** Any time you add an entry to
   `TAG_DEFS`, `ORIGIN`, `CHARACTER`, `METRICS`, `HUE`, or any
   other object literal, search the file first:
   `grep -nE "^\s+(newkey):" index.html`. JS silently drops earlier
   duplicates — bugs become invisible.

2. **Coordinate collisions.** Element data has `(row, col)`
   pairs. Two elements with the same coordinates render on top of
   each other; the lower one disappears. Verify uniqueness:
   ```
   grep -oE "\[\s*\d+,'\w+','\w+',\d+,\d+" index.html | sort | uniq -c | sort -rn
   ```
   No row should show count > 1 for `(row,col)`.

3. **Smoke test.** Run the JS through a stub-DOM headless eval
   to catch syntax errors and init-time failures:
   ```
   node -e "const fs=require('fs');const h=fs.readFileSync('index.html','utf8');
   const m=h.match(/<script>([\s\S]*?)<\/script>/);
   const stub='const _e=()=>({innerHTML:\"\",textContent:\"\",value:\"\",style:{setProperty(){}},dataset:{},classList:{add(){},remove(){},toggle(){},contains(){return false}},appendChild(){},querySelector(){return _e()},querySelectorAll(){return[]},addEventListener(){},blur(){},focus(){},select(){},scrollIntoView(){}});const document={getElementById:_e,querySelectorAll:()=>[],querySelector:_e,addEventListener(){},createElement:_e,activeElement:null};const window={matchMedia:()=>({matches:false})};';
   new Function(stub+m[1])();console.log('OK');"
   ```

4. **Copy consistency.** If you change the dek, masthead, or any
   user-visible label, search the file for adjacent copy that
   may need to follow:
   - `<title>` tag
   - `<meta name="description">`
   - `<meta property="og:title">` and `<meta property="og:description">`
   - `<meta name="twitter:title">` and `<meta name="twitter:description">`
   These five must agree about what the product is and what its
   features are called.

5. **OG image text.** If a lens name, masthead, or marketing
   line changes, regenerate `og.png`. Text baked into the PNG
   goes stale silently and ships broken to every social share.

6. **Dead-code check.** After any deletion, grep the file for
   leftover references — DOM IDs, CSS classes, function calls
   that point at things you removed. The compiler won't help.

## Hand-off format

When you're done with a change set, end your final message with
this block (verbatim format — the reviewer Claude is looking for it):

```
## What changed
[one sentence]

## Why
[one sentence — what problem this solves]

## What I tested
- duplicate keys: [count, expected 0]
- coordinate collisions: [count, expected 0]
- smoke test: [PASS/FAIL]
- copy locations checked: [list of meta/title/og/twitter tags reviewed]
- og image: [regenerated / not needed because no copy changed]

## What I'm uncertain about
[anything you'd want a reviewer to scrutinize, or "nothing"]

## Files attached
- index.html  [byte size]
- og.png      [byte size, only if regenerated]
```

The "What I'm uncertain about" line is the most important one.
If you, the author, are honest about your blind spots, the reviewer
goes straight to those questions instead of doing redundant
verification. Be specific. "I'm not sure my new tag synonyms don't
overlap with existing ones" is useful. "Looks good to me" is not.

## Bug classes I have personally shipped in this repo

Adding here so future-author-Claude can recognize the pattern:

- **He/Ne coordinate collision.** Neon's row was 1, same as
  Helium. Both rendered at (1, 18). Helium was hidden under Neon
  for everyone who visited the page. Reddit user MagicPaul caught
  this within minutes of launch.

- **Duplicate `solar` and `rocket` keys in `TAG_DEFS`.** Adding
  new tag entries without grepping first. JS silently dropped
  the earlier definitions; the affected synonyms became unreachable.

- **Stale OG card text.** The `og.png` had the old lens names
  baked in for hours after the in-page lenses were renamed. Every
  social share would have shown the wrong lens list.

- **Note search false positives.** Substring matching on element
  notes returned Helium for the search "rock" because "rocket" is
  a substring of "rocket." Curated tags handle this; full-text
  search of prose does not.

If you suspect a similar pattern in current work, flag it in
"What I'm uncertain about."

## Out of scope for the author

The author Claude does not get to:
- Self-approve and push to main without an external review pass.
- Skip the audit "because the change is small."
- Mark "What I'm uncertain about" as "nothing" reflexively. If
  there really is nothing, write one sentence about why.

The reviewer can be invoked by Jeremy in a fresh claude.ai chat
in the Substance project. The author's hand-off format above is
designed to drop straight into that chat.
