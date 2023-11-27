# Filter Lists Differential Updates

## Background

A "filter list" is a fundamental component utilized by ad blockers. Essentially, a filter list is a set of rules and patterns designed to identify and subsequently block unwanted content like advertisements, trackers, and malicious websites. When an individual surfs the internet using a browser with an ad blocker enabled, this filter list is referenced to determine which elements on a webpage should be blocked or allowed. These lists are maintained by communities and organizations, and they are regularly updated to remain effective against the ever-evolving landscape of online advertisements and trackers.

## Problem

The core issue revolves around the mechanism by which these filter lists are updated. As it stands, when there's a modification to a filter list, even if it's just a small change, the entire list needs to be redownloaded by the end user. This approach is inefficient for several reasons:

* Bandwidth Consumption: Continuously downloading the entire list, especially for minor updates, consumes unnecessary bandwidth.
* Latency: Each full download requires more time than if only the changes were fetched.
* Server Load: Hosting servers experience an unnecessary load.

## Solution

In essence, instead of fetching the entire filter list every time an update is available, users would only download the changes made since their last update (the 'diff' or difference). This would be achieved by employing a diff algorithm on the server side which identifies the changes between the current and previous versions of the filter list.

This approach significantly reduces bandwidth consumption, minimizes latency, and decreases server load, resulting in a more efficient and user-friendly experience.

### Changes To Filter Lists Metadata

#### `! Diff-Path`

In order to use the differential update mechanism we propose to add one new field to the filter list metadata: `Diff-Path`.

This field will provide the relative path where the differential file (diff) for the filter list can be found. This differential file will take the user from their current version of the filter list to the next version. Crucially, within this differential update, the `Diff-Path` field will be updated to point to the subsequent version's diff. This ensures that the ad blocker knows where to find the next differential update.

`Diff-Path` also encodes additional information in the file name:

```adblock
${patchName}[-${resolution}]-${epochTimestamp}-${expirationPeriod}.patch[#${resourceName}]
```

* `patchName` - name of the patch file, an arbitrary string to identify the patch.
* `epochTimestamp` - epoch timestamp when the patch was generated (the unit of that timestamp depends on `resolution`, see below).
* `expirationPeriod` - expiration time for the diff update (the unit depends on `resolution`, see below).
* `resolution` - is an optional field, that specifies the resolution for both `expirationPeriod` and `epochTimestamp`. It can be either `h` (hours), `m` (minutes) or `s` (seconds). If `resolution` is not specified, it is assumed to be `h`.
* `resourceName` - a name of the resource that is being patched. This is used to support batch updates, see [Batch Updates](#batch-updates) section for more details.

The following limitations are imposed on the `Diff-Path`:

* `Diff-Path` MUST be a relative path to the filter list file, i.e. `/list-472234-1.patch` or `../list-472234-1.patch` or similar.
* `Diff-Path` is a mandatory field for enabling the differential updates mechanism.
* `Diff-Path` MUST point to a file name with the name format conforming to the format described above. If the file name is different, the field is considered invalid and the differential update mechanism is disabled for the filter list.
* `patchName` MUST be a string of length 1-64 with no spaces or other special characters. Validation regex: `[a-zA-Z0-9_.]{1,64}`.
* `epochTimestamp` MUST be a valid epoch timestamp (considering the unit specified in the `resolution` field).
* `expirationPeriod` MUST be a positive integer.
* `resourceName` is an optional part, it's explained in the [Resource name](#resource-name) section.

#### Examples

* `list1_v1.0.0-m-28334180-60.patch#list1`
  * Patch name is `list1_v1.0.0`.
  * Resolution is set to `m` (minutes).
  * Timestamp is `28334180` minutes from epoch, i.e. `Wed, 15 Nov 2023 12:20:00 GMT`.
  * Expiration period is `60` minutes.
  * Resource name is set to `list1`.
* `list1_v1.0.0-472236-1.patch`
  * Patch name is `list1_v1.0.0`.
  * Resolution is not specified so it is assumed to be `h` (hours).
  * Timestamp is `472236` hours from epoch, i.e. `Wed, 15 Nov 2023 12:00:00 GMT`.
  * Expiration period is `1` hour.
  * Resource name is not specified, i.e. the patch does not support batch updates.

#### Resource name

If a list supports batch updates, the `Diff-Path` MUST also have a "hash" part, i.e. `/path.patch#resourceName`. This "hash" is the name of the resource to be patched. In this case, the ad blocker will only download the diff file once and then apply it to all lists that are specified in the diff file. See the [Batch Updates](#batch-updates) section for more details.

Later in the document it will be referred as "resource name".

* This part is only mandatory when the filter list supports batch differential updates.
* The "hash" part of the URL must be a string of length 1-64. Validation regex: `[a-zA-Z0-9-_]{1,64}`.
* When specified, `diff name` directive in the diff file MUST match the resource name, see [Diff Files Format](#diff-files-format) for more details.

#### `! Expires:`

`Expires` continues to work as it was working before, i.e. once in a while the ad blocker will do the so-called **"full sync"**. When differential updates are available it is recommended to increase the value of `Expires` to a large value, e.g. `10 days`. This will ensure that the ad blocker will not do the full sync too often.

#### Diff Files Format

We propose using the [RCS format](https://www.gnu.org/software/diffutils/manual/diffutils.html#RCS) for the diff files. This format is widely used in the software development industry and is well documented. It is also supported by the `patch` utility which is available on most operating systems.

In order to support batch updates and be able to validate patch result, the standard format is extended with the `diff` directive:

`diff name:[name] checksum:[checksum] lines:[lines]`

* `name` - name of a corresponding filter list. It is only mandatory when [resource name](#resource-name) is specified in the list.
* `checksum` - the expected SHA1 checksum of the file after the patch is applied. This is used to validate the patch.
* `lines` - the number of lines that follow that make up the RCS diff block. Note, that `lines` are counted using the same algorithm as used by `wc -l`, i.e. it basically counts `\n`.

`diff` directive is optional. If it is not specified, the patch is applied without validation.

Note, that it is possible to extend the `diff` directive with additional fields not specified in the spec. The implementation should be able to ignore unknown fields.

> It is recommended to use the `diff checksum:` directive to validate the patching result. This will ensure that the patch is applied correctly and the resulting file is not corrupted.

### Algorithm

#### 1. Check for Update

1. Refer to the `Diff-Path` to see if a differential update is available.
    * If there are several lists with the same `Diff-Path`, download the diff file only once. Refer to the [Batch Updates](#batch-updates) section for the details on how batch patches are applied.
    * Calculate the patch expiration date: `(${epochTimestamp} + ${expirationPeriod}) * ${resolution}`. If the expiration date in the past, the patch is considered expired and the ad blocker SHOULD attempt to download the update.
1. If the differential update is available, download and apply it to the current filter list.
    * If the differential update is not empty and applied, the `Diff-Path` within the list MUST be updated to point to the next differential update.
    * At this point the ad blocker may decide to wait for a while before checking for the next differential update (see [2. Set Update Timer](#2-set-update-timer)) to ensure that the server is not overloaded.
1. If the differential update is not available the server may signal about that by returning one of the following responses:
    * `404 Not Found`
    * `200 OK` with empty content (content length is 0)
    * `204 No Content`

    In this case the ad blocker SHOULD wait for a while and then try again, see [2. Set Update Timer](#2-set-update-timer).

#### 2. Set Update Timer

The update timer depends on the previous update check result.

1. If the differential update was not empty and applied successfully, the ad blocker SHOULD check the new `Diff-Path` file expiration time.
    * If the expiration time is in the future, the ad blocker SHOULD wait until that time and then check for the update again.
    * If the expiration time is in the past, the ad blocker SHOULD try to download the new patch and apply it.
1. If the differential update was empty and the list's `Diff-Path` stayed the same, the ad blocker SHOULD delay the next update for at least 30 minutes to avoid overloading the server.

#### 3. Fallback Mechanism

Any unexpected error during the update process SHOULD be treated as a fatal error and the ad blocker should wait until it is time for the full sync. Note, that it should respect the `Expires` value set by the filter list.

### Batch Updates

The mechanism allows having a single diff file for multiple filter lists. In order to achieve this, the [resource name](#resource-name) MUST be specified for each filter list that supports batch differential updates. The [resource name](#resource-name) is then used to match a filter list with its corresponding patch in the diff file. This is achieved by using the `diff name:` directive in the diff file which links a patch to a filter list.

* The list that is getting patched MUST have this exact patch file specified in `Diff-Path`.
* If a filter list specified inside the batch patch is not installed in the ad blocker, the patch for this file SHOULD be ignored.
* At least one filter list MUST be updated in result of applying the batch patch. Otherwise, the patch should be considered invalid, see [Fallback Mechanism](#3-fallback-mechanism).

#### Example

Let's take an example:

* List 1.

  List URL is `https://example.com/list1/list1.txt`.

    ```adblock
    ! Title: List 1
    ! Diff-Path: ../patches/batch-m-28334120-60.patch#list1
    ```
  
  * `Diff-Path` is relative to `list1.txt` location so the final URL of the diff file will be `https://example.com/patches/batch.patch`.
  * [Resource name](#resource-name) is set to `list1` here. It is mandatory for lists that support batch differential updates.
  * Patch expiration period is set to `60` minutes, creation time is set to `28334120` (Unix timestamp in minutes).

* List 2

  List URL is `https://example.com/list2/list2.txt`.

    ```adblock
    ! Title: List 2
    ! Diff-Path: ../patches/batch-m-28334120-60.patch#list2
    ```

  * `Diff-Path` is relative to `list2.txt` location so the final URL of the diff file will be `https://example.com/patches/batch.patch`.
  * [Resource name](#resource-name) is set to `list2` here. It is mandatory for lists that support batch differential updates.
  * Patch expiration period is set to `60` minutes, creation time is set to `28334120` (Unix timestamp in minutes).
  
* `batch-28334120-60.patch`

  A file that contains patches for both `list1.txt` and `list2.txt`. It uses the `diff name:` directive to point at which patch should be applied to which list.

   ```diff
    diff name:list1 checksum:e3c9c883378dc2a3aec9f71578c849891243bc2c lines:3
    d2 1
    a2 1
    ! Diff-Path: patches/batchnew-m-28334180-60.patch#list1
    diff name:list2 checksum:be09384422b8d7f20da517d1245360125868f0b9 lines:3
    d2 1
    a2 1
    ! Diff-Path: patches/batchnew-m-28334180-60.patch#list2
    ```

### More Examples

Please find examples in the [examples directory](./examples/).
