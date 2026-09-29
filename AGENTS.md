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

## Reminders

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
