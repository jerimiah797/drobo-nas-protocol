# Drobo NAS protocol (`DRINASD`) — notes and a read-only status tool

Drobo NAS units hide their physical disks. The Linux system inside sees a single
large virtual LUN and no `/dev/sdb..sde`, so there is no SMART, no per-drive
capacity, no serial numbers — nothing. That data was only ever visible in Drobo
Dashboard, a Windows application for a company that no longer exists.

It turns out you do not need Dashboard, or the firmware, or any reverse
engineering of the storage layer. **The management daemon streams it in plaintext
XML to anyone who opens a TCP socket.**

## Scope, honestly

Everything here was established against **one Drobo 5N2 running firmware 4.3.1
(`13.48.117497`)**. Other NAS models (5N, B810n, …) very likely speak the same
protocol, but that is an inference and has not been tested. Nothing here has been
verified against a second unit.

This tool **only reads**. It opens a connection, reads one frame, and closes.
It never sends a command.

## The protocol

`nasd` listens on TCP **5000** and **5001**. On connect to 5000 it immediately
pushes a status document. There is **no authentication and no TLS**.

```
+----------------+-------------+------------------+------------------+------+
| "DRINASD\0"    | version     | length           | UTF-8 XML        | NUL  |
| 8 bytes        | 4 bytes     | 4 bytes, BIG-EN  | <length-1> bytes | 0x00 |
+----------------+-------------+------------------+------------------+------+
```

Observed version bytes: `01 01 00 00`. Root element: `<ESATMUpdate>`.

### Two details that will cost you an hour

1. **The length is big-endian.** On a little-endian host, reading it as LE turns a
   7 KB frame into ~2.7 GB, and the socket read hangs until your timeout rather
   than failing cleanly.
2. **The payload is NUL-terminated and the length includes the NUL.** Feed it
   straight to an XML parser and you get `not well-formed (invalid token)`
   pointing at a line *past* the closing root tag. `rstrip(b"\x00")` first.

### Per-slot fields

Slots appear as `<n0>`…`<n5>` (a 5-bay unit reports 6 — the last is the mSATA
accelerator).

| field | meaning |
|---|---|
| `mSlotNumber` | 0-based slot |
| `mMake` | drive model string, e.g. `SEAGATE  ST8000VN004-3CP1` |
| `mSerial` | drive serial |
| `mDiskFwRev` | drive firmware revision |
| `mPhysicalCapacity` | bytes |
| `mDiskType` | 0 = HDD, 4 = mSATA accelerator |
| `mDiskState` | 16 = HDD in service, 32 = accelerator |
| `mStatus` | 3 = normal |
| `mErrorCount` | error count |
| `SSDLifeRemaining` | percent; only meaningful for the SSD slot |
| `mTemperature` | **present but always 0 on this unit — do not trust it** |

Array-level fields include `mTotalCapacityProtected`, `mUsedCapacityProtected`,
`mFreeCapacityProtected`, `mDiskPackID`, `mModel`, `mDiskPackStatus`. The capacity
figures match `esa vxphyscapacity` / `esa vxusedcapacity` byte for byte, which is
a useful independent cross-check.

## How this was found

No firmware analysis. `strings /usr/sbin/nasd` — the daemon is stripped ARM ELF,
but its **command enum is in plaintext**: 105 entries including
`eCmdGetAllDevicesStatusXML`, `eCmdGetSysInfo`, `eCmdGetFanInfo`,
`eCmdGetPowerInfo`, `eCmdGetFirmwareInfo`, `eCmdGetDiskPackId`.

Then simply connecting to port 5000 was enough — the daemon volunteers the
status document without being asked.

## Two warnings

**Security.** Anything on your LAN can read drive serials, models, capacities and
array identifiers from port 5000 without credentials. That is how Dashboard was
designed to work and it is unremarkable on a trusted network — but the port
should never be exposed beyond it. The vendor is defunct, so there is no patch
and nobody to report it to.

**The same enum contains destructive verbs.** `eCmdInsertDrive` and
`eCmdRemoveDrive` sit alongside the harmless getters. Anything built on this
should allowlist the `Get*` verbs explicitly rather than exposing a generic
command channel.

## Relationship to `drobo-utils`

[`drobo-utils`](https://github.com/petersilva/drobo-utils) is the established
open-source Drobo tool, and it is **not** what this is. It drives *direct-attached*
Drobos through SCSI passthrough (`SG_IO`, INQUIRY `0x12` and MODE SENSE `0x5A`) to
a `/dev/sdX` node. NAS models were never in its supported list and have no such
node on the client.

This is a different transport for a different product line, which is why it is a
separate tool rather than a patch. `drobo-utils` is also unmaintained — its author
notes he no longer has hardware to test with.

## Usage

```
./drobo-nas-status 192.168.1.174          # human-readable table
./drobo-nas-status 192.168.1.174 --json   # machine-readable
./drobo-nas-status 192.168.1.174 --raw    # the XML document as received
```

Python 3, standard library only.
