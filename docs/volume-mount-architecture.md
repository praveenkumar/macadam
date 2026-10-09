# Volume Mount Architecture in Podman-Macadam

This document describes how VM volume mounts work in the Macadam + Podman-Macadam stack.

## High-level flow

```text
User input: --volume /host/path:/vm/path:ro
            ↓
Macadam CLI (crc-org/macadam)
            ↓
Podman-Macadam shim.Init()
            ↓
Volume strings are parsed into MachineConfig.Mount entries
            ↓
MachineConfig is persisted to disk
            ↓
VM starts
            ↓
Provider-specific mount logic executes
            ↓
Host path becomes accessible inside the VM
```

---

## 1. CLI input

The Macadam CLI collects `--volume` values and passes them into the Podman init options.

Relevant code in the Macadam repo:

```go
VolumeFlagName := "volume"
flags.StringArrayVarP(&initOptsFromFlags.Volumes, VolumeFlagName, "v", []string{}, "Volumes to mount, source:target")

initOpts.Volumes = initOptsFromFlags.Volumes
return shim.Init(*initOpts, vmProvider)
```

This is the user-facing entry point. Example:

```bash
macadam init --volume /home/user/data:/mnt/shared:ro fedora-cloud.raw
```

---

## 2. Conversion to internal mount objects

The real parsing happens in the Podman-Macadam repository.

Relevant file:
- `pkg/machine/shim/host.go`

```go
if mp.VMType() != machineDefine.WSLVirt {
    mc.Mounts = CmdLineVolumesToMounts(opts.Volumes, mp.MountType())
}
```

The helper function is here:
- `pkg/machine/shim/volume.go`

```go
func CmdLineVolumesToMounts(volumes []string, volumeType vmconfigs.VolumeMountType) []*vmconfigs.Mount {
    mounts := []*vmconfigs.Mount{}
    for i, volume := range volumes {
        if volume == "" {
            continue
        }
        var mount vmconfigs.Mount
        tag, source, target, readOnly, _ := vmconfigs.SplitVolume(i, volume)
        switch volumeType {
        case vmconfigs.VirtIOFS:
            virtioMount := machine.NewVirtIoFsMount(source, target, readOnly)
            mount = virtioMount.ToMount()
        default:
            mount = vmconfigs.Mount{
                Type:          volumeType.String(),
                Tag:           tag,
                Source:        source,
                Target:        target,
                ReadOnly:      readOnly,
                OriginalInput: volume,
            }
        }
        mounts = append(mounts, &mount)
    }
    return mounts
}
```

The parser is here:
- `pkg/machine/vmconfigs/volumes.go`

```go
func SplitVolume(idx int, volume string) (string, string, string, bool, string) {
    tag := fmt.Sprintf("vol%d", idx)
    paths := pathsFromVolume(volume)
    source := extractSourcePath(paths)
    target := extractTargetPath(paths)
    readonly, securityModel := extractMountOptions(paths)
    return tag, source, target, readonly, securityModel
}
```

This turns strings like:

- `/host/path:/vm/path`
- `/host/path:/vm/path:ro`
- `/host/path:/vm/path:rw`

into structured metadata like:

```go
Mount{
    Source: "/host/path",
    Target: "/vm/path",
    ReadOnly: true,
    Type: "virtiofs",
    Tag: "vol0",
}
```

---

## 3. Mount metadata is stored in the machine config

Relevant file:
- `pkg/machine/vmconfigs/config.go`

```go
type MachineConfig struct {
    // ...
    Mounts []*Mount
    // ...
}
```

This means the mount configuration is persisted on disk as part of the VM config. It survives across VM lifecycle operations and is reloaded when the machine is started again.

---

## 4. Mounting happens after the VM is ready

The actual mount execution is triggered in the start flow.

Relevant code:
- `pkg/machine/shim/host.go`

```go
if err := mp.MountVolumesToVM(mc, opts.Quiet); err != nil {
    return err
}
```

This runs after the VM boots and SSH connectivity is established, because the mount commands are executed through SSH into the guest.

---

## 5. Provider-specific mount implementations

The provider decides how the host path is exposed to the guest.

### 5.1 QEMU / Linux uses virtiofs

Relevant file:
- `pkg/machine/qemu/stubber.go`

```go
func (q *QEMUStubber) MountType() vmconfigs.VolumeMountType {
    return vmconfigs.VirtIOFS
}
```

And the actual guest mount command is roughly:

```go
mountOptions := []string{"-t", "virtiofs"}
mountOptions = append(mountOptions, []string{mount.Tag, strconv.Quote(mount.Target)}...)
mountFlags := fmt.Sprintf("context=\"%s\"", machine.NFSSELinuxContext)
if mount.ReadOnly {
    mountFlags += ",ro"
}
mountOptions = append(mountOptions, "-o", mountFlags)
err = machine.LocalhostSSH(..., append([]string{"sudo", "mount"}, mountOptions...))
```

This effectively does:

```bash
sudo mount -t virtiofs <tag> /mnt/shared -o context="system_u:object_r:nfs_t:s0",ro
```

Before QEMU starts, the host also spawns a `virtiofsd` helper for each mount so the host directory can be shared into the VM.

---

### 5.2 AppleHV uses virtiofs + Ignition

Relevant file:
- `pkg/machine/applehv/stubber.go`

```go
if ignBuilder != nil {
    virtiofsMounts := make([]machine.VirtIoFs, 0, len(mc.Mounts))
    for _, mnt := range mc.Mounts {
        virtiofsMounts = append(virtiofsMounts, machine.MountToVirtIOFs(mnt))
    }

    virtIOIgnitionMounts, err := apple.GenerateSystemDFilesForVirtiofsMounts(virtiofsMounts)
    if err != nil {
        return err
    }
    ignBuilder.WithUnit(virtIOIgnitionMounts...)
}
```

This is different from QEMU: instead of mounting after boot with SSH, AppleHV creates the mount configuration through Ignition/systemd so the guest automatically mounts the shared path during startup.

The tag generation logic is here:
- `pkg/machine/volumes.go`

```go
func (v VirtIoFs) generateTag() string {
    sum := sha256.Sum256([]byte(v.Target))
    stringSum := hex.EncodeToString(sum[:])
    return stringSum[:36]
}
```

For AppleHV, the tag must stay short enough for the virtiofs mechanism.

---

### 5.3 Hyper-V uses 9p over vsock

Relevant files:
- `pkg/machine/hyperv/volumes.go`
- `pkg/machine/hyperv/vsock/vsock.go`

Hyper-V does not use virtiofs. It uses 9p shares over a Windows vsock socket. The actual guest-side mount logic is based on a Unix socket under `/run/9p-<port>.sock` and then runs:

```bash
mount -t 9p -o trans=unix,version=9p2000.L /run/9p-<port>.sock /mnt/shared
```

This is a different transport from QEMU/AppleHV, but the overall model is the same: a host-side share is exposed and the guest mounts it at the requested target path.

---

## 6. Security and validation

Before mounting, the code rejects problematic targets that would replace important system directories.

Relevant code:
- `pkg/machine/shim/host.go`

```go
func validateDestinationPaths(dest string) error {
    illegalMounts := map[string]struct{}{
        "/bin": {}, "/boot": {}, "/dev": {}, "/etc": {},
        "/home": {}, "/proc": {}, "/root": {}, "/run": {},
        "/sbin": {}, "/sys": {}, "/tmp": {}, "/usr": {},
        "/var": {},
    }
    mountTarget := path.Clean(dest)
    if _, ok := illegalMounts[mountTarget]; ok {
        return fmt.Errorf("machine mount destination cannot be %q", mountTarget)
    }
    return nil
}
```

This prevents users from mounting over critical directories such as `/etc`, `/bin`, `/usr`, or `/var`.

---

## 7. End-to-end example

Example input:

```bash
macadam init --volume /home/user/project:/workspace:ro myvm.raw
```

Flow:

1. Macadam CLI captures the string.
2. Podman-Macadam parses it into a mount record.
3. Mount record includes:
   - source: `/home/user/project`
   - target: `/workspace`
   - readOnly: `true`
   - type: provider-specific (`virtiofs` or `9p`)
4. VM boots.
5. Provider-specific code exposes the host directory.
6. Guest mounts it at `/workspace`.

Result: the host path is visible inside the VM at the requested mount point.

---

## 8. Summary

The volume system is structured as follows:

- CLI accepts `--volume`
- Podman-Macadam parses the value into a `Mount`
- `MachineConfig.Mounts` stores the mount metadata
- `shim.Start()` triggers `MountVolumesToVM()` after the VM is up
- provider implementations mount the host path using:
  - `virtiofs` for QEMU and AppleHV
  - `9p` over vsock for Hyper-V

This design is cross-platform and keeps the machine config persistent while letting each VM provider implement the actual guest mount mechanics in the most native way supported by that backend.

---

## Related files

- `crc-org/macadam`: `cmd/macadam/init.go`
- `crc-org/podman-macadam`: `pkg/machine/shim/host.go`
- `crc-org/podman-macadam`: `pkg/machine/shim/volume.go`
- `crc-org/podman-macadam`: `pkg/machine/vmconfigs/config.go`
- `crc-org/podman-macadam`: `pkg/machine/qemu/stubber.go`
- `crc-org/podman-macadam`: `pkg/machine/applehv/stubber.go`
- `crc-org/podman-macadam`: `pkg/machine/hyperv/volumes.go`

This is the architecture that explains how the mount directory works in the VM.
