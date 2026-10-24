# The Structable Format

A fusion of CSV and Markdown: typed tables with rowdocs for every row, saved views and metrics in the file, a native Mac app, and MCP for agents.

CSV for the structure. Markdown for the substance. One file.

Structable Documents are saved in an open document format, the Structable Format. This repository holds its specification. The current version is 2.0.0.

## Inside a Structable Document

A Structable Document is a folder with the extension `.structable`, which Finder shows as one file:

```text
Project Tracker.structable/
├── manifest.json        the format version and the document's identity
├── schema.json          the columns and their types
├── data.csv             the rows, each with a stable ID in the last column
├── README.md            optional: the document's description
├── views.json           optional: saved views
├── Rowdocs/             optional: Markdown rowdocs, one folder per rowdoc column
│   └── <column>/
│       └── <row-id>.md
├── Metrics/             optional: metrics, one folder each
│   └── <metric-id>/
│       ├── config.json
│       └── data.json    optional: the last computed result
└── AGENTS.md            optional: a guide for agents
```

## The specification

[Structable file format specification](spec/2.0/structable-specification.md), version 2.0.0, defines each file, its rules and how values are stored.

## License

The specification and this README are licensed under the [Creative Commons Attribution 4.0 International License](LICENSE) (CC BY 4.0).

Copyright 2026 Cynapse Technologies Inc.

Structable is made by Cynapse.

Structable is a trademark of Cynapse.
