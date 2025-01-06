# Phylogeny Dataframe Concept

[![CI](https://github.com/mmore500/phylo-df-concept/actions/workflows/ci.yaml/badge.svg)](https://github.com/mmore500/phylo-df-concept/actions/workflows/ci.yaml)
[![GitHub stars](https://img.shields.io/github/stars/mmore500/phylo-df-concept.svg?style=flat-square&logo=github&label=Stars&logoColor=white)](https://github.com/mmore500/phylo-df-concept)

- manuscript draft: <https://mmore500.github.io/phylo-df-concept/>

To set up locally,
```bash
git clone --single-branch https://github.com/mmore500/phylo-df-concept.git
cd phylo-df-concept
./submodules.sh
```

# Summary

The goal of this document is to establish conventions for representing phylogenetic trees in tabular DataFrame formats to enable
- high-performance in-memory processing operations,
- space-efficient, fast save/load to tabular data formats, and
- a decentralized, highly composable, and interoperable ecosystem of phylogeny libraries.

# Why a DataFrame-based tree representation?

- DataFrames are scripting-friendly and end-user extensible
- potential for composable, interoperable ecosystem
    - R ecosystem's success with the `ape` data structure
    - push this idea further, with a fully tabular format
- fast and highly portable load/save*
    - e.g., `pandas.read_csv`, `polars.read_parquet`, `read.table`, etc.
    - libraries will transparently fetch from url, cloud providers (s3, google cloud, etc)
    - unified serialized and in-memory representations
- benefit from modern tabular data formats
    - granular deserialization of selected columns (e.g., Parquet)
    - compression configuration that is transparent to end-user (e.g., Parquet) 
    - columnar compression for efficient storage (e.g., Parquet)*
    - categorical strings for efficient storage (e.g., Parquet)
    - explicit column typing (e.g., Parquet)
    - options exist for both binary and text formats
- benefit from modern high-performance dataframe tooling
    - memory-efficient representation*
    - larger-than-memory streaming operations (e.g., Polars)*
    - distributed computing operations (e.g., Dask)*
    - multithreaded operations (e.g., Polars)*
    - vectorized operations (e.g., NumPy)*
    - just-in-time compilation (e.g., Numba)*
- rich interoperative ecosystem
    - multi-language interoperation
      - e.g., zero-copy interop between R and Python [via reticulate and Arrow](https://blog.djnavarro.net/posts/2022-09-09_reticulated-arrow/)
      - e.g., zero-copy Polars DataFrames shared between Rust and Python
    - multi-library interoperation
      - e.g., highly-optimized conversion, or even [zero copy](https://pythonspeed.com/articles/polars-pandas-interopability), interoperation between Polars and Pandas  
      - e.g., [Python dataframe protocol](https://data-apis.org/dataframe-protocol/latest/API.html)

_TODO claims marked with asterisk* should be benchmarked_
    1. phydf,
    2. alife v1.0,
    3. newick (treeswift? or compacttree?)

# Questions
- should `PREFIX_` be `alstd_`, `alst2_`, or `phydf_`
- should brand as alife standard phylogeny v2 or as alife asexual phylogeny standard or non-alife standard? 

# Key Concepts

- define 2 formats (WORKNG and STORAGE)
- define standardized transform (WORKING -> STORAGE) and WORKING -> STORAGE

- working format
    - goal: support optimized processing
    - context: in memory operations (e.g., `pd.DataFrame` `pl.DataFrame`)
    - taxon id MUST match row number (no id column)
    - MUST be topologically sorted (ancestors BEFORE descendants)
    - MUST be non-reticulating asexual tree(s)
    - MAY have multiple roots
    - PRESENCE of origin_time implies rooted; otherwise, unrooted

- working format: transient attributes
    - semantically, can be calculated on-demand
    - semantically, may be a supporting “ingredient” in calculation of interest (e.g., “node depth from root”)
    - physically, is a named column in dataframe
    - prefixed with `__`
    - MAY have a standardized definition (to enable utility composability)
    - MAY alternately be bespoke or ad hoc
    - MAY become invalidated when tree is mutated
        - MUST be updated or deleted when tree is mutated
    - do not serialize (by default) 

- storage format
    - goal: flexibly support raw output from simulations or outside pipelines
    - SUPERSET of working format
    - SUPERSET of alifestd v1 format
    - context: on disk format (e.g., `.parquet` `.csv` `.tsv` etc)
    - PRESENCE of origin_time implies rooted; otherwise, unrooted
 
 - sexual pedigrees (WORKING FORMAT)
    - operations intended for scenarios where ids may have more than one parent
    - MUST take arguments as >1 discrete asexual dataframes (eg as a list of dataframes or n discrete parameters)
    - taxon row index MUST correspond between dataframes

- sexual pedigrees (STORAGE FORMAT)
    - SHOULD be stored as `n` contiguous chunks, each chunk as an asexual tree;
        - in this case, MUST be partitioned by integer index 0,1,… as column `alstd_pedigree_index`
    - MAY be stored as alifestd v1 ancestor list

# problem
- how to indicate stored data is un working format —- if not topologically sorted, id or alstd id col MUST be provided
- suffix with _rX

# Schematic

ALSTD1.1
- as specified on website
- deprecated
- fix empty list specification

ALSTD2 storage format
- optionally, includes ALSTD1.1 and ALSTD1 (otherwise, must error)
- MUST include id or alstd_id column if NOT working format

ALSTD2 working format 
- is valid for storage format
- MUST include 
- may NOT include id or alstd_id column
- non-conventional _*alstd_.* columns are UNDEFINED BEHAVIOR
- all __alstd_ columns must be dropped or left in a valid state

working to storage format transform
- drop all __-prefixed columns


storage to working format transform
- sort topologically
- assign contiguous ids
- add alstd_ prefix
- add extra underscore to _alstd_* items
  - what about user _-prefixed columns?

working format transforms
- that affect the tree topology or branch lengths MUST drop all __ prefixes, unless it can be guaranteed that they will be updated
 
