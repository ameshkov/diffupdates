# Filter Lists Differential Updates

## Background

A "filter list" is a fundamental component utilized by ad blockers. Essentially, a filter list is a set of rules and patterns designed to identify and subsequently block unwanted content like advertisements, trackers, and malicious websites. When an individual surfs the internet using a browser with an ad blocker enabled, this filter list is referenced to determine which elements on a webpage should be blocked or allowed. These lists are maintained by communities and organizations, and they are regularly updated to remain effective against the ever-evolving landscape of online advertisements and trackers.

## Problem

The core issue revolves around the mechanism by which these filter lists are updated. As it stands, when there's a modification to a filter list, even if it's just a small change, the entire list needs to be redownloaded by the end user. This approach is inefficient for several reasons:

* Bandwidth Consumption: Continuously downloading the entire list, especially for minor updates, consumes unnecessary bandwidth.
* Latency: Each full download requires more time than if only the changes were fetched.
* Server Load: Hosting servers experience an unnecessary load

## Solution

In essence, instead of fetching the entire filter list every time an update is available, users would only download the changes made since their last update (the 'diff' or difference). This would be achieved by employing a diff algorithm on the server side which identifies the changes between the current and previous versions of the filter list.

This approach significantly reduces bandwidth consumption, minimizes latency, and decreases server load, resulting in a more efficient and user-friendly experience.

### Changes To Filter Lists Metadata

In order to use the differential update mechanism, we propose several new metadata fields.

#### `! Diff-Path:`

This field will provide the relative path where the differential file (diff) for the filter list can be found. This differential file will take the user from their current version of the filter list to the next version. Crucially, within this differential update, the `Diff-Path` field will be updated to point to the subsequent version's diff. This ensures that the ad blocker knows where to find the next differential update.

* `Diff-Path` must be a relative path to the filter list file.
* `Diff-Path` is a mandatory field for enabling the differential updates mechanism.

#### `! Diff-Name:`

This field is only mandatory for filter lists that support batch differential updates. It specifies the name of the resource to be patched. See the [Batch Updates](#batch-updates) section for more details.

* `Diff-Name` must be a string of length 1-64. Validation regex: `[a-zA-Z0-9-_ ]{1,64}`.
* `Diff-Name` is only mandatory when the filter list supports batch differential updates. In all other cases it is ignored.

#### `! Diff-Expires:`

This is essentially a time-to-live (TTL) for the differential update. It dictates the frequency with which the ad blocker should attempt to fetch the differential update. For instance, if the `Diff-Expires` is set to `1 hour`, it means the ad blocker should attempt to download the differential update once every hour.

`! Diff-Expires:` is an optional field. If it is not set, the ad blocker may fallback to `! Expires:` or to some pre-defined default value. It is recommended to have it specified to avoid inconsistency between different ad blockers.

#### `! Expires:`

`Expires` continues to work as it was working before, i.e. once in a while AdGuard will do the so-called **"full sync"**. When differential updates are available it is recommended to increase the value of `Expires` to a large value, e.g. `10 days`. This will ensure that the ad blocker will not do the full sync too often.

#### Diff Files Format

We propose using the [RCS format](https://www.gnu.org/software/diffutils/manual/diffutils.html#RCS) for the diff files. This format is widely used in the software development industry and is well documented. It is also supported by the `patch` utility which is available on most operating systems.

In order to support batch updates and be able to validate patch result, the standard format is extended with the `diff` directive:

`diff name:[name] checksum:[checksum] lines:[lines]`

* `name` - name of a corresponding filter list (see `Diff-Name`).
* `checksum` - the expected SHA1 checksum of the file after the patch is applied. This is used to validate the patch.
* `lines` - the number of lines that follow that make up the RCS diff block. Note, that `lines` are counted using the same algorithm as used by `wc -l`, i.e. it basically counts `\n`.

`diff` directive is optional. If it is not specified, the patch is applied without validation.

> It is recommended to use the `diff checksum:` directive to validate the patching result. This will ensure that the patch is applied correctly and the resulting file is not corrupted.

### Algorithm

#### 1. Check for Update

1. Refer to the `Diff-Path` to see if a differential update is available.
    * If there are several lists with the same `Diff-Path`, download the diff file only once. Refer to the [Batch Updates](#batch-updates) section for the details on how batch patches are applied.
1. If the differential update is available, download and apply it to the current filter list.
    * Once the differential update is applied, the `Diff-Path` within the list will be updated to point to the next differential update.
    * At this point the ad blocker may decide either to wait for the `Diff-Expires` period and then try again or to immediately try to fetch the next differential update.
1. If the differential update is not available the server may signal about that by returning one of the following responses:
    * `404 Not Found`
    * `200 OK` with empty content (content length is 0)
    * `204 No Content`

    In this case the ad blocker wait for the `Diff-Expires` period and then try again.

#### 2. Set Update Timer

Using the `Diff-Expires` value, set a timer for the next update check. For example, if it's set to `1 hour`, the ad blocker will wait for that duration before checking the `Diff-Path` again.

#### 3. Fallback Mechanism

Any unexpected error during the update process should be treated as a fatal error and the ad blocker should wait until it is time for the full sync. Note, that it should respect the `Expires` value set by the filter list.

### Batch Updates

The mechanism allows having a single diff file for multiple filter lists. In order to achieve this, the `Diff-Name` field MUST be specified for each filter list that supports batch differential updates. The `Diff-Name` field is then used to match a filter list with its corresponding patch in the diff file. This is achieved by using the `diff name:` directive in the diff file which links a patch to a filter list.

* The list that is getting patched MUST have this exact patch file specified in `Diff-Path`.
* If a filter list specified inside the batch patch is not installed in the ad blocker, the patch for this file SHOULD be ignored.
* At least one filter list MUST be updated in result of applying the batch patch. Otherwise, the patch should be considered invalid, see [Fallback Mechanism](#3-fallback-mechanism).

#### Example

Let's take an example:

* List 1.

  List URL is `https://example.com/list1/list1.txt`.

    ```adblock
    ! Title: List 1
    ! Diff-Path: ../patches/batch.patch
    ! Diff-Name: list1
    ```
  
  * `Diff-Path` is relative to `list1.txt` location so the final URL of the diff file will be `https://example.com/patches/batch.patch`.
  * `Diff-Name` is mandatory for lists that support batch differential updates.

* List 2

  List URL is `https://example.com/list2/list2.txt`.

    ```adblock
    ! Title: List 2
    ! Diff-Path: ../patches/batch.patch
    ! Diff-Name: list2
    ```

  * `Diff-Path` is relative to `list2.txt` location so the final URL of the diff file will be `https://example.com/patches/batch.patch`.
  * `Diff-Name` is mandatory for lists that support batch differential updates.

* `batch.patch`

  A file that contains patches for both `list1.txt` and `list2.txt`. It uses the `diff name:` directive to point at which patch should be applied to which list.

    ```diff
    diff name:list1 checksum:e3c9c883378dc2a3aec9f71578c849891243bc2c lines:3
    d2 1
    a2 1
    ! Diff-Path: patches/batch_new.patch
    diff name:list2 checksum:be09384422b8d7f20da517d1245360125868f0b9 lines:3
    d2 1
    a2 1
    ! Diff-Path: patches/batch_new.patch
    ```

### Examples

Please find examples in the [examples directory](./examples/).
