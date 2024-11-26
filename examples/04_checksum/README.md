# Checksum Example

A filter list with differential updates support and a checksum attribute. The filter lists in this example are
outfitted with a `Checksum:` attribute which enables checking for download and patch application correctness.

* [filter_v1.0.0.txt](./filter_v1.0.0.txt) - the oldest version of the list.
* [patches/v1.0.0-472234-1.patch](./patches/v1.0.0-472234-1.patch) - a patch that provides a differential update from `v1.0.0` to `v1.0.1`.
* [filter.txt](./filter.txt) - the final version of the list. After all differential updates are applied you should get this version.

## How the example files were prepared

* The checksum is computed thusly:
    ```
    cat filter_v1.0.0.txt | grep -v "! Checksum:" | awk "NF{$1=$1;print}" | head -c -1 | openssl md5 -binary | openssl base64 | sed -e "s/=*$//"
    cat filter.txt | grep -v "! Checksum:" | awk "NF{$1=$1;print}" | head -c -1 | openssl md5 -binary | openssl base64 | sed -e "s/=*$//"
    ```
    That is, the line with the checksum itself, empty lines, and all leading and trailing whitespace, as well as the newline character and the end of file, is discarded before computing the checksum.
    The checksum is the MD5 digest of the input, encoded in standard Base64 without padding characters.
    The line `! Checksum: <checksum>` is inserted near the beginning of the filter list.

* The patches are created using the `diff` utility:
    ```shell
    diff -n filter_v1.0.0.txt filter.txt > patches/v1.0.0-472234-1.patch
    ```
