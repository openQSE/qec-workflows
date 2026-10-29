# QEC workflows

Documentation exploring end-to-end quantum error correction (QEC) workflows,
software actors, interface boundaries, and open architecture questions.

Start with the [workflow guide](docs/qec-workflows/README.md). Its numbered
chapters progress from resource reservation and application execution through
compilation, decoder configuration, real-time QEC, learning, and recovery.
Diagrams use Mermaid and can be viewed in GitHub's Markdown renderer.

## Documents

| Document | Contents |
| --- | --- |
| [QEC workflow guide](docs/qec-workflows/README.md) | Numbered sequence diagrams, component relationships, shared actors, and open questions. |
| [QEC workflow analysis](docs/qec-wl-analysis.md) | Background analysis of responsibilities, physical mapping, decoder binding, and timing domains. |
| [October 19 meeting minutes](10-19-2026.md) | Initial discussion of modular QEC workflows and stack integration. |
| [October 21 CCAM/Rust meeting minutes](docs/rust-infra.md) | Hybrid execution, facility integration, and the need for an end-to-end stack description. |

The documents distinguish proposed architecture from documented interfaces.
Meeting notes are inputs to the design rather than specifications. Some source
links reference a separate local openQSE knowledge base and require that checkout
to be available; vendor references link to their published documentation.
