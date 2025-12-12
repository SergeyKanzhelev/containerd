# Proposal: A `meta-overlayfs` default snapshotter

## Goals

This document proposes the creation of a new, built-in snapshotter, `meta-overlayfs`, and its adoption as the default snapshotter for containerd.

The `meta-overlayfs` snapshotter is designed to orchestrate layer management across multiple sources, providing a flexible and powerful architecture. It acts as a federation layer over three distinct types of "child" backends:
1.  **Snapshotter Backends:** Lazy-pulling snapshotters like `stargz-snapshotter`.
2.  **Cache Directory Backends:** A managed directory for hydrated layers, written to by other processes (e.g., `stargz`) but garbage collected by `meta-overlayfs`.
3.  **Directory Backends:** A strictly read-only directory of layers (e.g., a mounted disk of "golden" images) that is never written to or garbage collected by containerd.

This hybrid, three-tiered approach provides fast container startup, long-term reliability for running workloads, and the ability to securely share base images.

## Configuration Changes

The `meta-overlayfs` will become the default snapshotter. A user can configure it with a prioritized list of children of any type.

```toml
# Configure the stargz-snapshotter to enable layer hydration
[plugins."io.containerd.snapshotter.v1.stargz"]
  # The root for stargz's internal metadata and virtual filesystem mounts.
  root = "/var/lib/containerd/io.containerd.snapshotter.v1.stargz"
  # This new option tells stargz where to place fully hydrated layers.
  hydration_dir = "/mnt/shared-cache/snapshots"

# Configure the primary, default meta snapshotter
[plugins."io.containerd.snapshotter.v1.meta-overlayfs"]
  # This is the 'main' root for meta-overlayfs. It hosts its own writable
  # layers and any newly pulled layers that are not found in the children.
  root = "/var/lib/containerd/io.containerd.snapshotter.v1.meta-overlayfs"

  # List of child backends, prioritized by order.
  children = [
    # A read-only source of golden images. Never garbage collected.
    { type = "directory", path = "/mnt/golden-images/snapshots" },
    # The managed cache dir is checked next for durable, hydrated layers.
    { type = "cache_dir", path = "/mnt/shared-cache/snapshots" },
    # Stargz is the final source for lazy-pulled layers.
    { type = "snapshotter", name = "stargz" }
  ]
```

## Implementation Details

### Layer Discovery and Mounting (`Prepare`, `Mounts`)

The layer search order is determined by the `children` array:
1.  **`meta-overlayfs` root:** Checks its own local storage first.
2.  **Children Backends:** Iterates through the `children` array in order. For each child, it performs a lookup based on its `type`:
    *   **`directory` / `cache_dir`:** Scans the filesystem at the configured `path`.
    *   **`snapshotter`:** Calls the child plugin's `Mounts()` method.

### Garbage Collection (`Remove`)

The `meta-overlayfs` snapshotter's GC behavior is type-dependent:
1.  It deletes the layer from its own `root` directory.
2.  If a child of type **`cache_dir`** is configured, it deletes the layer from that directory.
3.  It broadcasts a `Remove(key)` call to all **`snapshotter`** children, ignoring `errdefs.ErrNotFound` errors.
4.  It **never** attempts to delete layers from a child of type **`directory`**.

## Child Snapshotter Design and Compatibility

### Standard Interface Implementation

Child snapshotters are not a special class of plugin; they are complete, functional snapshotters that can be used independently. As such, any snapshotter configured as a `snapshotter` child **must** implement the standard `containerd.io/snapshotter/v1.Snapshotter` interface in its entirety. This ensures that a child snapshotter can be fully validated on its own and can be used as a primary snapshotter if needed, promoting modularity and testability.

### OverlayFS Compatibility

The `meta-overlayfs` snapshotter works by aggregating directory paths into the `lowerdir` option of a final `overlayfs` mount. Consequently, it is only compatible with child snapshotters that can expose their layers as a directory on a filesystem. Snapshotters that rely on different storage mechanisms (e.g., `btrfs` subvolumes, `zfs` datasets) cannot be used as children because they do not provide a simple directory path that `overlayfs` can consume.

### Compatibility Detection Mechanism

The contract for `overlayfs` compatibility is defined by the behavior of the `Mounts()` method:
- The `meta-overlayfs` snapshotter will call `child.Mounts(ctx, key)` on a child snapshotter.
- It will then expect the `Source` field of the first `mount.Mount` object in the returned slice (`mounts[0].Source`) to be an absolute path to a directory on the local filesystem.
- If the child snapshotter returns an incompatible `mount.Mount` object (e.g., one that does not point to a directory path suitable for `overlayfs`), `meta-overlayfs` will log a warning and effectively ignore that child for the current layer lookup. This allows for graceful degradation without crashing containerd.
- This path is then used directly as an entry in the `lowerdir` string.

Any snapshotter that upholds this contract when its `Mounts()` method is called is considered compatible and can be used as a `snapshotter` backend.

## Streamlined Hydration and Caching Workflow

This design enables a powerful and flexible workflow:

1.  **Initial Pull:** `meta-overlayfs` requests a layer. It checks for a golden image in the `directory` first. If not found, it checks the `cache_dir`. Finally, it falls back to `stargz` for a lazy-pull.
2.  **Background Hydration:** `stargz` hydrates the layer in the background and writes it to the shared `cache_dir`.
3.  **Subsequent Pulls:** New containers will find the hydrated layer in the `cache_dir` before falling back to `stargz`, ensuring they use the durable, local copy.
4.  **Cleanup:** When the layer is no longer in use, `meta-overlayfs.Remove()` deletes it from its `root` and the `cache_dir`, and notifies `stargz` to clean up its own metadata.

This approach creates a clear separation of concerns: `stargz` is responsible for fetching and hydrating, while `meta-overlayfs` is responsible for orchestrating lookups and managing the lifecycle of the cached layers.

## Configuring and Preparing Backends

### Snapshotter Backends

These are configured just like any other snapshotter and are expected to behave as fully functional, standalone snapshotters. For `stargz-snapshotter`, the user would follow its official documentation.

### Cache Directory Backends

This directory serves as the target for `stargz`'s `hydration_dir` option. To prepare it:
1.  Ensure the directory exists and has appropriate permissions for `containerd` and `stargz-snapshotter` to write to it.
2.  `stargz-snapshotter` will populate this directory in the background as layers are hydrated.

### Directory Backends

This is for purely read-only, externally managed layer sources. The process is:
1.  On a donor machine, pull images into a standard `overlayfs` snapshotter (e.g., in `/tmp/golden-images-prep`).
2.  The `snapshots` subdirectory of that snapshotter's root (e.g., `/tmp/golden-images-prep/snapshots`) can then be used as a `directory` backend on other machines. The `metadata.db` file from the donor snapshotter's root is not needed by the `meta-overlayfs` snapshotter for this backend type.
3.  Mount this directory as read-only on the target nodes.

---

## Appendix: On the Impossibility of Generic Snapshotter Chaining

While this proposal introduces a powerful federation, it is important to understand that it is an **`overlayfs`-specific aggregator**, not a generic mechanism for chaining any two snapshotter types. True generic chaining (e.g., using a `btrfs` snapshot as a layer for an `overlayfs` snapshot) is not feasible with containerd's current architecture for several fundamental reasons:

1.  **Snapshotters as Black Boxes:** The `Snapshotter` interface is a high-level abstraction. Each implementation (`overlayfs`, `btrfs`, `zfs`) is a self-contained "black box" that manages its own on-disk format, metadata, and storage primitives in a way that is opaque to other snapshotters.

2.  **Incompatible Storage Primitives:** The underlying storage technologies are fundamentally different and incompatible at the mount level.
    *   **`overlayfs`** operates on directories. Its `lowerdir` option requires a colon-separated list of directory paths.
    *   **`btrfs`** operates on subvolumes, which are not the same as directories.
    *   **`zfs`** operates on datasets, which are also distinct from simple directories.

3.  **Kernel Mount Limitations:** The Linux `mount` syscall is what assembles the final filesystem. One cannot arbitrarily mix these primitives. For example, it is not possible to pass a `btrfs` subvolume identifier in the `lowerdir` string of an `overlayfs` mount option. The kernel simply does not know how to interpret it.

The `meta-overlayfs` snapshotter proposed here works precisely because it standardizes on the lowest common denominator required by `overlayfs`: a simple directory path. It leverages the `overlayfs`-compatibility contract—that `child.Mounts()` will return a directory path—to aggregate multiple compatible sources into a single `overlayfs` mount.
