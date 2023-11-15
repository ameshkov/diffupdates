# Diff Example With Validation

Example of a filter list with differential updates support where the patch files also contain the `diff` directive that can be used to validate the patching result.

* [filter_v1.0.0.txt](./filter_v1.0.0.txt) - the oldest version of the list.
* [patches/v1.0.0-m-28334060-60.patch](./patches/v1.0.0-m-28334060-60.patch) - a patch that provides differential update from `v1.0.0` to `v1.0.1`.
* [filter_v1.0.1.txt](./filter_v1.0.1.txt) - the next version of the list.
* [patches/v1.0.1-m-28334120-60.patch](./patches/v1.0.1-m-28334120-60.patch) - a patch that provides differential update from `v1.0.1` to `v1.0.2`.
* [filter.txt](./filter.txt) - the final version of the list. After all differential updates are applied you should get this version.
* [patches/v1.0.2-m-28334180-60.patch](./patches/v1.0.2-m-28334180-60.patch) - empty patch that signals that there's no patch available for the next update yet.

Note, that resolution is specified in the patch names (`m` for minutes).

## How patch files were prepared

The patches are created using the `diff` utility:

```shell
# Calculating the RFC diff for filter_v1.0.1.txt.
diff -n filter_v1.0.0.txt filter_v1.0.1.txt > patches/v1.0.0-m-28334060-60.patch

# Calc the SHA1 sum of filter_v1.0.1.txt and append it to the patch file.
FILENAME="filter_v1.0.1.txt" && \
PATCHFILE="patches/v1.0.0-m-28334060-60.patch" && \
SHASUM=$(shasum -a 1 $FILENAME | awk '{print $1}') && \
    NUMLINES=$(wc -l < $PATCHFILE | awk '{print $1}') && \
    echo "diff checksum:$SHASUM lines:$NUMLINES" | cat - $PATCHFILE > temp.patch && \
    mv temp.patch $PATCHFILE

# Calculating the RFC diff for filter.txt.
diff -n filter_v1.0.1.txt filter.txt > patches/v1.0.1-m-28334120-60.patch

# Calc the SHA1 sum of filter.txt and append it to the patch file.
FILENAME="filter.txt" && \
PATCHFILE="patches/v1.0.1-m-28334120-60.patch" && \
SHASUM=$(shasum -a 1 $FILENAME | awk '{print $1}') && \
    NUMLINES=$(wc -l $PATCHFILE | awk '{print $1}') && \
    echo "diff checksum:$SHASUM lines:$NUMLINES" | cat - $PATCHFILE > temp.patch && \
    mv temp.patch $PATCHFILE

# Make an empty patch file to signal that there's no patch available for the
# next update yet.
touch patches/v1.0.2-m-28334180-60.patch
```
