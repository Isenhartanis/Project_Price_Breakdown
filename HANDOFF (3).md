# Project Breakdown — handoff

Paste this into a new chat together with `projectbreakdown.html`. It replaces the old
conversations, so none of that history needs re-reading. Last updated after the phone
work: the card deck, touch input, the left File and Page drawers, the History and Wage
drawers, and a long run of mobile fixes. Current build stamp: `2026-09-24 12:16 UTC`.

---

## What it is

A construction estimating workbook for sawcutting / concrete work — takeoffs, options,
unit pricing. One self-contained HTML file, no build step, no dependencies. Open it in a
browser and it runs. ~13,200 lines: `<style>`, markup, then one big IIFE of ~330 functions.
The only outside requests are Google Fonts (IBM Plex, Fira Sans), each with a fallback.

## How to work on it

- **Edit in place.** Never rewrite the file wholesale.
- **Parse-check after every edit:**
  `new Function(html.split('<script>')[1].split('</script>')[0].replace('load().then(showView);',''))`
- **After anything structural, also check:** no duplicate ids, balanced markup, every
  `getElementById` target exists, every function called is defined, and column counts
  still agree. Bulk edits have silently deleted whole functions before.
- **Always assert before replacing.** A `str_replace` that quietly no-matches is the most
  common way this file breaks.
- **Use a headless browser harness** (Playwright, when the container has it). Most of the layout bugs
  below were found by measuring, not by reading. What makes it work:
  - Stub `window.storage` in `addInitScript` with
    `get: k => Promise.resolve(mem[k] == null ? null : {value: mem[k]})`, and load the page
    with `goto('file://…')`. `setContent` skips init scripts, so the stub never lands.
  - Seed the whole tree, `lists → companies → projects → takeoffs[{sheets, active}]`, with
    `activeList` and `activeTakeoff` set. The open option comes from the takeoff's own
    `active`, so a bare `book.sheets` opens on option 1.
  - Serve fonts: route `fonts.googleapis.com` to a CSS of `@font-face` rules and
    `fonts.gstatic.com` to local woff2 files from `@fontsource/*` (npm). Without it every
    screenshot falls back to system fonts.
  - To prove a change leaves a theme alone, pixel-diff a screenshot of the old file against
    the new one. A single stray pixel at the Fee % input is the caret blinking.
- **The last session used jsdom** instead (`npm install jsdom`), loading the page with
  `runScripts:'outside-only'` and `w.eval(script)`. It proves behaviour and computed
  classes, never appearance, and it has gaps that bit repeatedly. Check these from the
  stylesheet text rather than `getComputedStyle`: `border-*` shorthands containing `var()`
  are dropped (write them longhand), pseudo-element styles are not computed, a `padding`
  shorthand is not resolved against a more specific longhand, and its cascade has reported
  a Medieval rule on a body that was not Medieval. Useful stubs:
  `Object.defineProperty(w,'innerWidth',{value:390})` for a phone, a
  `getBoundingClientRect` override that gives each `tr` a slot for the drag maths,
  `w.matchMedia` for `(hover:none)`, and a fake `w.Date.now` so two taps in one test are two
  gestures, as they always are for a thumb.
- **Nothing here emulates the user's phone** (Android, the file opened in the Claude app's
  viewer, Medieval theme). Many rounds were lost shipping theories about touch. Two
  diagnostics are built in, so ask for them before guessing: the **This build** block at the
  foot of the `?` menu (`const BUILD`, plus the live layout and screen size — **bump
  `BUILD` on every ship** so a stale copy shows at once), and the **Tap test** pad at the
  top of the Page drawer, which prints every event one tap produced and whatever element
  sits on top of it. The user's screenshots have found more of these bugs than the tests.
- **After `renderAll()` every row element is new.** Code and tests that hold a row across
  anything that redraws go stale without a sound. Re-query.
- The user wants token use kept down: batch related changes, and they may say
  **"quiet mode"** to suppress verification output.

---

## Data model

One `book` object, saved to `window.storage` under `breakdown:book` (debounced), also
exportable as JSON.

```
book
├── lists[]                    named project lists
│   └── companies[]
│       └── projects[]         name, address, contacts, status, notes, custom{}
│           └── takeoffs[]     each owns its own workbook
│               └── sheets[]   the options / numbered tabs
├── activeList, activeTakeoff, active (sheet id), view, zoom, theme, dark
├── libs[]                     named template libraries
│   └── templates{ items, sections, scopes, parts, constructs, services }, folders[]
├── catalog{ subtypes[], folders[] }
├── wageGroups[]               wage calc
├── notes[]                    sticky notes: {id, title, text, open, x, y, w, h}
├── statuses[], customFields{company,project,takeoff}, filterFields[], fieldPicks{}
├── libW, projW, sideW, wageW, edW{}, picsOn, gameCard, summaryNotes
└── projSort, histSort, dense, headShut, deck       panel sort orders and phone layout
```

`book.sheets` is a **live pointer** to the active takeoff's `sheets` array, set by
`activeTakeoff()`. `state` is the sheet on screen.

**A sheet:** `{ id, num, title, note, notes, color, rows[], fees[], units[], widths{},
hiddenCols[], roundStep, roundTotal, loadCalc[], loadOn }`

**A row:** `{ id, type:'item'|'section'|'sectionEnd', kind, name, note, pics[], picsOpen,
count, time, days, cost, markup, meta, parts[], uom, notes, notesLocked, deckShut }`

`deckShut` is a line folded in the phone's card deck; a section's own fold is `collapsed`.
Load calc rows carry an `id` too (older rows get one in `loadRowsFor`).

`kind`: `labor`, `equip`, `material`, `part`, `service`, `construct`, `none` (blank
spacer), plus `hybrid`. **Adding a kind means updating four places:** `KINDS`, `KIND_KEYS`,
`iconSvg()`, and a `--kind` CSS colour variable. A missing `KINDS` entry has bitten us
before. The row type menu iterates `PICK_KINDS`, not `KINDS`, so a derived kind stays out
of it.

**Main types and hybrid.** `labor`, `equip` and `material` are the main types
(`MAIN_KINDS`); parts, services and constructs are built out of them. `derivedKind(row)`
reads what is inside a service or construct, walking nested `items[]` and `parts[]`: one
main type reads as that type, more than one reads as `hybrid`, nothing readable leaves the
metatype alone. It is never picked by hand. The row's type icon, its grip colour and its
hover label all follow it, and `recalc` repaints them so a card edit shows up at once.

---

## The maths (`calc`)

```
base     = count × time × days × derivedCost(row)
subtotal = base × (1 + markup/100)
fee_n    = subtotal × fee_n.pct/100
grand    = subtotal + Σ fees
U/P_n    = grand ÷ unit_n.qty
Round    = roundTo(grand, sheet.roundStep)      display only, never feeds the maths
```

`derivedCost(row)`:

- **construct** → `constructCost()` — Σ children `ct×t×d×cost×(1+mk/100)`
- **service** → `serviceCost()` — base rate **plus** `serviceExtra()`, the same child sum
  but every child field may be a formula
- everything else → the typed `cost`

For construct and service the Cost cell is **read-only and derived**; edit those in the
card (card icon on the row).

**Rounding is always a display, never a change to a number.** Per-line rounding is a
column with the step chosen in its header ($1/$10/$100/$1,000); it does not change
Subtotal, Fee or Grand, and sections and the totals row sum the rounded amounts.
`roundTotal` (Page drawer) works the same way: the grand total keeps its true figure and
the rounded one appears on a tab hanging off the bottom of the table. It feeds nothing —
not the unit prices, not `sheetTotals`, which reports it separately as `grandRounded`.

That tab is `#tGrandLip`, absolute, a direct child of `#sheetCard` so nothing clips it.
`placeGrandTab()` sets its top from the grand cell's bottom edge and its left/width from
that cell's horizontal span clamped to the scroller, on every recalc, layout and scroll. It
therefore follows the totals row whether that row is pinned to the bottom of a long sheet
or sitting part way down a short one, and hides when the grand column scrolls out of view.
Two 19px spacers keep clear room for it: `.grand-gap` under the table inside the scroller,
and `#grandRail` between `.scroll` and `.bar`. It cannot live inside the scroller, and the
scroller cannot carry bottom padding — see the conventions.

### Service formulas

Child fields take a number or an expression in `CT`, `T`, `D` — the service's own amounts.
Supports `ROUND`, `ROUNDUP`, `ROUNDDOWN`, `MIN`, `MAX`, `ABS`, arithmetic, parentheses.
`ROUND(T/2, 0)` with T=9 → 5, rounding **half away from zero** like Excel, not like JS.
Junk returns `NaN` and falls back to a default. See `evalFormula()`.

**Known wrinkle:** add-ons sit inside the derived cost, so `Ct × T × D` multiplies them
too — a formula referencing `T` scales twice. That was a deliberate request ("sum like
constructs"). Mention it if the user hits it.

---

## Layout

Four views via `book.view`, switched by `showView()`:

| view | what |
|---|---|
| `sheet` | the numbered option tabs |
| `summary` | all options, collapsible project/takeoff blocks, notes |
| `scopes` | gear tab — every option's scope, colour, drag to reorder |
| `load` | disposal volume calculator |

The wage calc used to be a fifth view and is a drawer now; `adoptBook` turns a saved
`view:'wage'` into `sheet` so an older file does not open on a blank screen.

The pricing table is `#sheetTable`. Selectors that mean "the pricing table" must say so:
`#sheetCard` holds a second table when the load calc is folded in, so `#sheetCard thead th`
would sweep up both.

### The drawers and their tabs

Eight slide-out panels on two rails, plus the editor card:

- **Right rail** (`#tabRail`): **LIBRARY** (templates, one folder level at a time),
  **CATALOG** (subtype definitions), **NOTES** (the note list; opening one puts a sticky on
  the screen), **HISTORY** (unit price history, below) and **WAGE** (the wage calc).
- **Left rail** (`#leftRail`): **PROJECTS** (companies → projects → takeoffs), **FILE**
  (Save file, Load file, Excel, Print) and **PAGE** (Tap test, + Item / + Section,
  + Load calc, Collapse all detail, pictures, Layout, Magnify, Round total, Delete option,
  Reset sheet).
- **Editor card** — floats beside the relevant panel; `ed-left` for project cards.

Each rail's drawers are mutually exclusive: right through `swapDrawer()`, left through
`openLeft()` / `swapLeft()`. Every tab toggles its own drawer and stays visible while it is
open — parked on the drawer edge, larger and amber. Each rail is a fixed flex column centred
on its edge, so the open tab growing pushes its neighbours instead of covering them, and
hiding one (the library tab is sheet-only) closes the gap. **Never position these tabs
individually.** A rail slides as a whole: right to `var(--libW)` (`var(--wageW)` for Wage),
left to `var(--projW)` or `var(--sideW)`.

Two moments need the transition suppressed, both via `no-slide` on the body:

- **Handing over** from one drawer to another (`swapDrawer()`, one frame). Without it the
  outgoing panel slides out while the incoming one slides in and the edge looks empty for a
  fifth of a second, under tabs that are deliberately not moving.
- **Resizing** a drawer by dragging its edge (for the duration of the drag). Without it the
  rail eases after the edge and looks like it is chasing it.

Body gutters are `--tabW` wide so the closed tabs sit just clear of the cards. Panel widths
live in `--libW`, `--projW`, `--sideW` (File, Page) and `--wageW`, and **a panel's width and
its rail's offset must be the same variable** — never a hardcoded px, and nothing may resize
a panel behind its variable (a media query that did left the tab parked short of the panel
edge). `fitPanels()` clamps all four to `innerWidth - 56` on load and resize, from the
stored `book.*W` or the defaults 340 / 320 / 232 / 520; the wage panel is also capped at
`min(var(--wageW), 100vw - 46px)`. All panels resize by dragging their inner edge
(`addPanelGrabs()`; File and Page go down to 170px, the rest to 260); the editor card too,
remembering a width per mode in `edW`.

**On a narrow screen** (≤720px) an open drawer takes the screen instead of squeezing the
card, and the **opposite** rail hides, since an open panel reaches most of the way across
and both rails would end up on one edge. An earlier version sent the left rail to `right:0`
instead, which interleaved PROJECTS/FILE/PAGE with LIBRARY/HISTORY. Don't.

**Project list options** behaves like a tab: a second press closes the card, and
`markListOpts()` keeps the button lit only while `edMode` is `listopts`.

**The Projects drawer.** A company is a framed card (`.p-co`, carrying `dataset.company`)
with a banded header, each project a panel inside it (`.p-pr`), and its takeoffs hang off a
rule (`.p-tks`). The A–Z button cycles A–Z → Z–A → Manual (`book.projSort`, `byName()`) at
all three levels. Because sorting moves the cards, a project's drop target is found by
`dataset.company`, never by the card's position.

### History

The HISTORY drawer averages unit prices across **every takeoff in every list**.
`eachSheetEverywhere()` walks the book directly, not through `projectTree()`, which
retargets `book.companies`. `histGroups()` keys each option by `histKey()` of its scope's
first line plus the unit label, so spelling and punctuation variants group together and
$/SF never averages with $/LF. There is one entry per unit column that has both a quantity
and a price, and a group needs **two or more** entries to show. The headline is
**quantity-weighted** (Σ price × qty ÷ Σ qty, marked `wtd`). The detail line adds low,
high, spread, the **flat** mean and the total quantity, and each entry shows its share of
the weight. **Items** previews that option's lines; **Open** jumps to it, switching lists if
it has to (`histGo()`). Sort cycles By scope / Dearest first / Most used (`book.histSort`).

### Help

The round `?` at the far right of the top rail, before the saved marker. The option tabs
live in a strip of their own, `#railTabs`, which `renderTabs()` clears and refills
wholesale; the help button and `#status` sit outside it in `#rail`, so nothing a render
creates can pile up beside them (the old rule about inserting before `railEnd()` no longer
applies). When the strip overflows, `railFit()` switches on sideways scrolling, gives each
option tab a plain `title` because the scroller would clip the styled `.tab-tip`, and
scrolls the current option into view. A second press on `?` closes the menu. `openHelp()`
builds it from the `HELP` table and ends with the **This build** block (build stamp,
Layout, Screen). New explanations belong in `HELP`, not in the bar.

### Appearance and magnify

**Themes** are Light, Dark and Medieval, kept in `book.theme` and applied by
`applyTheme()`. **Medieval is the default.** `themeName()` uses `book.theme` when set;
otherwise a file from before themes had names keeps its choice (`dark:true` is Dark,
`dark:false` was a deliberate pick of Light), and a book with neither opens Medieval. The
script adds `dark medieval` to the body as soon as it runs, so the default paints before
the saved book loads, and `applyTheme()` corrects it if the saved choice differs.
`setTheme()` keeps `book.dark` true for any non-light theme so an older build still opens
it dark. The Theme buttons (`lightMode`, `darkMode`, `medievalMode`) sit in **Project list
options** for now, parked there until there is a real options screen, so `applyTheme()`
has to cope with them not existing. A theme change refits the table, since Medieval's frame
changes the room it has.

Printing always uses the light colours: `beforeprint` strips `dark` and `medieval` from the
body and `afterprint` puts them back. The print pages share the screen's variables, so
without that a dark theme printed light text on white paper.

**Type** is three variables, `--f-sans`, `--f-cond` and `--f-mono`. No rule names a font
family directly, so a theme swaps the lettering by redefining those three.

**Dark** is `body.dark`. Every light surface in the stylesheet is a variable (`--s1`…`--s59`,
plus the named ones) and `body.dark` redefines them all, so a new rule should reach for an
existing surface variable rather than a fresh hex. Two things deliberately stay out of it:
the **game card** and the **sticky notes** (`.gcard`, `.gc-*`, `.sticky*`) keep their own
parchment and paper in every theme. Watch for colours that mean opposite things in the two
modes: `--slate` is a text colour and `--slate-bg` the navy surfaces that used to share it,
and tooltips use `--tip-bg` rather than `--ink` or they invert into light-on-light.

**Medieval** is `body.dark.medieval`: `applyTheme()` sets both classes, so every dark rule
still applies (drawers, editor, menus) and one block at the end of the stylesheet, starting
`/* ---------- Medieval theme ----------` just before the `max-width:640px` media query,
restyles the rest. It follows a reference picture the user supplied: speckled stone for the
page, tabs, card heads and bars; a riveted iron frame round each card's scroller; parchment
table bodies with striped item rows; cracked stone section rows; gold rails and corner
brackets on the Grand column; copper buttons to add things, slate for file actions, bronze
for Delete option and Reset sheet; Fira Sans with tabular figures. Type colours stay, in
earthier tones: lighter on dark surfaces, darker on paper.

- **Textures** are inline SVG data URIs in `--tx-*` variables on `body.medieval`: `grain`
  and `mottle` (paper), `stone`, `speck`, `speckd` (stone), `cracks`, `cracks2`, `cracks3`
  (section slabs), `iron` (the frame's `border-image`), `cornL`/`cornR` (Grand brackets).
  Each is `feTurbulence` → `feColorMatrix` that turns one channel into alpha over a fixed
  colour. To change one, `urllib.parse.unquote` it, edit, and re-encode with `#` written
  raw; a hand-written `%23` gets encoded twice and the image silently fails. Crack lines
  come from the turbulence **alpha** channel (the colour channels are clamped where alpha
  is low and draw pools), with a gain: a lower frequency needs a higher gain or the lines
  thicken into blotches.
- The iron frame is a **border** on `.card > .scroll`, never padding. A border sits outside
  the scrollport, so the pinned header and totals stay flush (see the sticky-row notes).
- Parchment is a **variable scope**, not a pile of colours: `#sheetTable tbody/tfoot`,
  `.sum-table` bodies, `.calc-table` and `.grand-lip` redefine `--ink`, `--rule*`, `--slate`
  and the type colours for dark ink on paper, and `tr.section-head` scopes them back to
  light for its stone.
- Paper and stone paint on the **row** (`tr`), which Chrome draws unbroken across the
  cells; cells add only translucent tints (calc, grand, round, hover, focus). Positioned
  cells (`td.mk`, `td.has-tip`) restart the texture, which their borders hide. **Sticky
  cells do not carry the row's paint**, so the pinned Item column and every totals cell
  paint their own paper, or rows show through them.
- Section rows are **seven stone slabs**, picked by `:nth-child(7n+k of .section-head)`,
  each with its own tone, crack texture and `background-position` for all four layers, so
  neighbouring rows never match, collapsed or not. The sticky Item cell repeats the same
  values. Hover is an inset `box-shadow` wash rather than an extra background layer, which
  would throw the position list out of step and shift the pattern.
- Section subtotal rows are a darker, warmer band (`#D8CDB5`) with a rule above and a heavy
  rule below, so they stand apart from the striped lines.
- Stripes use `:nth-child(even of [data-type="item"])`, so they count across the whole body
  rather than restarting per section.
- The sheet bar now holds only the chips that bring a hidden column back (`#hiddenCols`)
  and hides itself (`.bar-empty`) when there are none. Everything else it carried moved to
  the File and Page drawers; `+ Add item` and `+ Add section` went, since the Item column
  header, the deck's add bar and the Page drawer all add lines.
- Wage and load calc: stone heads on wage groups and load blocks, stone section bands, tan
  subtotal rows and a gold-framed grand total. The load calc's own section rows are still
  parchment.

**Magnify** is one knob for the whole app: `book.zoom` (100% to 200% in small steps)
applied as CSS `zoom` on `<body>` by `applyZoom()`, from the select in the Page drawer.
Everything scales together, edge tabs included.

### The top of the card

Two controls at the head's top right, both saved. **Compact** (`book.dense` →
`body.dense`) tightens the option tabs, the head, every thead and the bar. The caret beside
it (`book.headShut` → `body.head-shut`) folds the scope of work down to one line,
`#headPeek`, which reopens the head with the caret in the title when tapped. Both go
through `applyDensity()`. The fold is scoped to `#sheetCard`; Compact is global.

### Sticky notes

They live in `#stickies`, a fixed full-screen layer with `pointer-events:none` and the
notes themselves `auto`. A note only closes on its own x, and resizes from its corner in
both directions (`resize:both`, with a `ResizeObserver` storing `w`/`h`). The whole head is
the drag handle, including the title box, so the drag waits for 4px of movement before it
starts and a plain click still lands in the input.

### The library

It shows **one folder level at a time**, the way a file window does: `libPath` holds the
folder you are standing in, a breadcrumb trail leads back up, and `renderTree` draws the
folders directly under `libPath` followed by that folder's own templates. The trail buttons
carry `.folder-head` too, so they are drop targets like any folder. Folders drag onto other
folders (`startFolderDrag` / `moveFolder`, which rewrites every affected path). The catalog
still uses the old collapsing tree and its own `folderShut`.

Every card says what it is: `tplTag()` gives the chip text — an `item` shows its own type,
a `scope` reads **Option** because that is what it makes — and `t-<kind>` on the card sets
`--tpl-ink` for the chip and the stripe down its left edge.

---

## Phone: the card deck

Under 760px (`DECK_AT`) the pricing sheet restyles itself as one card per line
(`body.cards`). It is **the same table**, not a second renderer: every input, money div
and handler is the table's own, so `recalc` paints the deck without knowing it exists. The
Page drawer's **Layout** pick (`book.deck`: auto / cards / table) forces it either way, and
`applyDeck()` runs from `showView()` and on resize.

- **Labels.** `labelCells()` copies each column's name onto its cells as `data-label`,
  shown through `td::before`. It reads the live header (`.h-lab`'s `dataset.full`, since
  `fitLabels()` abbreviates narrow headers) and walks by **colspan**, so section rows map
  correctly. Runs on render, and from `layout()` while cards are on.
- **Layout.** In card mode the table, `tbody` and `tr` are block or flex, so `display:flex`
  on a `td` is safe **there and only there**. `layout()` returns early without setting pixel
  widths, and `placeGrandTab()` shows the rounded total as a plain line.
- **Scrolling.** Card mode hands scrolling to the page: `body.cards` is `height:auto` and
  `overflow:visible`, and the card and scroller stop clipping. `overflow:hidden` on body
  propagates to the viewport, so leaving it on locked the page. The shell uses `100dvh`
  where supported, or the foot of the card sits under the phone's address bar. The option
  tabs stay sticky at the top while the deck scrolls.
- **A card.** The head is the name on the left and the line total on the right
  (`td.c-item` order 0 with `flex:1 1 0`; `td.calc.grand` order 1, no Grand label), then a
  forced line break (`tbody tr::after`: order 1, 100% wide, zero height), then the typed
  fields (order 2, label over field, two or three to a row) and the worked figures (order
  3, one per line, hairline between). **Without that break the zero-basis head collapses**
  and Count rides up beside the total. The type is a stripe down the left edge
  (`--row-kind`, set in `paintKind()`). The grip, type picker, duplicate and pictures
  buttons are hidden; note and card show at 55%, because `.note-btn` is `opacity:0` until
  hover, a phone never hovers, and the invisible buttons were eating the name's width.
  Changing a line's type on a phone therefore means the card editor or Table layout.
- **Folding.** The caret (`.deck-caret`) flips `row.deckShut`, and `paintCardShut(tr, row)`
  sets `card-shut` and turns the caret. It must run **after the row is assembled**: it
  finds the caret with `tr.querySelector`, and called earlier the arrow never turned on a
  re-render. A folded card shows its name and total only. **Fold all** (`#deckFold`) sets
  `deckShut` on every line and `collapsed` on every section, then calls `renderAll()`,
  drawing from the data rather than repainting through cached refs. `anyCardOpen()` counts
  open sections too, and `syncFoldBtn()` keeps the Fold all / Unfold all label honest on
  every render.
- **Sections.** The head is a band: caret, name, + Item and the section total. The five
  summary figures appear once, on the foot. On section rows `td.mk` is a figure laid out in
  a row, not a field, or its `$` drops to a line of its own. A collapsed section hides its
  cards **and its foot** through `tr.row-hidden`, which ties with the card row rules on
  specificity and so sits after them.
- **Totals.** `tfoot` is kept out of every card-head rule (they all say `tbody`) and styled
  like a section subtotal: TOTAL and the figure on one line (`tfoot td.grand`, which has no
  `calc` class), its own `tfoot tr::after` break, then the breakdown with hairlines.
- **Add bar.** `#deckAdd` sits **after** the table, where a new line lands: + Item,
  + Section and Fold all, under a "Press and hold a card to move it" hint. `addItemRow()` and
  `addSectionRows()` call `revealRow()`, so the new card scrolls into view before its name
  takes focus.
- **Moving a card** is a long press anywhere that is not a control (Touch input, below). A
  shrunken copy follows the finger and the row turns into a dashed slot.

### Card-mode rules that have each broken something

- Every card rule carries `#sheetTable`. An ID outranks any number of classes, so a card
  rule without one (the first fold rules) silently loses.
- **Medieval rules come later in the stylesheet at equal specificity** and win on order.
  Where a card rule has to beat one, name both classes, `body.medieval.cards …`. The totals
  row kept its 3px double rules until that.
- A cell that anchors an absolutely positioned child must stay positioned, hence
  `body.cards #sheetTable tbody td{position:relative}`. When card mode made cells static,
  `td.mk .money` (`position:absolute`, `top/bottom/right:0`, `min-width:100%`, `z-index:2`)
  climbed to `#sheetCard` and laid a transparent sheet over the whole deck: nothing could be
  tapped, and a lone `$` sat at the card's edge. `.mk-cash` did the same. Both are switched
  off in card mode.
- `mk-cash-on` was set by `mouseenter` on the markup total, which a tap fires with no
  `mouseleave` after it, so it latched on for good. It now sets only on `(hover:hover)`.
- Scope card rules to `tbody` or they reach the totals row.

## Touch input

Hard won. Every rule here was a shipped bug on the user's phone.

- **`tapBind(el, fn)`** is the one handler for button taps: the deck's add bar, the Item
  header's add buttons and the Page drawer's. A gesture opens on its down event and closes
  on its click. It arms on `touchstart`, `pointerdown` or `mousedown`, acts on `touchend` or
  `pointerup`, and swallows the click that follows as the gesture's tail (`sawDown`); a
  click with no down before it (keyboard) acts on its own. **No timing window**: a 700ms
  rule made any held tap fire twice. **No geometry check**: it silently rejected real taps.
  A `pointercancel` or `touchcancel` still counts as a tap if the finger moved under 12px,
  because near the foot of a long page the browser takes the gesture and sends no release.
  A `mousedown` within 900ms of a touch is the phone **replaying the gesture as mouse
  events** and is ignored: it re-armed the gesture, the click fired the action again, and
  Fold all folded and then unfolded.
- **The fold caret** is delegated on `#body` (`caretTaps`) and acts on the **press**. The
  gesture stays open (`openTap`) until its click closes it; if the click never comes, a
  press more than 450ms later opens a new one.
- **Long press** (`PRESS_MS` 400, 12px slop, highlight after `ARM_AT` 140ms so a quick tap
  doesn't flash) starts on anything except `input, textarea, select, button, a,
  .pic-strip`. It runs on **touch events** on `#body` (`touchStart` / `touchMove` /
  `touchEnd`): a card has `touch-action:auto`, and the pointer-driven first version got
  `pointercancel` the moment a finger twitched. The press fires while the finger is still,
  so preventing the next `touchmove` claims the rest of the gesture and no scroll starts. A
  pointer-driven twin (`ptrDown` / `ptrMove`) covers browsers that send only pointer events;
  whichever arms first wins. It works in either layout.
- **Dragging** shares `beginDrag(tr, y, x)` between the grip (`startDrag`) and the long
  press. `startDrag` attaches its window listeners **before** `setPointerCapture`, which is
  wrapped: by the time a long press fires the pointer may be gone, capture throws
  `NotFoundError`, and that used to skip the listeners and leave the drag unfinishable. The
  column resizer is guarded the same way.
- **The drag copy** lives in a second tbody, `#ghostBody`: inside `#sheetTable`, so every
  card rule applies to it, and outside `#body`, so no ordering code sees it (`liftGhost`,
  `moveGhost`, `dropGhost`). It is `pointer-events:none`, shrinks and tilts while held, and
  slides into the slot on release.
- **The sort lock.** While `tbody.sorting` is set, fields, type buttons and delete buttons
  are `pointer-events:none`, so a mouse drag can't select text. A stuck lock made the whole
  sheet untappable. `endDrag()` calls `releaseSortLock()` **first**; the long press
  releases in a `finally`; `healSortLock()`, on any `touchstart` or `pointerdown` in the
  capture phase, clears a lock that outlived its drag and also sweeps a latched
  `mk-cash-on` and any orphaned drag copy; and on `(hover:none)` the lock never covers
  fields at all.
- **`touch-action`.** Buttons, inputs and tabs are `manipulation`, which removes the
  double-tap-zoom delay. Every drag handle — `.grip`, `.lc-grip`, `.col-resizer`,
  `.sow-num`, `.tpl`, `.panel-grab`, `.folder-head`, the option tabs — must be `none`, set by
  a rule **after** the blanket one. The blanket rule once switched off every drag on touch.
- `html{-webkit-tap-highlight-color:transparent}`, and a `(hover:none)` block that stops row
  hover sticking after a tap. Both read to the user as "colours flash when I tap".

---

## Editor modes

One panel, eleven modes: `item`, `part`, `construct`, `service`, `section`, `scope`,
`subtype`, `company`, `project`, `takeoff`, `listopts`. `openEditor(mode, obj)` fills a
draft, `renderEditor()` dispatches to `render*Ed(box)`, and Save has a branch per mode.

The name field is outlined and tinted so it never reads as the detail box under it, the
number fields carry the same tint, and `miniGrid()` adds a **UoM** box after Cost and
Markup.

The item card ends with a **Notes** block (`notesBlock`, `d.notes`) — free-form, separate
from the one-line `note` under the name, and lockable. It uses `blockHead(d, label,
keyLock, keyOpen)`, the same collapsible-and-lockable head the Catalog and Folder blocks
use; locked means the textarea goes read-only, not hidden. `uom`, `notes` and `notesLocked`
all ride back to the row on save. Only the item editor has the Notes block so far.

Item templates also have a **game-card view** (toggle at the foot of the card), laid out
like a Hearthstone card: main type orb at the far top left (showing the derived type, so a
mixed construct reads Hybrid), meta type badge top right, art, the name on a banner slung
under it, description, then the gold cost plate with the UoM beside it, both bottom left.

## Calculators

- **Load calc** (`sh.loadCalc[]`, **per option**) — SF = Ct×L×W, CY = SF×(D/12)/27 with
  **D in inches**, Fluff = CY×pct, Total = CY+Fluff, Loads = Total ÷ L/T. Section rows roll
  up the lines beneath them positionally, like the main sheet, band the group, and collapse
  it from their caret (`r.shut`). `buildLoadTable(table, sheet, again)` renders one option's
  table and `loadBlock()` wraps it with a heading and its own add buttons, so the same code
  serves both places: the Load Calc tab stacks a block per option with an all-options total
  under them, and the sheet folds one block in under the pricing table when `sh.loadOn` is
  set (the `+ Load calc` button in the Page drawer). In the tab each block folds from its head (`sh.loadShut`,
  the whole head is the toggle, the caret is only its keyboard handle) and a folded block
  shows its SF, CY, total CY and loads in the head instead of the table. **Collapse all**
  (`#loadFoldAll`, in the tab's bar) folds every option, reads **Expand all** once they are
  all folded, and `renderLoad()` keeps its label in step. The block folded into a sheet has
  no head, so it never folds. A `loadCalc` left on a takeoff by an older file
  migrates onto its first option in `adoptBook`.

  Lines and whole section bands drag by their grip (`lcGrip`, `startLoadDrag`; a section
  carries its lines through `lcBlock`), Alt+arrow nudges one slot (`nudgeLoad`), and the
  new order is written back into `sh.loadCalc` **in place** by `syncLoadOrder()` so the
  live pointer stays valid. A row's × finds itself by identity, not by its index at build
  time, which went stale after any reorder. There is no edge-scroll while dragging here.
- **Wage calc** (`book.wageGroups[]`). Overtime is 1.5× and Double 2× the base rate,
  computed not editable. Fringes are flat dollar adds; burden percentages apply to the
  hourly subtotal (wage + fringes), matching the user's spreadsheet. Verified to the cent
  against their sheet: $46.48 base → totals $93.79 / $122.81 / $151.84.

  It lives in the **WAGE drawer** on the right (`#wagePanel`, `--wageW`, 520px by
  default), not a view, and `renderWage()` runs when the drawer opens. The table keeps a
  440px floor and the drawer body scrolls to it; the body needs `min-width:0`, or it grows
  to fit the table and pushes the panel off the screen. **Below 720px** it hides
  Overtime, Double and Notes so the table fits; the rate cells are tagged `w-rt`, `w-ot`,
  `w-dt` after the build, matching the header's `th` classes. The Optional sections
  heading takes a line of its own so its three buttons stack evenly.

  Below Total hourly rate come three **optional sections**, in this order. They are off for
  a new group and switched on from the group's *Optional sections* row at the bottom:

  | Section | Data | Works out |
  |---|---|---|
  | Direct cost items | `g.addons[]`, `g.addOn`, fold `addon` | flat dollars, same in all three columns |
  | Direct cost percentage | `g.dcs[]`, `g.dcOn`, fold `dc` | rates summed, on total + direct cost items |
  | Overhead & profit | `g.ops[]`, `g.opOn`, fold `op` | rates summed, on total + all direct costs |

  So O&P **compounds** the direct cost percentage, while the rates inside one section sit
  side by side (Profit is not figured on Overhead). Direct cost items keep their old
  add-on field names in the data. Switching a section on seeds one blank line; O&P seeds
  Overhead and Profit. Each percentage section ends with a subtotal labelled with its
  combined %. "Total with direct cost items" and "Total with direct costs" appear only when
  a later section uses them as its base, and **Grand total** only while any section is on.
  Checked: $81.41 total, $5 of items, 10% + 2% direct cost and 10% + 8% O&P give $96.78
  with direct costs and a $114.20 grand total; Overhead's 10% line reads $9.68.

  An optional section comes off from the × on its band. Its lines are kept for when it
  comes back, but while off it counts for nothing in `wageMath()`. Every section band
  (fringes, burden and the three optional ones) folds from its caret (`g.fold[key]`);
  folded lines are not built and their subtotal rows stay. Whole groups collapse from the
  caret on their header (`g.shut`) and then show their three grand totals in the header.
  `wageNorm()`, run from `wageGroups()`, brings older groups up to date: a single `g.op`
  becomes one "Overhead & profit" rate, and the on/off flags default to whether the group
  already has lines. `lineRows()` draws every section's lines and the paint step reads them
  back as `{o, cells}`.

**Both calculators must not re-render while typing.** They build the table once and repaint
only output cells (`paintLoad`, `paintWage`); a full render steals focus mid-entry. Full
renders happen only on add/remove.

## Drag and drop

Pointer-event based, ~5px movement threshold so clicks still work, and the drop decision is
read from the **release point**, not the last move event (a fast flick loses the last
move). Sheet rows reorder or drag into the Library to become templates; library cards drag
onto the sheet, into folders, or into the section/construct/service editors; tabs reorder
or drop on the Library as scope templates. On touch, a long press anywhere on a card picks
the line up and a copy follows the finger; see Touch input.

## Export

- **Excel** — hand-rolled `.xlsx`: CRC32 + stored-entry ZIP, styled sheet XML, live
  formulas with cached values. **Element order matters** — `calcPr` after `</sheets>`,
  `dimension` before `sheetViews`, valid DOS dates in the zip. Excel rejects the file
  otherwise and openpyxl won't notice, so validate ordering explicitly.
- **Print** — generates fresh HTML from the data rather than printing the DOM, so it
  reflects hidden columns. Menu offers this page or all options. Sheets print with the
  scope of work as the heading; never "Option #".
- Company and project lists export as their own spreadsheets from Project list options.
- **Library files** — the library switcher exports the active library as JSON and imports
  one back. Import accepts a library file or a whole workbook file, takes fresh ids all the
  way down, and lands each library in the library list rather than merging into the current
  one.

---

## Conventions worth keeping

- `confirm()` and `prompt()` are **blocked** in embedded frames. Use `confirmPop()` or the
  two-step arming pattern on destructive buttons.
- Never put `display:flex` on a `<td>` — it drops out of table layout. Use an inner wrapper.
  The one exception is card mode, where the table has stopped being a table.
- Hide-class selectors must match the element: `tr.row-hidden` will not hide a `div`.
- Column counts must agree across **six** row builders: header, item row, blank row,
  section head, section foot, totals. Fee columns insert before `thGrand`, unit columns
  before `thRound`, and an emptied group still renders a placeholder cell.
- Money cells use accounting layout — `$` left, figure right.
- Prose in the UI: plain, no em-dash asides, no "not X but Y".

### Sticky rows and the scrollers — the same bug three times

- Both scrollers are inset (`padding` on `.scroll`), so `layout()` and `sumLayout()` fit
  their table to `clientWidth` **minus that padding**. Fitting to `clientWidth` alone puts
  the last column past the right edge — that was "the right side gets cut off".
- That inset must never include `padding-top` **or `padding-bottom`**. A sticky row sticks
  at the padding edge, so an inset leaves a strip beyond it where rows show through as they
  scroll past: above the header with a top inset, below the totals row with a bottom one.
  Breathing room goes on the wrapper inside (`#sheetNotesWrap`), and anything that needs to
  sit under the table goes outside the scroller entirely.
- Anything in `tfoot` that overrides `position` drops out of the sticky footer and is left
  behind as the table scrolls. `td.mk-total` needs `position:sticky`, not `relative`, even
  though it is only positioned so the markup figures can grow leftward.
- A header `.tip` wider than its own column used to hang past the last column and widen the
  scroll area. `anchorTips()` runs after every layout and flips such a tip to `tip-r`.

### Columns and cells

- Round is hideable like Time and Days. Its header hides through the usual `HIDEABLE` path,
  while the body cells go through `#sheetTable.round-off`, so all six row builders keep
  emitting the cell and the column counts stay equal.
- `Object.keys(HIDEABLE).filter(colHidden)` handed `colHidden` the array index as its sheet
  argument, so only the first hideable column ever offered a chip to bring it back. It
  needs `filter(k => colHidden(k))`.
- An emptied fee or unit group renders a placeholder cell with no money div in it, so
  `money()` returns early on a missing element and every unit painter checks
  `state.units[i]` before reading it. Without that, deleting the last fee threw on recalc.

### Summary

- `#summaryCard .scroll` has no bottom padding, for the same sticky-row reason as the sheet:
  with it, rows showed through under the pinned All options row.
- Unit prices are a right-aligned list rather than wrapped chips: price in bold, the
  `/qty unit` part in a fixed-width column so the prices line up. `.s-none` is right-aligned
  only where it sits in that list.

### Editor blocks

- Every outlined block (`ed-namebox`, `ed-meta`, `ed-folder`, `ed-attach`) takes the
  `--slate` border. Attached pictures used `--rule-strong` and stood out.
- `.ed-folder` has a top margin but nothing below, so a block that follows it (Notes on the
  item card) gets its own `margin-top` through `.ed-folder + …`.

### Load calc

- The section caret is `.caret`, which is `display:flex` everywhere else. Inside the load
  table's name cell it must be `inline-flex`, or it drops onto its own line above the name
  and doubles the row height. `.lc-caret.shut` rotates it like the other carets.

### Pictures

`showPics` / `hidePics` are absolute: both clear every per-line `picsOpen` across the whole
takeoff, so nothing can come back open on its own, and a per-line toggle that lands on the
sheet setting deletes its override instead of storing it.

## Still open

- **Unconfirmed on the phone:** whether the last wage change (hiding Overtime and Double
  below 720px) finally fits. The user said "still too big" twice before it. If it still
  doesn't, the next step is stacking each labour group into label and value rows, the way
  the sheet deck works.
- A line's type can't be changed from a card on the phone; the type picker is hidden to
  give the name room. The card editor or Table layout still can. Offered to bring back a
  tappable type if wanted.
- The Markup line shows `$ —` when markup is zero. Offered `$ 0.00` instead.
- Card mode under the Medieval theme is only partly verified: jsdom can't render, so its
  look is known only from the user's screenshots.
- Opening the Projects drawer narrows the card but may not refit the pricing table.
  `openLeft()` now calls `layout()` 260ms after opening; whether that cured it is unchecked.
- The Wage calc's Print button prints the current sheet, not the wage groups. A wage
  printout was offered.
- Offered, not yet asked for: stone section rows in the Medieval load calc, a Collapse all
  button on the Wage calc, and figuring Profit on top of Overhead inside the O&P section.
- Takeoffs under a company named "Template" show a badge but have no behaviour attached.
- Duplicating a Chrome tab scrambles field values. Mitigated with `autocomplete="off"` on
  generated inputs and a re-render on `pageshow`; not fully solved, low priority.
- The load calculator is still a view tab and the wage calculator a drawer; the user may
  fold either into special part types later.
- The Notes block is on the item card only. Parts, constructs and services could take the
  same block if asked.
- The notes drawer has no resize grab of its own; `addPanelGrabs()` covers library,
  catalog, history, wage, projects, file and page.

## Tests to rebuild

The last session's checks lived in the container and are gone. About 390 of them, in these
suites, each worth recreating when that area is touched: **harness** (history maths and
grouping, projects sort, load calc drag), **smoke** (column counts across all six row
builders, every view, export), **drawers** (File and Page drawers, the tab strip),
**deck** (card mode, labels, Auto / Cards / Table), **scroll** (page scroll in card mode),
**tap** (`tapBind` under every event model, drag handles all `touch-action:none`),
**fold** (caret, Fold all, long press, the Android tap sequence including the mouse
replay), **press** (every part of a card that should and shouldn't start a drag),
**lock** (a stuck sort lock heals, a throwing drop still frees the sheet), **overlay**
(nothing absolutely positioned escapes its cell), **ghost** (the drag copy), **layout**
(card head, the break, the totals row), **section** (section band, collapse, Fold all with
sections), **gap** (panel widths and rail offsets at 320, 360, 390, 430 and 1280px),
**med** (folding under plain, dark, medieval and medieval dark), **capture**
(`setPointerCapture` throwing mid-drag) and **wage** (the drawer and its narrow layout).
