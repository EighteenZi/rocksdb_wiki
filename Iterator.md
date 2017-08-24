## Error Handling
`Iterator::status()` returns the error of the iterating. The errors include I/O errors, checksum mismatch, unsupported operations, internal errors, or other errors.

If there is no error, the status is `Status::OK()`. If the status is not OK, the iterator will be invalidated too. In another word, if `Iterator::Valid()` is true, `status()` is guaranteed to be `OK()` so it's safe to proceed other operations without checking status().

On the other hand, if `Iterator::Valid()` is false, there are two possibilities: (1) We reached the end of the data. In this case, `status()` is `OK()`; (2) there is an error. In this case `status()` is not `OK()`. It is always a good practice to check `status()` if the iterator is invalidated.

## Prefix Iterating
Prefix iterator allows users to use bloom filter or hash index in iterator, in order to improve the performance. However, the feature has limitation and may return wrong results without reporting an error if misused. So we recommend you to use this feature carefully. For how to use the feature, see [[Prefix Seek|Prefix Seek API Changes]]


Coming soon...