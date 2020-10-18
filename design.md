# Restore progress update

Velero _Backup_ resource provides real-time progress of an ongoing backup by means of a _Progress_ field in the CR. Velero _Restore_, on the other hand, only shows one of the phases (InProgress, Completed, PartiallyFailed, Failed) of the ongoing restore. In this document, we propose detailed progress reporting for Velero _Restore_. With the introduction of the proposed _Progress_ field, Velero _Restore_ CR will look like:

```yml
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: test-restore
  namespace: velero
spec:
    [...]
status:
  phase: InProgress
  progress:
    itemsRestored: 100
    totalItems: 140
```

## Goals

- Enable progress reporting for Velero restore

## Non Goals

- Estimate time to completion

## Background

The current _Restore_ CR lets users know whether a restore is in-progress or completed (failed/succeeded). While this basic piece of information about the restore is useful to the end user, there seems to be room for improvement in the user experience. The _Restore_ CR can show detailed progress in terms of the number of resources restored so far and the total number of resources to be restored. This will be particularly useful for restores that run for a longer duration of time. Such progress reporting already exists for Velero _Backup_. This document proposes similar implementation for Velero _Restore_.


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

This approach divides the restore process in two steps. The first step collects the number of all items to be restored from the backup tarball. It applies the label selector and include/exclude rules on the resources / items and stores all items (preserving the priority order) in an in-memory data structure. The second step reads the collected items and restores them. 

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

### Two pass approach

#### Progress struct

A new struct will be introduced to store progress information:

```go
type RestoreProgress struct {
    TotalItems    int `json:"totalItems,omitempty`
    ItemsRestored int `json:"itemsRestored,omitempty`
}
```

`RestoreStatus` will include the above struct:

```go
type RestoreStatus struct {
    [...]

    Progress *RestoreProgress `json:"progress,omitempty"`
}
```

#### Modifications to restore.go

Currently, the restore process works by looping through the resources in the backup tarball and restoring them one-by-one in the same pass:

```go
func (ctx *context) execute(...) {
    [...]

    for _, resource := range getOrderedResources(...) {
        [...]

        for namespace, items := range resourceList.ItemsByNamespace {
            [...]

            for _, item := range items {
                [...]

                // restore item here
                w, e := restoreItem(...)
            }
        }
    }
}
```

We propose to remove the call to `restoreItem()` in the inner most loop and instead store the item in a data structure. Once all the items are collected, we loop through the array of collected items and make a call to `restoreItem()`:

```go
func (ctx *context) execute(...) {
    [...]

    // get all items
    resources := ctx.getOrderedResourceCollection(...)

    for _, resource := range resources {
        [...]

        for _, items := range resource.itemsByNamespace {
            [...]

            for _, item := range items {
                [...]

                // restore the item
                w, e := restoreItem(...)
            }
        }
    }

    [...]
}

func (ctx *context) getOrderedResourceCollection(...) {
    for _, namespace := range getOrderedResources(...) {
        [...]

        for namespace, items := range resourceList.ItemsByNamespace {
            [...]

            for _, item := range items {
                [...]

                // store item in a data structure
                collectedResources.itemsByNamespace[originalNamespace] = item
            }
        }
    }
    return collectedResources
}
```

We introduce two new structs to hold the collected items:

```go
type restoreResource struct {
    resource            string
    itemsByNamespace    map[string][]restoreItem
}
```

```go
type restoreItem struct {
    targetNamespace string
    name            string
}
```

Each group resource is represented by `restoreResource`. The map `itemsByNamespace` is indexed by `originalNamespace`, and the values are list of `items` in the original namespace. Each item represented by `restoreItem` has `name` and the resolved `targetNamespace`.
