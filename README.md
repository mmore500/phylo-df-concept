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

# Objectives

- fast load/save*
- benefit from modern datframe tooling
    - granular deserialization of selected columns (e.g., Parquet)
    - larger-than-memory streaming operations (e.g., Polars)*
    - distributed computing operations (e.g., Dask)*
    - multithreaded operations (e.g., Polars)*
    - columnar compression for efficient storage (e.g., Parquet)*
    - categorical strings for efficient storage (e.g., Parquet)
    - vectorized operations (e.g., NumPy)*
    - just-in-time compilation (e.g., Numba)*
    - language interoperation
      - e.g., zero-copy interop via reticulate and Arrow https://blog.djnavarro.net/posts/2022-09-09_reticulated-arrow/
      - e.g., Polars between Rust and Python
    - multi-library interoperation
      - highly optimized copying between Rust and Python  
      - dataframe protocol: https://data-apis.org/dataframe-protocol/latest/API.html
    - transparent fetch from url, cloud providers (s3, google cloud, etc)
- scripting-friendly and end-user extensible
- memory efficient representation*
- unified serialization and processing

_TODO claims marked with asterisk* should be benchmarked_
    1. phydf,
    2. alife v1.0,
    3. newick (treeswift? or compacttree?)

Hopefully, this flexibility will replicate aspects of the R ecosystem's success with the `ape` datastructure

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
 
