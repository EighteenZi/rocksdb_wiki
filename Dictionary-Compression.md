## Purpose

Dynamic dictionary compression algorithms have trouble compressing small data. With default usage, the compression dictionary starts off empty and is constructed during a single pass over the input data. Thus small input leads to a small dictionary, which cannot achieve good compression ratios.

In RocksDB, each block in a block-based table (SST file) is compressed individually. Block size defaults to 4KB, from which the compression algorithm cannot build a sizable dictionary. The dictionary compression feature presets the dictionary with data sampled from multiple blocks, which improves compression ratio when there are repetitions across blocks.

## Usage

Set `rocksdb::CompressionOptions::max_dict_bytes` (in `include/rocksdb/options.h`) to a nonzero value indicating the maximum per-file dictionary size.

## Implementation

The dictionary will be constructed by sampling the first output file in a subcompaction when the target level is bottommost. Samples are 64 bytes each and taken uniformly/randomly over the file. When picking sample intervals, we assume the output file will reach its maximum possible size. If not, some of the sample intervals will lie outside the file's data, thus they will not be included in the dictionary and its size will be less than `max_dict_bytes`.

Once generated, this dictionary will be loaded into the compression library before compressing/uncompressing each data block of subsequent files in the subcompaction. The dictionary is stored in the file's meta-block in order for it to be known when uncompressing.

## Limitations

* Uniform random sampling is not optimal (see https://github.com/facebook/rocksdb/pull/1835)
* Requires repetitions across files since we generate dictionary from subcompaction's first file and only apply it to later files