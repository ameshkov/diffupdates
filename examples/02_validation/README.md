# Diff Example With Validation

Example of a filter list with differential updates support where the diff files also contain the `f` directive that can be used to validate the patching result.

* [filter_v1.0.0.txt](./filter_v1.0.0.txt) - the oldest version of the list.
* [patches/v1.0.0.patch](./patches/v1.0.0.patch) - a patch that provides differential update from `v1.0.0` to `v1.0.1`.
* [filter_v1.0.1.txt](./filter_v1.0.1.txt) - the next version of the list.
* [patches/v1.0.1.patch](./patches/v1.0.0.patch) - a patch that provides differential update from `v1.0.1` to `v1.0.2`.
* [filter.txt](./filter.txt) - the final version of the list. After all differential updates are applied you should get this version.

## How patch files were prepared

The patches are created using the `diff` utility:

```shell
# Calculating the RFC diff for filter_v1.0.1.txt.
diff -n filter_v1.0.0.txt filter_v1.0.1.txt > patches/v1.0.0.patch

# Calc the SHA1 sum of filter_v1.0.1.txt and append it to the patch file.
FILENAME="filter_v1.0.1.txt" && \
PATCHFILE="patches/v1.0.0.patch" && \
SHASUM=$(shasum -a 1 $FILENAME | awk '{print $1}') && \
    NUMLINES=$(wc -l < $PATCHFILE | awk '{print $1}') && \
    echo "f filter.txt $SHASUM $NUMLINES" | cat - $PATCHFILE > temp.patch && \
    mv temp.patch $PATCHFILE

# Calculating the RFC diff for filter.txt.
diff -n filter_v1.0.1.txt filter.txt > patches/v1.0.1.patch

# Calc the SHA1 sum of filter.txt and append it to the patch file.
FILENAME="filter.txt" && \
PATCHFILE="patches/v1.0.1.patch" && \
SHASUM=$(shasum -a 1 $FILENAME | awk '{print $1}') && \
    NUMLINES=$(wc -l $PATCHFILE | awk '{print $1}') && \
    echo "f filter.txt $SHASUM $NUMLINES" | cat - $PATCHFILE > temp.patch && \
    mv temp.patch $PATCHFILE
```
