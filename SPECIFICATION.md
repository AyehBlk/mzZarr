# mzZarr 1.0 Format Specification

**Version:** 1.0
**Status:** Normative specification, frozen 29 October 2026

## 1. Scope and terminology

mzZarr is a vendor-independent storage format for mass-spectrometry
spectra. It uses Zarr version 3 for numerical peak arrays and Apache
Parquet for per-spectrum metadata.

A conforming mzZarr dataset is defined by the requirements in this
document, independently of any particular software implementation.

The words **MUST**, **MUST NOT**, **REQUIRED**, **SHOULD**, and **MAY**
express normative requirements.

PTERO is a reference implementation, not a dependency of mzZarr.

This specification defines a single logical dataset stored as a Zarr
version 3 group with a Parquet sidecar. It does not define an archive
container, network protocol, or mandatory directory-name extension.

## 2. Dataset layout

The root MUST be a Zarr version 3 group.

The following objects are REQUIRED:

| Path | Storage | Type | Dimensions |
|---|---|---|---|
| `peaks/mz` | Zarr array | float64 | One-dimensional |
| `peaks/intensity` | Zarr array | float32 | One-dimensional |
| `offsets` | Zarr array | int64 | One-dimensional |
| `spectra.parquet` | Parquet file | See Section 5 | One row per spectrum |

The following object is OPTIONAL:

| Path | Storage | Type | Dimensions |
|---|---|---|---|
| `peaks/ion_mobility` | Zarr array | float32 | One-dimensional |

All peak arrays MUST share the same flattened peak order.

Numerical array types refer to their logical Zarr data types.
Implementations MUST NOT depend on a particular chunk size, compressor,
or chunk-key encoding beyond what is valid for the selected Zarr v3
representation. Readers claiming support for a particular Zarr v3 codec
configuration MUST decode it correctly; support for every optional
codec is not implied.

## 3. Spectrum boundaries and reconstruction

Let `N` be the number of spectra and `P` the total number of peaks.

The `offsets` array MUST satisfy all of the following:

- Its length is exactly `N + 1`.
- Its first element is zero.
- Its values are nonnegative and monotonically nondecreasing.
- Its final element is exactly `P`.
- `peaks/mz` and `peaks/intensity` each contain exactly `P` elements.
- If `peaks/ion_mobility` exists, it contains exactly `P` elements.

For zero-based spectrum index `i`, the corresponding peak interval is
the half-open range `[offsets[i], offsets[i+1])`.

Equal adjacent offsets represent a valid empty spectrum. Empty spectra
contribute zero values to every peak array. A dataset with zero spectra
has an offsets array containing one zero and zero peak values.

The order of spectra MUST be the order represented by `offsets` and
the rows of `spectra.parquet`.

## 4. Format identity and version

The root Zarr group attributes MUST include:

| Attribute | Required value |
|---|---|
| `mzzarr_format` | String `mzZarr` |
| `mzzarr_format_version` | String `1.0` |

A reader MUST NOT silently treat an unknown format identifier or an
unsupported format version as conforming to mzZarr 1.0.

Implementations MAY offer separate compatibility modes for historical
stores without these identifiers. Such compatibility does not make
those historical stores conforming mzZarr 1.0 datasets.

The attributes `ptero_format` and `ptero_format_version` are
implementation-specific compatibility metadata and are not required
for mzZarr conformance.

## 5. Per-spectrum Parquet metadata

`spectra.parquet` MUST contain exactly `N` rows, in spectrum order.

At minimum, the following columns are REQUIRED:

| Column | Parquet logical type | Requirements |
|---|---|---|
| `scan_index` | int64 | Nonnull; row `i` has value `i`, starting at zero |
| `ms_level` | int64 | Nonnull; positive integer |

`scan_index` is the mzZarr zero-based spectrum position. It MUST NOT
be assumed to be the native scan number assigned by the source
instrument or acquisition software.

Additional columns MAY be included. Their names, types, and semantics
MUST NOT change the interpretation of the required columns.

Column `ion_mobility`, when present, describes per-spectrum mobility
metadata and is distinct from the optional per-peak
`peaks/ion_mobility` array. Its presence alone MUST NOT imply that
per-peak mobility is available.

This version does not require a fixed set of additional precursor,
retention-time, isolation-window, or vendor-specific metadata columns.
Consumers MUST tolerate additional Parquet columns.

## 6. Optional root attributes

The following attributes MAY be present:

| Attribute | Meaning |
|---|---|
| `n_spectra` | Integer number of spectra, equal to `N` |
| `has_4d` | Boolean indicating presence of per-peak ion mobility |
| `acquisition_mode` | String: `dda`, `dia`, or `unknown` |
| `peaks_variables` | Ordered list of stored peak-variable names |
| `rtime_unit` | String `second` |

When `has_4d` is present, it MUST be true if and only if
`peaks/ion_mobility` exists.

When `peaks_variables` is present, it MUST equal either
`["mz", "intensity"]` or
`["mz", "intensity", "ion_mobility"]`, consistently with the
actual peak arrays.

When `n_spectra` is present, it MUST equal the Parquet row count and
the number of offset intervals.

When `acquisition_mode` is present, its value MUST be one of the three
listed values. `unknown` does not imply a particular acquisition mode.

When `rtime_unit` is present, its value MUST be `second`. The attribute
does not by itself require a retention-time column.

Additional root attributes MAY be present. PTERO-specific QC
attributes, including `ptero_qc_v1`, are not normative mzZarr fields.

## 7. Ion-mobility semantics

The optional `peaks/ion_mobility` array represents one mobility value
for each flattened peak, in exactly the same order as `peaks/mz`.

If per-peak mobility is present, the root attributes MUST contain both:

- `peak_ion_mobility_type`
- `peak_ion_mobility_unit`

If a per-spectrum `ion_mobility` metadata column is present and carries
mobility measurements, the root attributes MUST contain both:

- `spectrum_ion_mobility_type`
- `spectrum_ion_mobility_unit`

For mzZarr 1.0, the supported quantity/unit pairs are:

| Type | Unit |
|---|---|
| `inverse_reduced_ion_mobility` | `V.s/cm2` |
| `ion_mobility_drift_time` | `millisecond` |

The type and unit MUST describe the actual measurement. Writers MUST
NOT guess or silently substitute one mobility quantity for another.

A mobility declaration MUST NOT be present without corresponding data
at its declared scope. Peak-level and spectrum-level declarations are
independent; one MUST NOT be used as a substitute for the other.

## 8. Missing data and mobility completeness

When `peaks/ion_mobility` exists, its length MUST equal the total peak
count, including across batches and empty spectra.

A writer MUST NOT fabricate per-peak mobility values solely to make
the mobility array align with other peak arrays.

In particular, a conversion that represents nonempty spectra with
per-peak mobility MUST NOT silently replace absent source mobility
arrays with generated NaN placeholders.

An empty spectrum contributes no mobility values and therefore does
not require a source mobility array.

This specification does not define a universal missing-mobility
sentinel or a blanket prohibition on IEEE floating-point NaN values.
A NaN already present in source measurements is a separate question
from fabricating values to conceal absent per-peak measurements.
Conformance of numerical values does not itself establish their
scientific correctness.

When a source dataset cannot be represented without inventing required
per-peak mobility measurements, the writer MUST reject that
representation rather than silently manufacture values.

## 9. Conformance and validation

A strictly conforming mzZarr 1.0 dataset MUST satisfy the required
layout, data types, spectrum boundaries, metadata columns, identity,
and applicable semantic declarations in this specification.

A conformance checker SHOULD report failures separately for:

- Physical layout and Zarr version.
- Required array types, shapes, and lengths.
- Offset consistency.
- Parquet row count and required column types and values.
- Root identity and optional attribute consistency.
- Ion-mobility quantity, unit, and array alignment.

Structural validation does not establish correct interpretation of
source-vendor data, calibration, peak picking, identification results,
or scientific measurement accuracy.

A reader MAY support additional historical or nonconforming datasets,
but MUST distinguish that compatibility behavior from strict
mzZarr 1.0 conformance.

## 10. Extensibility and version evolution

Writers MAY include additional Zarr arrays, Parquet columns, and root
attributes, provided they do not contradict or redefine required
mzZarr 1.0 fields.

Readers SHOULD ignore unknown optional metadata that does not affect
interpretation of required fields.

An incompatible change to required layout or semantics MUST NOT be
silently introduced under the unchanged version string `1.0`.
A future specification revision must explicitly define its version
identifier and compatibility rules.

This document does not prescribe a future version numbering scheme
beyond the exact identity required for v1.0.

## 11. Implementation independence and evidence

No PTERO package, Python module, proprietary vendor SDK, or
PTERO-specific root attribute is required to implement this format.

Independent implementations can use Zarr v3 and Parquet libraries to
construct and interpret conforming datasets.

The separately maintained `RELEASE_EVIDENCE_v1.0.md` describes the
verification available at the time of the specification freeze. That
evidence is not itself part of the normative format definition.

The existence of this specification does not imply endorsement by
HUPO-PSI, formal standards-body approval, FAIR certification, or
independently demonstrated interoperability across all languages,
vendors, and implementations.
