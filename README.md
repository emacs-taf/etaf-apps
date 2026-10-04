# ETAF applications

Applications built on the ETAF text-application framework. Each application is an
independent Emacs Lisp repository with its own history, public interface and checks;
this root owns their integration checkpoint, shared tooling and the conventions they
follow.

| Application | Responsibility | Framework dependencies |
| --- | --- | --- |
| [etaf-ncm](https://gitea.vhkd.top/geekinney/etaf-ncm) | NetEase Cloud Music player: library browsing, playback control, lyrics and queue | etaf, etaf-ui, ebox |
| [etaf-weread](https://gitea.vhkd.top/geekinney/etaf-weread) | WeChat Reading client: shelf, notebook, highlights, store search, statistics and a local SQLite highlight index | etaf, etaf-ui, etaf-db, ebox |
| [etaf-md-preview](https://gitea.vhkd.top/geekinney/etaf-md-preview) | Markdown reading view: centered measure, headings, tables, callouts, code, footnotes and media cards | etaf, etaf-ui, ebox |
| [etaf-rss](https://gitea.vhkd.top/geekinney/etaf-rss) | RSS reader: subscriptions, folders, unread and starred scopes, reading pane, search and OPML | etaf, etaf-ui, ebox |
| [etaf-reader](https://gitea.vhkd.top/geekinney/etaf-reader) | Local ebook reader: library scan, EPUB/TXT/Markdown/HTML/FB2/PDF import, paged and scrolling reading, annotations and progress | etaf, etaf-ui, etaf-db, ebox |
| [etaf-pi](https://forgejo.vhkd.top/geekinney/etaf-pi) | Native Pi agent client: streamed conversations, tool output, session recovery and multiline input | etaf, etaf-ui, ebox |
| [etaf-telega](https://forgejo.vhkd.top/geekinney/etaf-telega) | Telegram client on Telega's TDLib backend: chats, live message stream, in-place composer and message actions | etaf, etaf-ui, ebox |

## Requirements

- Emacs 29.1 or newer, with the framework packages available on `load-path`.
- Python 3.10+ and Git for the repository policy checks.
- The runtime dependencies of the application you run; see its README.

The framework repositories are the dependency providers (`etaf`, `etaf-ui`, `ebox`,
`ecss`). Clone them as siblings of this directory, keeping those directory names; each
application's `scripts/api-workspace.json` records the providers it is checked against
and can fetch them for a standalone checkout.

## Setup

```sh
git clone -o laycat git@gitea.gklazycat.heiyu.space:geekinney/etaf-apps.git
cd etaf-apps
git clone -o laycat git@gitea.gklazycat.heiyu.space:geekinney/etaf-ncm.git
make setup-hooks
```

Each application README documents its own installation and usage. This root adds the
collective checks:

```sh
make check                  # policy, compilation, public API and acceptance for every application
make check EMACS=/path/to/emacs
make api-check              # public entry bindings and signatures of every application
(cd etaf-ncm && python3 scripts/check-api.py --root . --fetch-providers --providers-dir ../..)
                            # fetch the providers the applications are checked against
```

The recorded revisions in `workspace.json` identify the application commits verified
together; they are an integration checkpoint, not a release certification.

## Documentation

- [Application and integration contract](docs/architecture.zh-CN.md)
- [Development instructions for this workspace](AGENTS.md)
- Per-application manuals and architecture notes in each application repository.

## Contributing

Commit in the owning repository, keep the application's `make check` green, then
refresh the recorded revision in `workspace.json`. Message and structure gates run
from `.githooks` after `make setup-hooks`.
