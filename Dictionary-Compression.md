## Purpose

Dynamic dictionary compression algorithms have trouble compressing small data. With default usage, the compression library's dictionary starts off empty and is constructed during a single pass over the input data. Thus small input leads to a small dictionary, which cannot achieve good compression ratios.

In RocksDB, each block in a block-based table (SST file) is compressed individually. Block size defaults to 4KB, from which the compression library cannot build a sizable dictionary. This feature presets the library's dictionary with data sampled from multiple blocks, which improves compression ratio when patterns exist across blocks.

## Usage

## Implementation

## Future