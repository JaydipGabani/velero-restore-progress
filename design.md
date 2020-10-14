# Restore progress update

Velero backup supports real-time update with total number of items that is going to be backed up and items that are already backed up. Velero Restore only provides state of the restore, `InProgress, Completed, PartiallyFailed, Failed`, as an update for restore. Restore workflow needs to have same support as backup and update the Restore CR in real-time with same information.

## Goals

- Enable progress reporting for Velero restore

## Non Goals



## Background



## High-Level Design

There are two approches to give out this update:

1. One phase approach (short term goal):
    - Get the total number of items from `Backup` CR
    - Maintain a counter for skipped items and restore item
    - Give out update when an item is skipped or restore in the form of:
    
            ```
            TotalItems: n
            Skipped: y
            Restored: x
            ```

2. Two Phase approach (long term goal):
    - Predetermine which resources are going to be restored before actual restore takes place
    - Give out an update when an item is restored in form of:
        ```
        TotalItem: n
        Restored: x
        ```
        This could be a similar implementation as Backup progress update.


## Detailed Design

`Bacup` CR contains information regarding totalItems present in `backup`, determined while backup is running. `Restore` CR has a referenece to corrosponding `Backup` CR. The information about total Items backed up can be extracted at restore with this reference. Out of these total items, not every item is going to be restored. Restoring items depends on the exclusion list and label/selectors mentioned at the restore. 

We propose to maintain two counters, one for skipped items and one for restored items. Weather an item or a resoure is going to be skipped or not is decided on the run. A counter can be maintained within [execute()](https://github.com/vmware-tanzu/velero/blob/e69fac153ba60dc5129cdda51480a64fbf47b851/pkg/restore/restore.go#L352) function to track the count when an item is skipped and when an item is restored. Update the `Restore` CR status field with message indicating `TotalItems`, `Skipped` , and `Restored` whenever an Item is restored.


