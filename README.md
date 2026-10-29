# mzZarr

**mzZarr v1.0** is a proposed vendor-independent mass spectrometry data format based on Zarr v3 and Apache Parquet.

## Specification

Read the [mzZarr v1.0 Format Specification](SPECIFICATION.md).

Version: 1.0

Status: Normative specification, frozen 29 October 2026.

## Overview

mzZarr separates numerical peak arrays from per-spectrum metadata to support selective access to mass spectrometry data.

The format defines required arrays for mass-to-charge values, intensities, and spectrum offsets, together with a Parquet metadata table. Optional ion mobility information is supported with explicit quantity and unit declarations.

## Reference implementation

[PTERO Desktop](https://ayehblk.github.io/ptero-website/) provides a Python and Windows reference implementation.

PTERO is not required to read or write mzZarr.

## Standardization status

mzZarr is a proposed format and is not formally endorsed by HUPO-PSI.

Independent cross-language interoperability has not yet been demonstrated.

## Version and integrity

mzZarr v1.0 was frozen on 29 October 2026. The versioned specification is available in this repository under the `v1.0` Git tag.

The SHA-256 checksum of `SPECIFICATION.md` is:

`9A6A5572B028B9DA4B6D40817D4D8448E15F8DF0EFCA00E6232BC6A2CC3C9F1D`
