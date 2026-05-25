# Cluster General Data Search

A lightweight Python tool to retrieve object-centric metadata for galaxy clusters
(or cluster candidates) from major public databases, including SIMBAD and NED.

This script is intentionally standalone, minimal, and conservative in scope.
It retrieves cluster-level information (coordinates, redshifts, alternate names,
publications) without returning every galaxy in a field, making it suitable as an
early-stage lookup or orchestration component in larger cluster analysis pipelines.

---

## Core Design Philosophy

This script is object-centric, not region-centric.

Instead of performing wide cone searches that return thousands of member galaxies,
it:

1) Resolves the input (name or coordinates) to one best cluster-like object
2) Queries SIMBAD and NED about that object only
3) Collects:
   - Coordinates
   - Object type
   - Redshift estimates
   - Alternate names / cross-identifications
   - Bibliographic references (best-effort)

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
- Clean, machine-readable outputs (JSON + text)
- Each backend can fail independently; failures are recorded in notes

---

## Installation

Create and activate a Python environment (recommended):

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

Two environment variables control where the script reads and writes by
default. CLI flags (`--outdir`, `--map-csv`) always override.

| Variable           | Purpose                                              | Fallback                                                       |
| ------------------ | ---------------------------------------------------- | -------------------------------------------------------------- |
| `CLUSTER_DATA_DIR` | Output root. Each run lands in `<root>/<cluster>/pub_search/`. | Sibling `Clusters/` directory next to this repo's parent dir.  |
| `CLUSTER_ID_MAP`   | Path to the alias-mapping CSV.                        | `cluster_id_map.csv` in that same sibling `Clusters/` directory. |

Example shell setup:

    export CLUSTER_DATA_DIR=~/path/to/my/cluster_data
    export CLUSTER_ID_MAP=~/path/to/my/cluster_id_map.csv

Leaving them unset is fine; the fallbacks resolve relative to the script's
location so a fresh clone has somewhere sensible to write without any
machine-specific configuration.

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

For a target resolved as RMJ121917_6+505432_8, the output directory may contain:

    cluster_general_output/
    ├── RMJ121917_6+505432_8_general_summary.json
    ├── RMJ121917_6+505432_8_alt_names.txt
    ├── RMJ121917_6+505432_8_publications.txt
    ├── RMJ121917_6+505432_8_publications_bibcodes.json
    └── RMJ121917_6+505432_8_publications_resolved.json  (if ADS token set)

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
- No attempt to infer cluster membership or extent

---

## Roadmap (Planned, Not Implemented)


- Configurable allowed object types
- Better handling of ambiguous multi-object regions
- Integration with photometric, redshift, and radio lookup tools
- Unified multi-wavelength cluster metadata driver


