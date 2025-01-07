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

_TODO claims marked with asterisk* should be benchmarked_ to compare
1. proposed standard,
2. alife standard v1.0 (with `ancestor_list`),
3. newick format (treeswift? or compacttree?),
4. nextrain's `.tsv` data (???)

# Approach

- define a more restrictive WORKING format to simplify/streamline processing operations
   - requires `id` to equal row number
   - requires taxa to be topologically sorted
   - requires names of standard fields to be prefixed with PREFIX_
   - includes concept of "TRANSIENT" columns (see below)
- define a more flexible STORAGE format
   - better accommodate raw data from simulation or inference pipelines
   - backwards compatibility with alifestd v1 data
- WORKING and ALIFESTDV1 formats are distinct subsets of STORAGE format
    ```
    +------ storage format ------+
    |                            |
    |   +-- working format --+   |
    |   +--------------------+   |
    |                            |
    |   +-- alifestdv1 fmt --+   |
    |   +--------------------+   |
    |                            |
    +----------------------------+
    ```
- WORKING format may be detected by lack of `id` and `PREFIX_id` columns
    - non-WORKING format data MUST include `id` or `PREFIX_id` columns
- define standardized `from_storage` and `to_storage` transforms
    - Pandas, Polars, and R tools are provided implementing `from_storage`
    - in-memory processing tools should assume working format and may raise an error if storage format data is passed
    - tools implementing `from_storage` transform are encouraged but not required to support full breadth of storage format (e.g., alifestd v1)

# WORKING format

- goal: support optimized processing
- context: in-memory operations (e.g., `pd.DataFrame` `pl.DataFrame`)
- taxon id MUST match row number (no `PREFIX_id` column)
- MUST be topologically sorted (ancestors BEFORE descendants)
- MUST be non-reticulating asexual tree(s)
- MAY have multiple roots
- PRESENCE of `PREFIX_origin_time` implies **rooted** tree; otherwise, **unrooted**
- operations must respect TRANSIENT column rules, see below

# TRANSIENT columns

- identified by column names prefixed with `__` (two underscores) or `___` (three underscores)
- `___` (3 underscores) indicates the column is "scratch" and to be dropped at the conclusion of the current function (not passed to end-users)
   - i.e., stronger ephemerality --- no need for re-use
- `__` columns may be returned from functions in library or end-user code
   - can be used other library or end-user code
   - when present, `__` columns may be assumed to be valid and up-to-date
- `__` must be dropped by operations that mutate tree structure
    - unless calling user passes a `keep` override
    - unless KNOWN to not be invalidated by particular mutation performed
    - unless recalculated/repaired as part of mutation operation
- `__` must be dropped in `to_storage` transform
    - unless calling user passes a `keep` override
- standardized "conventional" columns invalidated by changes in tree structure are prefixed with `__`
- end-users or library authors may also prefix their own columns with `__`

what should be treated as a transient column?
- can be calculated on-demand from other columns
- may be a supporting "ingredient" in follow-on calculation of interest
    - e.g., "node depth from root," used as part of later calculation)
- MAY have a standardized definition and prefixed with PREFIX_
    - or, alternately, be bespoke/ad hoc (user/library-defined)
- MAY become invalidated when tree is mutated
    - therefore MUST be updated or deleted when tree is mutated

# STORAGE format `-->` WORKING format transform

- unpack and drop `ancestor_list`
    - if >1 ancestor, a list of dataframes must be returned
    - alternately, a NotSupported error may be thrown if this column is present
- re-assign `id`s to be
    - contiguous `0-n
    - topologically sorted (i.e., `ancestor_id` <= `id`)
    - `id` == row number
    - alternately, a NotSupported error may be thrown if the `id` or `PREFIX_id` column is present
- drop `id` (or `PREFIX_id`) column
- if `PREFIX_name` not provided, prefix conventional columns with `PREFIX_`
    - e.g., `origin_time` -> `PREFIX_origin_time`
    - if both `origin_time` and `PREFIX_origin_time` provided, keep as-is

# WORKING format `-->` STORAGE format transform

- drop `__`-prefixed columns
   - unless overridden to `keep` by caller

# WORKING format vs alife phylogeny standard v1

- replace `ancestor_list` column with `ancestor_id` column
    - `ancestor_id` avoids complications (and large slowdown) in save/load and processing operations
    - `ancestor_id` explicitly differentiates reticulated and non-reticulated trees (see SEXUAL PEDIGREES below)
    - `ancestor_id` == `id` for root
- enable strong assumptions about layout/indexing for easy/fast processing in WORKING format
- use PREFIX_ to prevent name collisions
- add concept of TRANSIENT columns
    - flexibly and extensibly handle data invalidation by changes to tree structure
- add explicit support/differentiation between rooted and unrooted trees
    - key for non-Alife use case

# API design suggestions for library authors

- encouraged to accept a `mutate` param, default `False` (data is copied before mutating operations, unless `True`)
- encouraged to return DataFrame by value
- encouraged to accept `keep` param, that allows end users to prevent select transient columns from being dropped
   - `keep` param should allow regex
- encouraged to drop all `___`-prefixed columns
- encouraged to return same column names and data types, no matter what input data is passed
- for Python, a function decorator that handles these operations is provided in standard support library

- encouraged to prefix all columns added with distinct library `SLUG_`
- for columns added that may have a more general use case,
    - encouraged to suggest column name for standardization (`PREFIX_XYZ`) and
    - list/link library into registry as implementing the calculation of this column

# Support for representation and storage on SEXUAL PEDIGREES

 - i.e., scenarios where `taxa` may have >1 parent
 - in WORKING FORMAT context, sexual pedigrees must be represented as a collection of discrete DataFrames
     - each dataframe is an independent asexual phylogeny
     - e.g., separate matrilineal phylogeny and patrilineal phylogeny
     - operations intended to explicitly support pedigrees therefore MUST take arguments as >1 discrete asexual dataframes
         - (eg as a list of `n` dataframes or n discrete parameters)
    - if number of parents varies, taxa with fewer than `k` parents should be represented as a root in the `k`th DataFrame
    - a taxon's row index MUST correspond exaclty between DataFrames
- in STORAGE FORMAT context, pedigrees SHOuLD be
    - stored as `n` contiguous chunks within the same table, each chunk as an asexual tree;
        - in this case, MUST be partitioned by integer index 0,1,..., `n` as column `PREFIX_pedigree_index`
    - stored as `n` different tabular data files
    - pedigress MAY instead be represented using alifestd v1 `ancestor_list`, although this is discouraged

# Conventional columns

these two allow iterating over all children (like a linked list), shoud be set to `id` of taxon if has no children or has no next sibling (is "last" child):
- `PREFIX_first_child_id`
- `PREFIX_next_siblng_id`

- `PREFIX_origin_time`
- `PREFIX_edge_time`

TODO scrape alifestd_* functions [here](https://github.com/mmore500/hstrat/tree/e85acde9566472ce2bb1e54e21e0e977084cf26b/hstrat/_auxiliary_lib) for column names used

# Reserved names

- non-conventional `PREFIX_`, `_PREFIX`, `__PREFIX` or `___PREFIX` columns are UNDEFINED BEHAVIOR
- might also reserve `PREFIX1_`, `PREFIX2_`, `PREFIX3_` etc.

# Questions/Problems

- should `PREFIX_` be `alstd_`, `alst2_`, or `phydf_`?
- should brand as alife standard phylogeny v2 or as alife asexual phylogeny standard or non-alife standard?
- should working format require independent trees to be in contiguous row sections?
   - this would create problems with representing sexual pedigrees as distinct trees
