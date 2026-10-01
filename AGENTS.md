# Workspace development instructions

This root owns the integration checkpoint, shared tooling and conventions for the
ETAF text applications. Every application is an independent Jujutsu/Git repository in
this directory, ignored by this repository and committed from its own checkout.

## Layout and providers

- Applications live at `./<name>` (`etaf-ncm` today). Their framework providers
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
- **Walk every affordance the buffer exposes**: list selection and paging, row
activation, pane and tab switching, the search input, section navigation,
refresh, export, open-in-browser (stub the external program), local-index build
and query, theme switch, help, and close. An affordance without a scenario is an
untested affordance.
- **Verify the layout matrix, not one window**: every breakpoint the application
defines (wide, medium and narrow for the current readers) plus a short height,
asserting the resolved layout state and the content of each column, and
reviewing the captures of the states that matter.
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
  is pushed automatically after validation, laycat
  (`git@gitea.gklazycat.heiyu.space:...`) is published manually.

## Documentation

- [Application and integration contract](docs/architecture.zh-CN.md)
- Shared engineering policy: `AGENTS.md` in the enclosing `emacs-taf` workspace root.
- User-facing documentation lives in the `README.md` of this root and of each
  application.
