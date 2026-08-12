# bootc Integration Guide

A practical guide for getting host RPM software working on RHEL image mode (bootc).

**Read it here: [marrusl.github.io/bootc-integration-guide](https://marrusl.github.io/bootc-integration-guide/)**

## Who this is for

Partner integration engineers: your company ships software as a host RPM or an installer, your customers are adopting RHEL image mode, and you won't build the bootc image yourself. Your customer does that. This guide is for you.

## What it covers

Upstream bootc docs are written for the people who build OS images. RHEL docs are written for the people who run the systems. This guide fills the gap in between: it's for the vendor whose software ends up inside an image someone else builds. The message up front is that what you need to change is probably less than you think. Most of what you already ship works unchanged, the walls are a short, clearly marked list, and each one has a standard fix.

## Status

**Pre-1.0 working draft.** It's public because it's useful now, not because it's finished. Expect gaps, and expect it to keep changing as bootc and RHEL image mode do.

## Contributing

Found something wrong, outdated, or missing? Open an issue or a pull request. See [CONTRIBUTING.md](CONTRIBUTING.md) for the conventions this guide follows.

## License

This repository uses two licenses:

- **The guide's prose** is licensed under [CC BY 4.0](LICENSE-DOCS).
- **Code samples** (Containerfile snippets, unit files, and similar) are licensed under the [Apache License 2.0](LICENSE).

See [LICENSE-DOCS](LICENSE-DOCS) and [LICENSE](LICENSE) for the full terms.
