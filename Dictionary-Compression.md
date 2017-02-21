## Purpose

Dynamic dictionary compression algorithms have trouble compressing small data. With default usage, the dictionary starts off empty and is constructed during a single pass over the input data. Thus small input leads to a small dictionary, which cannot achieve good compression ratios.

In RocksDB, each block in the block-based table is compressed individually. Dictionary compression presets the compression library's internal state with patterns seen across blocks, which improves compression ratio particularly for smaller block sizes.

## Usage

## Implementation

## Future