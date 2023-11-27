# Simple Diff Example

A filter list with differential updates support.

* [filter_v1.0.0.txt](./filter_v1.0.0.txt) - the oldest version of the list.
* [patches/v1.0.0-472234-1.patch](./patches/v1.0.0-472234-1.patch) - a patch that provides differential update from `v1.0.0` to `v1.0.1`.
  Expiration period is set to `1` hour, creation time is set to `472234` (Unix timestamp in hours, i.e. `Wed, 15 Nov 2023 10:00:00 GMT`).
* [filter_v1.0.1.txt](./filter_v1.0.1.txt) - the next version of the list.
* [patches/v1.0.1-472235-1.patch](./patches/v1.0.0-472234-1.patch) - a patch that provides differential update from `v1.0.1` to `v1.0.2`.
  Expiration period is set to `1` hour, creation time is set to `472235` (Unix timestamp in hours, i.e. `Wed, 15 Nov 2023 11:00:00 GMT`).
* [filter.txt](./filter.txt) - the final version of the list. After all differential updates are applied you should get this version.

## How patch files were prepared

The patches are created using the `diff` utility:

```shell
diff -n filter_v1.0.0.txt filter_v1.0.1.txt > patches/v1.0.0-472234-1.patch
diff -n filter_v1.0.1.txt filter.txt > patches/v1.0.1-472235-1.patch
```
