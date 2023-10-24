# Batch Update Example

Example of several filter lists that are updated in one batch as they both are:

1. Downloaded from the same location.
2. Have `Diff-Path` pointing to the same patch file.

## How patch files were prepared

The patches are created using the `diff` utility.

First, for lists `v1.0.0` -> `v1.0.1`.

```shell
# Calculating the RFC diff for list1_v1.0.1.txt.
diff -n list1/list1_v1.0.0.txt list1/list1_v1.0.1.txt > patches/list1_v1.0.0.patch

# Calc the SHA1 sum of list1_v1.0.1.txt and prepend it to the patch file.
FILENAME="list1/list1_v1.0.1.txt" && \
PATCHFILE="patches/list1_v1.0.0.patch" && \
SHASUM=$(shasum -a 1 $FILENAME | awk '{print $1}') && \
    NUMLINES=$(wc -l < $PATCHFILE | awk '{print $1}') && \
    echo "f ../list1/list1.txt $SHASUM $NUMLINES" | cat - $PATCHFILE > temp.patch && \
    mv temp.patch $PATCHFILE

# Calculating the RFC diff for list2_v1.0.1.txt.
diff -n list2/list2_v1.0.0.txt list2/list2_v1.0.1.txt > patches/list2_v1.0.0.patch

# Calc the SHA1 sum of list1_v1.0.1.txt and prepend it to the patch file.
FILENAME="list2/list2_v1.0.1.txt" && \
PATCHFILE="patches/list2_v1.0.0.patch" && \
SHASUM=$(shasum -a 1 $FILENAME | awk '{print $1}') && \
    NUMLINES=$(wc -l < $PATCHFILE | awk '{print $1}') && \
    echo "f ../list2/list2.txt $SHASUM $NUMLINES" | cat - $PATCHFILE > temp.patch && \
    mv temp.patch $PATCHFILE

# Concatenate files into a batch diff file.
cat patches/list1_v1.0.0.patch patches/list2_v1.0.0.patch > patches/batch_v1.0.0.patch
```

Second, for lists `v1.0.1` -> `v1.0.2`.

```shell
# Calculating the RFC diff for list1.txt.
diff -n list1/list1_v1.0.1.txt list1/list1.txt > patches/list1_v1.0.1.patch

# Calc the SHA1 sum of list1_v1.0.1.txt and prepend it to the patch file.
FILENAME="list1/list1.txt" && \
PATCHFILE="patches/list1_v1.0.1.patch" && \
SHASUM=$(shasum -a 1 $FILENAME | awk '{print $1}') && \
    NUMLINES=$(wc -l < $PATCHFILE | awk '{print $1}') && \
    echo "f ../list1/list1.txt $SHASUM $NUMLINES" | cat - $PATCHFILE > temp.patch && \
    mv temp.patch $PATCHFILE

# Calculating the RFC diff for list2_v1.0.1.txt.
diff -n list2/list2_v1.0.1.txt list2/list2.txt > patches/list2_v1.0.1.patch

# Calc the SHA1 sum of list1_v1.0.1.txt and prepend it to the patch file.
FILENAME="list2/list2.txt" && \
PATCHFILE="patches/list2_v1.0.1.patch" && \
SHASUM=$(shasum -a 1 $FILENAME | awk '{print $1}') && \
    NUMLINES=$(wc -l < $PATCHFILE | awk '{print $1}') && \
    echo "f ../list2/list2.txt $SHASUM $NUMLINES" | cat - $PATCHFILE > temp.patch && \
    mv temp.patch $PATCHFILE

# Concatenate files into a batch diff file.
cat patches/list1_v1.0.1.patch patches/list2_v1.0.1.patch > patches/batch_v1.0.1.patch
```
