# Workspace development instructions

This root owns the integration checkpoint, shared tooling and conventions for the
ETAF text applications. Every application is an independent Jujutsu/Git repository in
this directory, ignored by this repository and committed from its own checkout.

## Layout and providers

- Applications live at `./<name>` (`etaf-ncm`, `etaf-weread`, `etaf-md-preview`
  and `etaf-reader` today). Their framework providers
  (`etaf`, `etaf-ui`, `ebox`, `ecss`) live one directory further up in the
  `emacs-taf` workspace checkout: `../../etaf`, `../../etaf-ui`, `../../ebox`,
  `../../ecss`. A standalone clone instead keeps the providers beside the application.
- Application Makefiles therefore detect the provider root instead of hard-coding it:
  use `../` when `../etaf/etaf.el` exists, `../../` when `../../etaf/etaf.el` exists.
  Keep that detection when you copy tooling into a new application.
- `scripts/api-workspace.json` lists the applications this root checks out; each
  application keeps its own registry naming its providers (`ebox`, `ecss`, `etaf`,
  `etaf-ui`). `python3 scripts/check-api.py --workspace . --fetch-providers` clones
  the applications listed here; an application's
  `python3 scripts/check-api.py --root . --fetch-providers --providers-dir ../..`
  clones its providers beside this repository. Do not add provider or application
  directories to version control.
- `workspace.json` records each application URL, branch and the verified revision.
  Refresh the revision after publishing an application commit.
- The copied `scripts/check-repository.py` resolves the provider directory itself: a
  standalone clone keeps providers beside the application, while an application in this
  root keeps them two directories up. Preserve that fallback when refreshing the shared
  tooling, otherwise `make check` cannot resolve `etaf`.
- `docs/` accepts only the fixed document names (`manual.md`, `architecture.md` and
  their translations), so screenshots and other media live under `examples/` and the
  README links to them there (`examples/etaf-ncm.png`).

## Commands

```sh
make check                     # policy plus every application's structure, compile, API and acceptance checks
make -C etaf-ncm check         # one application
make setup-hooks               # staged structure and message gates in this root and every application
python3 scripts/check-repository.py
python3 -m unittest discover -s tests/tools -p 'test_*.py'
```

Look up provider Lisp from the enclosing workspace root with
`python3 scripts/elisp.py def SYMBOL...` and `python3 scripts/elisp.py run ...`; run
the application's own checks for application code.

## Measuring interaction stalls

A perceived stall (a hover highlight that cannot keep up, a drag that lags) is
measured rather than guessed:

- **Stall probe**: schedule a repeating 50 ms timer that records
  `(- (float-time) (+ start (* n 0.05)))` and report the maximum, the number of
  samples over 20 ms and over 100 ms.  A healthy idle application stays at
  1-6 ms; a whole-view publication per second reaches 200-500 ms.
- **Attribute without editing code**: force the fallback path at runtime by
  pointing the retained Host it uses at a missing symbol
  (`(setq etaf-ncm--clock-host 'ncm-missing-host)`), run the same probe, then
  restore it.  The two runs differ only in which update path is taken.
- **Compare with interleaved rounds and medians**: alternate the two paths in
  one session (e.g. 6 rounds of 3 `benchmark-run` iterations each); a single
  pass on a loaded GUI varies by more than 50%.
- **Timers only run while Emacs waits**, so drive them in batch with
  `sleep-for`, not `accept-process-output` (which can block forever when no
  subprocess exists), and never use `sit-for` there.
- Measure the *rendered* height of a canvas with `window-text-pixel-size`
  (it accounts for display properties) rather than counting buffer lines or
  characters: `posn-at-point` on a line boundary easily reports the wrong row.

## UI acceptance

Every interaction a person can perform is verified in a real graphical session
before a change is complete. Provider unit scenarios, batch rendering and
screenshots taken for a README do not constitute application acceptance.

- **Drive the controls, not the model API.** Send the documented keys through
the command loop (`execute-kbd-macro`) or activate a control through ETAF's own
focus (`etaf-focus` + RET). OS-level synthetic clicks do not reach Emacs text
areas, so the native activation path stands in for a click and the scenario
records which path each step used.
- **Mount through the application's offline harness for capture passes.** The
application's tests already script the client and ship fixtures, so a mount
paints a full frame immediately; a session against the real backend keeps the
canvas blank until its service answers and cannot yield a checkpoint. Close the
verification instance when the pass ends, including a failed one.
- **Verify against the signed-in account before claiming completion.** Scripted
scenarios are the deterministic regression gate, not the acceptance result. A
change to an interactive surface is complete only after a pass on the real
account with real data and audible playback through the real backend, covering
every control the change touched. Demo-only evidence does not count.
- **Walk every affordance the buffer exposes**: list selection and paging, row
activation, pane and tab switching, the search input, section navigation,
refresh, export, open-in-browser (stub the external program), local-index build
and query, theme switch, help, and close. An affordance without a scenario is an
untested affordance.
- **Verify the layout matrix, not one window**: every breakpoint the application
defines (wide, medium and narrow for the current readers) plus a short height,
asserting the resolved layout state and the content of each column, and
reviewing the captures of the states that matter.
- **Keep every row within the frame line height.** A symbol that falls back to
a taller font, or a text-style variation selector (U+FE0E), raises its row and
clips the bottom rows of the canvas; scenarios assert this with
`etaf-gui-verifier-line-height-assertions` and settle a resize with
`etaf-gui-verifier-canvas-filled-p`.
- **Assertions and visual review are separate steps.** A green run certifies the
behavior it checked, not the appearance: inspect the captures before claiming a
layout renders as intended.
- **Record every problem a pass finds in one list, fix them one by one with
regression coverage, then re-run the affected scenarios.** That list belongs to
the task, not to the repository, and run evidence (manifest, captures,
recording) stays outside version control.

Maintain the scenarios as `scripts/<application>-gui-scenarios.el` on the shared
`emacs-gui-verifier` engine and run them from an existing frame of the size
the matrix needs; the engine's action, checkpoint, evidence and timing contract
lives in `etaf`'s manual under GUI verification tools.

## View width discipline

- In ETAF/Ebox views, `width: stretch` fills its containing block, not remaining space. For remaining space, use `flex-basis (ch 0)` with `flex-grow 1`; never add `flex-shrink 0`, which overflows the container.
- An over-wide canvas pushes centered blocks aside; clicking its right edge can horizontally scroll the overflow. Compare each line's `window-text-pixel-size` with `(window-body-width win t)`.
- Indent Chinese paragraphs with ordinary spaces, never U+3000; a missing font glyph can render it as an underlined escape glyph.

## Provider integration pitfalls

All of these cost real debugging time while building `etaf-reader`; check them
before writing a backend call or a wrapping view.

- `call-process` takes a *file name* for stderr, never a buffer.  Pass
  `(list OUTPUT-BUFFER STDERR-FILE)` plus `make-temp-file`, read the file back
  on failure, and delete it in `unwind-protect`.  A buffer there signals
  `wrong-type-argument stringp` from inside the call.
- `json-parse-string` signals `json-trailing-content` on a trailing newline.
  Trim a CLI's stdout first (`string-trim`); `json-parse-buffer` on the whole
  buffer is the alternative.
- `json-encode` guesses alist versus plist from the first element, so a *list
  of plists* encodes as an alist (`{"type":["p",...]}`).  Store nested values
  through a small recursive encoder that always keeps keyword-keyed plists as
  JSON objects, then parse them back with `:object-type 'plist`.
- In a `etaf-view` body, a helper call in a child position must be wrapped:
  `(expr (my-view ...))`.  A bare call is treated as an Elisp expression and
  rejected with "must be inside (expr FORM)".
- When a view wraps text itself, count every prefix (indent, quote bar) in the
  wrap width, not just the source text.  `ch` and `char-width` agree for CJK
  and ASCII, so an exactly full line is fine; leaving one cell of slack in the
  column width absorbs sub-cell rounding.  Without this the column clips the
  tail of the line and the text silently loses characters.

## Runtime-only defects

Only a real session against real data exposes these; batch scenarios and
scripted demos are green while they are broken.  They are cheap to avoid at
write time and expensive to find:

- A View or mount path must not touch storage, create Data Controllers or call
  the network: ETAF rejects it as `etaf-render-side-effect-error`.  Views read
  refs and cached values; data access happens before mounting or inside
  commands and events.
- The renderer's minor-mode map owns `SPC`, `S-SPC`, `DEL`, `C-v`, `M-v`,
  `RET`, wheel and `n`/`q`.  An application that replaces them installs a
  buffer-local `emulation-mode-map-alists` entry whose key is the
  buffer-local session variable (a wrong or void key silently disables the
  whole map); `RET` dispatches between the application action and
  `etaf-activate` so focused controls keep working.
- `(setf (plist-get plist :key) value)` only writes in place when the key
  exists; on a missing key it rebinds the local variable and the caller's
  plist keeps no change.  Create every hot field with the session, or use
  `plist-put`.
- A hand-written `'(.body . nil)` reads as the symbol `.body`, not
  `(body . nil)`, and every `eq` comparison against it silently fails.
  Verify such literals at runtime (`eac def`, a batch `eq` probe).
- Retained Host updates take a **symbol** ref (a string signals "Unknown
  committed Host"), the node needs both `:ref` and `:id`, and one missing
  target rejects the whole batch before applying anything: align the batch
  with the published structure or catch `etaf-runtime-error` and republish.

A write to a reactive source republishes the whole view at 200-500 ms; move
high-frequency state into plain fields and patch its Hosts through
`etaf-runtime-update-host-content(s)`, with a tracked `:revision` source as the
explicit republish trigger.  Measure "key press to repaint", never headless
model time.

## Retained Host updates

A scoped update through ETAF's retained Host boundary (content or geometry) is
cheap in what it touches and, with the provider's definite-flex-slot projection
(`ebox` c2f1ee74), close to constant time:

- It stays scoped: the update report reads `strategy owner-rerender` or
  `span-patch`, `dirty 1`, `patch 1`, and now also carries a
  `projection-kind` of `span-patch` or `owner-scoped` for the single changed
  owner.
- Its cost follows the changed subset instead of the view: the same scoped
  content update costs ~4-5 ms on 8 to 60 rows of the player shape, where it
  used to cost 13 ms on 8 rows and ~55 ms on the 30-row player with its lyric
  panel (measured with `benchmark-run` on the mounted view).  A per-second
  scoped update is therefore affordable; re-measure after a provider change.
- Publishing out of band from ETAF's own pipeline is only safe while the
  controlled state stays authoritative.  The player now drives its per-second
  clock and meter through retained Hosts, and every interaction publishes its
  local effect eagerly (the transport state flips before the reply), so the
  interaction state never has to wait for a later publication.  Verify an
  interaction (a button, a sidebar item) after adding or changing one.
- A mount can precede the frame resize the window manager performs.  Ebox then
  records the window's width in its render state while the painted rows still
  belong to the mount-time viewport, so its own resize path sees nothing to do
  and the player stays painted at the old canvas.  `etaf-ncm--settle-canvas`
  repairs it by repainting once at a neighbouring width and once at the real
  width when a settled window grew across a layout boundary.

## Reminders

- A binding in an application major-mode map can silently fail when the renderer's minor mode owns the key. In the application buffer, `(key-binding (kbd KEY))` returning a non-application command proves it is shadowed.
- An application that must replace a renderer-owned key installs a buffer-local `emulation-mode-map-alists` entry while opening the buffer; `etaf-weread--command-maps` in `etaf-weread-open` is the pattern.
- Use that override only for keys the application owns; preserve renderer semantics such as TAB and focused RET activation.
- This is not Weread-specific: `etaf-ncm`'s major-mode RET binding is potentially shadowed too.

- Commit in the owning repository, not here, when changing an application. This
  repository's commits cover orchestration, documentation and shared tooling.
- Never track application directories, byte-compiled output, fetched providers or
  evaluation reports here.
- Keep the application list here, in `.gitignore`, in `scripts/api-workspace.json`
  and in `workspace.json` consistent when adding or removing an application.
- Publish with the shared Jujutsu workflow: aliyun (`https://gitea.vhkd.top/...`)
  is pushed automatically after validation; push laycat
  (`git@gitea.gklazycat.heiyu.space:...`) at the same checkpoints once its
  repository exists.
- A remote repository for a new application must exist before its first push.
  Create it yourself through the host API instead of waiting for the user:
  `git credential fill` with `host=gitea.gklazycat.heiyu.space` supplies the
  laycat credentials, `host=gitea.vhkd.top` the aliyun ones, and
  `POST /api/v1/user/repos {"name":...,"private":false,"auto_init":false,
  "default_branch":"main"}` creates the repository.  A refused push means the
  repository is missing or the credentials are wrong; report it instead of
  retrying blindly.

## App design rules

Distilled from the music (`etaf-ncm`) and reading (`etaf-weread`) applications:
every rule below was a user requirement or a repeated defect. Read it before
building UI, and check the list before claiming a UI change complete.

**What done means**

- Good-looking, modern and deliberate; a rough demo is a failure. The result is
a pass on the real account with real content, driving every control the change
touched, at every layout breakpoint (plus one short height).
- Scripted scenarios and screenshots certify behavior, not appearance: inspect
the captures as a separate step.

**Layout and the vertical budget**

- The frame holds exactly the window: nothing clipped at the bottom, no empty
row left over. Budget in rows; never express vertical sizes as percentages.
- No glyph may grow a line beyond the budgeted row height.  Navigation uses
the shared SVG icon set (one set per application, uniform metrics); never bold
an icon glyph.  A larger reading font is allowed only where the model also
recomputes the row budget and the canvas line count (`etaf-weread`'s reading
font), never as an inline `:font-size` on a row that keeps its old budget.
- An active or selected state is carried by colour, surface, weight and icon
colour, never by a prefix glyph.  A `* `-style marker changes the label's
width and shifts the whole row or tab strip on every switch; the observable
criterion is that the row's text is byte-identical before and after the state
change.
- The background reaches every edge, and key bars keep a padding row above and
below.
- A narrow width never truncates (a `$` marker means a width is wrong) and a
resize reflows without stalling or scrambling boxes.

**Components**

- Interactive UI is extracted into shared components before an application
consumes it: the application customizes props, it does not re-implement the
control. Building applications is how the catalog grows.
- Component defaults are modern and coherent: one icon set per application
(sidebar and bars alike) with uniform metrics that never grow the line height,
and theme tokens for spacing, states and emphasis.
- Hierarchy comes from colour, weight and spacing, not from small type.
- Applications use ETAF's public API only; a lower-level need becomes an ETAF
API first, then the application consumes that.

**Interaction**

- Pointer shape follows the control's function (click, drag, resize) and is
bound by the component, not set per application.
- Hover follows the pointer fluidly, and the selected row highlights one text
row rather than a block.
- One press acts: clicks and double clicks never need repeating, and state
changes (playback, favourite, selection, pane switch) land immediately.
- Dragging is immediate and lands where released; sliders show a handle and
which part is filled, and the printed value matches the visual one.
- Users never see an internal error; an interaction error is handled where it
arises.

**Performance**

- A face switch replaces a region, not the frame: cache the data and refresh it
in the background so network latency stays out of the interaction.
- Long lists page or scroll and never show only the first screen.
- Media and UI stay in step: lyrics follow the line being sung, artwork paints.

**Product logic**

- Follow the native client's logic: "play all" queues every track, a playlist
opens its contents instead of playing, "another batch" reloads, and a group
header keeps a gap from its first item.

**Process**

- Test in the user's own configuration, reuse one instance and close it after.
- Capture per application or window, never by focus.
- Fix the framework first when the defect is there, then re-verify it through
the application.
- Unrelated work goes to a background subagent, and a handoff lists every open
requirement.

## Documentation

- [Application and integration contract](docs/architecture.zh-CN.md)
- Shared engineering policy: `AGENTS.md` in the enclosing `emacs-taf` workspace root.
- User-facing documentation lives in the `README.md` of this root and of each
  application.
