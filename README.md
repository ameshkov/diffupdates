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

In order to use the differential update mechanism, we propose two new metadata fields.

#### `! Diff-Path:`

This field will provide the relative path where the differential file (diff) for the filter list can be found. This differential file will take the user from their current version of the filter list to the next version. Crucially, within this differential update, the `Diff-Path` field will be updated to point to the subsequent version's diff. This ensures that the ad blocker knows where to find the next differential update.

* `Diff-Path` must be a relative path to the filter list file.
* `Diff-Path` is a mandatory field for enabling the differential updates mechanism.

#### `! Diff-Expires:`

This is essentially a time-to-live (TTL) for the differential update. It dictates the frequency with which the ad blocker should attempt to fetch the differential update. For instance, if the `Diff-Expires` is set to `1 hour`, it means the ad blocker should attempt to download the differential update once every hour.

`! Diff-Expires:` is an optional field. If it is not set, the ad blocker may fallback to `! Expires:` or to some pre-defined default value. It is recommended to have it specified to avoid inconsistency between different ad blockers.

#### `! Expires:`

`Expires` continues to work as it was working before, i.e. once in a while AdGuard will do the so-called **"full sync"**. When differential updates are available it is recommended to increase the value of `Expires` to a large value, e.g. `10 days`. This will ensure that the ad blocker will not do the full sync too often.

#### Diff Files Format

We propose using the [RCS format](https://www.gnu.org/software/diffutils/manual/diffutils.html#RCS) for the diff files. This format is widely used in the software development industry and is well documented. It is also supported by the `patch` utility which is available on most operating systems.

In order to support batch updates and validation of the result, the format is extended with the `f` directive:
`f [filename] [sha1sum] [numlines]`

* `filename` - a name or a relative path of the file to be patched. See batch updates below for more information on this.
* `sha1sum` - the expected sha1sum of the file after the patch is applied. This is used to validate the patch.
* `numlines` - the number of lines that follow that make up the RCS diff block. Note, that `numlines` are counted using the same algorithm as used by `wc -l`, i.e. it basically counts `\n`.

**IMPORTANT** The `f` directive is optional.

### Algorithm

#### Check for Update

1. Refer to the `Diff-Path` to see if a differential update is available.
1. If the differential update is available, download and apply it to the current filter list.
    * Once the differential update is applied, the `Diff-Path` within the list will be updated to point to the next differential update.
    * At this point the ad blocker may decide either to wait for the `Diff-Expires` period and then try again or to immediately try to fetch the next differential update.
1. If the differential update is not available, i.e. the server returns `404 Not Found`, then the ad blocker wait for the `Diff-Expires` period and then try again.

#### Set Update Timer

Using the `Diff-Expires` value, set a timer for the next update check. For example, if it's set to `1 hour`, the ad blocker will wait for that duration before checking the `Diff-Path` again.

#### Fallback Mechanism

Any unexpected error during the update process should be treated as a fatal error and the ad blocker should wait until it is time for the full sync. Note, that it should respect the `Expires` value set by the filter list.

### Batch Updates

The mechanism allows having a single diff file for multiple filter lists. This is achieved by using the `f` directive in the diff file. The `f` directive is used to specify the name of the file to be patched. The ad blocker should apply the patch only if the name of the file matches the name of the filter list.

* There is one important limitation: all filter lists that are updated using the same patch file MUST be hosted on the same server.
* If a file specified inside the batch patch is not installed on the user's device, the patch for this file should be ignored.

Let's take an example:

* List 1.

  List URL is `https://example.com/list1/list1.txt`.

    ```adblock
    ! Title: List 1
    ! Diff-Path: ../patches/batch.patch
    ```
  
  `Diff-Path` is relative to `list1.txt` location so the final URL of the diff file will be `https://example.com/patches/batch.patch`.

* List 2

  List URL is `https://example.com/list2/list2.txt`.

    ```adblock
    ! Title: List 2
    ! Diff-Path: ../patches/batch.patch
    ```

  `Diff-Path` is relative to `list2.txt` location so the final URL of the diff file will be `https://example.com/patches/batch.patch`.

* `batch.patch`

  A file that contains patches for both `list1.txt` and `list2.txt`. The `f` directive allows applying this patch to both `list1.txt` and `list2.txt`. Note, that the file path in the `f` directive is relative to the patch file location.

    ```diff
    f ../list1/list1.txt [checksum] 3
    d2 1
    a2 1
    ! Diff-Path: patches/batch_new.patch
    f ../list2/list2.txt [checksum] 3
    d2 1
    a2 1
    ! Diff-Path: patches/batch_new.patch
    ```

### Examples

Please find examples in the [examples directory](./examples/).
