# cluster-search

A lightweight Python tool to retrieve object-centric metadata for galaxy clusters
(or cluster candidates) from major public databases, including SIMBAD and NED.

The package is intentionally standalone, minimal, and conservative in scope.
It retrieves cluster-level information (coordinates, redshifts, alternate names,
publications) without returning every galaxy in a field, making it suitable as an
early-stage lookup or orchestration component in larger cluster analysis pipelines.

---

## Design Philosophy

The tool is object-centric, not region-centric.

Instead of performing wide cone searches that return thousands of member galaxies,
it:

1) Resolves the input (name or coordinates) to one best cluster-like object
2) Queries SIMBAD and NED for metadata about that object
3) Collects:
   - Coordinates
   - Object type
    - Redshift candidates
   - Alternate names / cross-identifications
    - Bibliographic references

This avoids the “member-galaxy soup” problem and keeps outputs lightweight and
interpretable.

---

## Features

- Resolve targets by:
  - Name / identifier (e.g. RMJ IDs, observation IDs, catalog names)
  - RA / Dec (ICRS degrees)
- Optional mapping CSV for:
  - Short numerical tokens
  - Observation IDs
  - Internal naming schemes
- Object-centric metadata retrieval:
  - SIMBAD:
    - Coordinates
    - Object type
    - Alternate IDs
    - Redshift (rvz)
    - Bibliography (bibcodes)
  - NED:
    - Coordinates
    - Redshift (+ uncertainty, if available)
    - Alternate names
    - References
- Intelligent fallback logic:
  - Name lookup → coordinate seed search → nearest cluster-type object
- Clean outputs (JSON + text)
- Each backend can fail independently; failures are recorded in notes

---

## Installation

Create and activate a Python environment (recommended):

    python -m venv .venv
    source .venv/bin/activate

Or with Conda:

    conda create -n cluster_lookup python=3.11
    conda activate cluster_lookup

Install editable from this repo:

    pip install -e .

This declares `astropy`, `astroquery`, `pandas`, `numpy`, and `requests` as
runtime dependencies (see `pyproject.toml`) and registers a
`cluster-data-search` console script on `PATH`.

Optional: resolving publication metadata via ADS

- Create an ADS API token
- Set it as an environment variable:

    export ADS_API_TOKEN="your_token_here"

---

## Configuration

Two environment variables control default paths. CLI flags (`--outdir`,
`--map-csv`) always override.

| Variable           | Purpose                                              | Fallback                                                       |
| ------------------ | ---------------------------------------------------- | -------------------------------------------------------------- |
| `CLUSTER_DATA_DIR` | Output root. Each run lands in `<root>/<cluster>/pub_search/`. | `Clusters/` directory near the checked-out repository (see code defaults). |
| `CLUSTER_ID_MAP`   | Path to the alias-mapping CSV.                        | `cluster_id_map.csv` in that same default `Clusters/` directory. |

Example shell setup:

    export CLUSTER_DATA_DIR=~/path/to/my/cluster_data
    export CLUSTER_ID_MAP=~/path/to/my/cluster_id_map.csv

If you do not set these variables, built-in defaults are used.

---

## Usage

Resolve by name:

    cluster-data-search --name "RMJ121917.6+505432.8"

Resolve by coordinates:

    cluster-data-search --radec 184.8235 50.9091

Specify output directory:

    cluster-data-search --name "RMJ0003" --outdir ./cluster_general_output

Use a custom mapping CSV:

    cluster-data-search --name "0881900801" --map-csv ./cluster_id_map.csv

Use a custom output tag:

    cluster-data-search --name "RMJ121917.6+505432.8" --tag RMJ_1219

The equivalent module form also works: `python -m cluster_search.data_search ...`.

---

## Mapping CSV (Optional)

If a mapping CSV is found at the resolved `CLUSTER_ID_MAP` path (or is
passed explicitly via `--map-csv`), it will be used to normalize
identifiers. See the Configuration section above for default resolution.

Required columns (case-insensitive):

    alias,full_name

Example:

    alias,full_name
    0003,RMJ000343.8+100123.8
    0881900801,RMJ121917.6+505432.8

This allows:
- Short numerical tokens
- Observation IDs
- Internal naming schemes

to resolve cleanly to a canonical cluster name.

---

## Output Structure

For a target with `--tag RMJ_1219`, files land under
`<outdir>/RMJ_1219/pub_search/`:

    <outdir>/
    └── RMJ_1219/
        └── pub_search/
            ├── general_summary.json          (machine-readable summary; primary output)
            ├── alt_names.txt                 (raw alt-name list from SIMBAD + NED)
            ├── alt_names_cleaned.txt         (canonicalized + deduped, with catalog-prefix descriptions header)
            ├── publications.txt              (raw bibcode list)
            ├── publications_bibcodes.json
            ├── publications_resolved.json    (full ADS metadata; only when ADS_API_TOKEN is set)
            └── publications_filtered.json    (subset whose title/abstract mention this cluster)


`--tag` overrides the auto-derived directory name. Without `--tag`, the
name comes from `safe_stem()` applied to the resolved cluster name.

Files are only written when meaningful content exists.

---

## General Summary JSON

The primary output is a structured JSON summary containing:

- Input resolution information
- Adopted target object name and type
- Coordinates and source of coordinate resolution
- Redshift candidates by source
- Deduplicated alternate names
- Cross-ID groupings
- Bibliographic references
- Notes describing failures, fallbacks, or ambiguities

This file is designed for downstream automation or ingestion by larger pipelines.

---

## Known Limitations

- Redshifts are reported by source, not combined or reconciled
- Object classification depends on SIMBAD/NED typing
- Bibliography coverage varies by service
- No bulk / batch mode yet


