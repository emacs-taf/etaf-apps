# ETAF applications (archived)

This repository is archived. The application workspace it hosted was merged into
the single [emacs-taf](https://github.com/emacs-taf/emacs-taf) workspace, where
every repository — framework packages, applications and games — is one sibling
checkout registered in `workspace.json`.

- Sources: each application is its own repository under the `emacs-taf`
  organisation (`etaf-ncm`, `etaf-weread`, `etaf-md-preview`, …).
- Development: clone the workspace root and follow its README. There is one
  checkout per repository, so no second workspace has to be kept in sync.
- Installation: `M-x etaf-install` from the package archive at
  <https://etaf.top/packages/>, or install a single package from MELPA-style
  archive metadata.
- The application table, the application and integration contract, the shared
  checks and the integration checkpoint moved to the workspace root repository
  (`README.md`, `docs/architecture.zh-CN.md`, `workspace.json`).

Git history here remains readable; nothing in it was rewritten.
