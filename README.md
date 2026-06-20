<h1 align="center">okf.mq</h1>

An [OKF](https://github.com/GoogleCloudPlatform/knowledge-catalog) (Open Knowledge Format) reader/writer implemented as an [mq](https://github.com/harehare/mq) module.

OKF represents knowledge as a directory ("bundle") of Markdown files with YAML
frontmatter — concept documents, plus the reserved `index.md` and `log.md`
files. This module parses and builds those documents, validates conformance,
and works with cross-links, citations, and log/index entries.

## Features

- Parse/build concept documents (`{frontmatter, body}`)
- Validate a single document, or a whole bundle, against the OKF conformance rules
- Classify and extract cross-links (`external` / `absolute` / `relative`), and find broken internal links
- Parse, format, and append entries in the `# Citations` list
- Parse and build date-grouped `log.md` sections (newest-first)
- Parse and build `index.md` entries, including the bundle-root `okf_version` frontmatter
- Summarize a bundle's concept documents for an overview table

## Installation

Copy `okf.mq` to your mq module directory, or place it anywhere and reference it with `-L`.

```sh
cp okf.mq ~/.local/mq/config/
```

### HTTP Import (no local installation needed)

If `mq` was built with the `http-import` feature, you can import directly from GitHub without any local setup:

```sh
mq -I raw 'import "github.com/harehare/okf.mq" | okf::okf_parse(.)' concept.md
```

Pin to a specific release with `@vX.Y.Z`:

```sh
mq -I raw 'import "github.com/harehare/okf.mq@v0.1.0" | okf::okf_parse(.)' concept.md
```

## Usage

```sh
mq -L /path/to/modules -I raw \
  'include "okf" | okf_parse(.)' concept.md
```

If you copied it to the mq built-in module directory:

```sh
mq -I raw 'include "okf" | okf_parse(.)' concept.md
```

## API

### Concept documents

| Function | Description |
|---|---|
| `okf_parse(input)` | Splits a Markdown string into `{frontmatter, body}`. Documents without a leading `---` block get an empty `frontmatter` dict. |
| `okf_stringify(doc)` | Renders a `{frontmatter, body}` doc back into a Markdown string. |
| `okf_new(type, fields, body)` | Builds a new doc; `fields` holds the rest of the frontmatter and `type` always wins. |
| `okf_type(doc)` | Returns the doc's `type`, or `None`. |
| `okf_validate(doc)` | Returns an array of `{rule, field, message}` conformance errors (empty = conformant). `rule` is a stable machine-readable code (`invalid_frontmatter`, `missing_required_field`, `empty_required_field`); `field` names the offending frontmatter key, or `None`. |
| `okf_is_conformant(doc)` | `true` if `okf_validate(doc)` is empty. |
| `okf_format_validation_error(error)` / `okf_format_validation_errors(errors)` | Render one or many validation errors as human-readable strings, e.g. `"/tables/orders.md: ... [field: type]"`. |

### Reserved filenames

| Function | Description |
|---|---|
| `okf_reserved_filenames()` | `["index.md", "log.md"]`. |
| `okf_is_reserved_filename(path)` | `true` if `path`'s basename is reserved. |

### Cross-links

| Function | Description |
|---|---|
| `okf_link_kind(url)` | `"external"`, `"absolute"` (starts with `/`), or `"relative"`. |
| `okf_extract_links(body)` | Extracts `[text](url)` links as `{text, url, kind}`. |
| `okf_check_broken_links(body, known_paths)` | Internal links whose `url` is missing from `known_paths`. |

### Citations

| Function | Description |
|---|---|
| `okf_citation_entry(number, text, url)` | Formats `"[n] [text](url)"`. |
| `okf_extract_citations(body)` | Parses the `# Citations` list into `{number, text, url}`. |
| `okf_add_citation(body, text, url)` | Appends an auto-numbered citation, creating the section if needed. |

### Log files (`log.md`)

| Function | Description |
|---|---|
| `okf_log_entry(kind, text)` | Formats `"**Kind**: text"`. |
| `okf_log_section(section)` | Formats one `{date, entries}` section. |
| `okf_build_log(sections)` | Renders `log.md` content, newest date first. |
| `okf_parse_log(input)` | Parses `log.md` content into `[{date, entries: [{kind, text}]}]`. |

### Index files (`index.md`)

| Function | Description |
|---|---|
| `okf_index_entry(entry)` | Formats one `{path, title?, description?}` list item. |
| `okf_build_index(entries, version = "")` | Renders `index.md`; pass `version` for the bundle-root `okf_version` frontmatter. |
| `okf_parse_index(input)` | Parses `index.md` content into `[{path, title, description}]`. |

### Loading files from disk

mq has no built-in directory walk, but OKF bundles are meant to be
self-describing via `index.md` — so discovering a bundle's files doesn't
require listing directories at all, only following the index(es) it already
has.

| Function | Description |
|---|---|
| `okf_load_concept(path)` | Reads and parses a single concept document. |
| `okf_load_bundle(paths, root = "")` | Reads `[{name, content}]` for an array of paths. With `root`, rewrites `name` to a bundle-relative absolute path (e.g. `/tables/x.md`), matching `okf_link_kind`. |
| `okf_discover_bundle(root, index_path = "/index.md")` | Returns every concept document's filesystem path reachable from `root`'s `index.md`, following nested `index.md` entries. Pure `read_file` — no directory listing. |

### Bundles

| Function | Description |
|---|---|
| `okf_validate_bundle(files)` | Validates every file in `[{name, content}]`: concept documents via `okf_validate`, and `index.md`/`log.md` against the reserved-file structure rules (no frontmatter, except `okf_version` on the bundle-root `index.md`; `log.md` date headings must be ISO `YYYY-MM-DD` and sections newest-first). Returns an array of `{rule, field, message, file}` dicts. |
| `okf_is_bundle_conformant(files)` | `true` if `okf_validate_bundle(files)` is empty. |
| `okf_check_bundle(root, index_path = "/index.md")` | One-call bundle health check: discovers, loads, runs `okf_validate_bundle`, and checks every file's internal links with `okf_check_broken_links`. Returns the combined array of `{rule, field, message, file}` dicts (`broken_link` errors also carry `url`/`text`). |
| `okf_summarize(files)` | Summarizes concept documents as `[{path, type, title, description}]`. |

### Validation rule reference

Every error from `okf_validate`, `okf_validate_bundle`, and `okf_check_bundle` has a stable `rule` code you can branch on:

| Rule | Meaning |
|---|---|
| `invalid_frontmatter` | A concept document's frontmatter did not parse into a YAML mapping. |
| `missing_required_field` | A concept document's frontmatter has no `type` field. |
| `empty_required_field` | A concept document's `type` field is present but empty or blank. |
| `frontmatter_not_allowed` | `index.md`/`log.md` has frontmatter (only the bundle-root `index.md`'s `okf_version` is exempt). |
| `invalid_date_heading` | A `log.md` `##` heading isn't an ISO 8601 `YYYY-MM-DD` date. |
| `log_not_newest_first` | A `log.md`'s date sections aren't ordered newest first. |
| `broken_link` | An internal (non-external) link's target isn't among the bundle's known files (`okf_check_bundle` only). |

## Example

Given `customers.md`:

```md
---
type: BigQuery Table
title: Customers
description: Customer records table.
resource: bigquery://project/dataset/customers
tags: [pii, core]
---

# Schema

| Column | Type |
| --- | --- |
| id | INT64 |
| email | STRING |

# Citations

[1] [Data dictionary](https://example.com/dict)
```

```sh
mq -L . -I raw 'include "okf" | okf_parse(.)["frontmatter"]["title"]' customers.md
# => "Customers"

mq -L . -I raw 'include "okf" | okf_validate(okf_parse(.))' customers.md
# => []

mq -L . -I raw 'include "okf" | okf_extract_citations(okf_parse(.)["body"])' customers.md
# => [{"number": 1, "text": "Data dictionary", "url": "https://example.com/dict"}]
```

Building a bundle-root `index.md`:

```sh
mq -I null -F text 'include "okf" |
  okf_build_index([
    {path: "/tables/customers.md", title: "Customers", description: "Customer records table."}
  ], "0.1")'
# => ---
#    okf_version: 0.1
#    ---
#
#    - [Customers](/tables/customers.md) — Customer records table.
```

## Working with a bundle directory

`okf_discover_bundle` follows the bundle's own `index.md` (and any nested
`index.md` it links to) to find every concept document — purely via
`read_file`, no directory listing. `okf_check_bundle` builds on it to run
the whole health check (discover, load, validate, broken-link check) in
one call:

```sh
mq -I null 'include "okf" | okf_check_bundle("./bundle")'
# => [{"rule": "missing_required_field", "field": "type", "message": "frontmatter must have a 'type' field, but it is missing", "file": "/tables/orders.md"}, ...]
```

Render errors as plain text with `okf_format_validation_errors`:

```sh
mq -I null 'include "okf" | okf_format_validation_errors(okf_check_bundle("./bundle"))'
# => ["/tables/orders.md: frontmatter must have a 'type' field, but it is missing [field: type]", ...]
```

If you only need one piece — e.g. validation without the link check, or a
custom file list instead of `okf_discover_bundle` — compose the smaller
functions `okf_check_bundle` itself is built from:

```sh
mq -I null 'include "okf" |
  let root = "./bundle"
  | okf_validate_bundle(okf_load_bundle(okf_discover_bundle(root), root))'
# => [...same validation errors, without the broken-link check]
```

This only finds documents the index actually lists — keep `index.md` files
up to date for `okf_discover_bundle`/`okf_check_bundle` to see everything.

## License

MIT
