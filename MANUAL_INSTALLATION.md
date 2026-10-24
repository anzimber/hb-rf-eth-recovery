# Manually installing HB-RF-ETH recovery on OpenCCU

This guide installs the recovery worker as a persistent OpenCCU startup hook.
It is intended for existing OpenCCU releases that do not yet include the
built-in recovery service. No firmware image needs to be flashed.

The worker starts automatically at boot. It activates recovery only for a
configured, detected dual-protocol RPI-RF-MOD connected through HB-RF-ETH.
Otherwise, it sleeps in the background and rechecks eligibility every
30 seconds. There is no enable marker to create.

**Version covered:** source commit `0275ab6016c3d1d93b01ef66187f495e8055d393` in
[anzimber/hb-rf-eth-recovery](https://github.com/anzimber/hb-rf-eth-recovery/tree/0275ab6016c3d1d93b01ef66187f495e8055d393).
This is a contribution under review in
[OpenCCU PR #4519](https://github.com/OpenCCU/OpenCCU/pull/4519), not a claim that
it is part of an official OpenCCU release. This installation path was tested on
an OpenCCU OVA installation with RPI-RF-MOD over HB-RF-ETH.

## 1. Requirements and preparation

You need:

- A working OpenCCU installation and root SSH access.
- An already configured HB-RF-ETH gateway, with a nonempty
  `/etc/config/hb_rf_eth` and an initialized `/var/hm_mode`.
- An RPI-RF-MOD used for **both** BidCos and HmIP through `/dev/raw-uart`.
  Other RF modules, mixed USB/Ethernet setups and LAN gateway mode are excluded.
- The `flock`, `timeout`, `readlink`, `monit`, `logger` and `sha256sum` commands.
- A Linux/WSL terminal with `curl`, `sha256sum`, `ssh` and `scp` for the examples
  below. On macOS, `shasum -a 256 FILE` can display the checksum instead.

Download a current CCU backup through the OpenCCU WebUI before proceeding.
Choose a maintenance period: an actual recovery restarts RF services and ReGa,
which interrupts RF communication and may interrupt running CCU automations.
Installing the hook on an otherwise healthy system normally starts monitoring
without restarting those services; an incomplete stack can trigger recovery.

Identify other custom HB-RF-ETH recovery hooks or automatic reboot programs
first. Stop or disable those controllers before activating this one. The
installation below handles an existing hook at the same exact path; it cannot
identify every differently named third-party recovery mechanism.

Replace `192.168.1.50` in the examples with your CCU's address. Use the account
`root`, not the WebUI administrator account.

### Check whether your firmware already provides recovery

Connect from your computer:

```sh
ssh root@192.168.1.50
```

Run on the CCU:

```sh
ls -l /bin/hb-rf-eth-recovery /etc/init.d/S98HBRFEthRecovery
```

If either of these files exists, check your firmware's documentation before
continuing. **Do not install a second hook alongside a built-in recovery
service.** Use the built-in service for firmware that already supplies it.
Missing files are expected on the older releases covered by this manual.

### Check prerequisites on the CCU

Run:

```sh
cat /VERSION
for tool in flock timeout readlink monit logger sha256sum; do
    command -v "$tool" || { echo "STOP: missing $tool"; break; }
done
flock -n /tmp/hb-rf-eth-install-test.lock true
timeout -k 1 2 true
rm -f /tmp/hb-rf-eth-install-test.lock
cat /sys/class/hb-rf-eth/hb-rf-eth/is_connected
```

All six command paths must be printed, and both test commands must succeed.
The last command should print `1` for a currently connected gateway. Stop here
if a required command is missing or the gateway is already disconnected.
Do not substitute `mkdir` for the worker's lock or remove a production lock file.

Inspect only the relevant detected hardware fields:

```sh
grep -E '^HM_MODE=|^HM_HMRF_DEV=|^HM_HMIP_DEV=|^HM_HMRF_DEVNODE=|^HM_HMIP_DEVNODE=|^HM_HMRF_DEVTYPE=|^HM_HMIP_DEVTYPE=' /var/hm_mode
```

Expected values:

| Field | Required value |
| --- | --- |
| `HM_MODE` | `NORMAL` |
| `HM_HMRF_DEV`, `HM_HMIP_DEV` | `RPI-RF-MOD` |
| `HM_HMRF_DEVNODE`, `HM_HMIP_DEVNODE` | `/dev/raw-uart` |
| `HM_HMRF_DEVTYPE`, `HM_HMIP_DEVTYPE` | `HB-RF-ETH@...` |

The worker cannot repair a gateway that was never identified and has no valid
hardware identity in `/var/hm_mode`. Configure and initialize that gateway first.

Check the standard init scripts:

```sh
ls -l /etc/init.d/S47InitRFHardware /etc/init.d/S60multimacd /etc/init.d/S61rfd /etc/init.d/S62HMServer /etc/init.d/S70ReGaHss
if [ -x /etc/init.d/S49InitInterfaces ]; then
    echo "Using S49InitInterfaces"
elif [ -x /etc/init.d/S49hs485d ] && grep -q 'InterfacesList.xml' /etc/init.d/S49hs485d; then
    echo "Using the older S49hs485d interface initializer"
else
    echo "STOP: no supported interface initializer found"
fi
monit summary
```

The RF services should be healthy before installation. Fix missing scripts or
unexplained service failures first. Start installation only after normal boot
has finished, not during another recovery or a firmware update.

## 2. Download and verify the worker on your computer

Run these commands in a local terminal on your computer, **not in the SSH
session**. They download a fixed revision rather than a moving branch.
While this repository is private, these download commands require repository
access. Instead, you can clone the repository using your authenticated GitHub
client, check out the source commit above, and run the checksum command on its
bundled worker. Never copy a GitHub access token onto the CCU:

```sh
curl -fL 'https://raw.githubusercontent.com/anzimber/hb-rf-eth-recovery/0275ab6016c3d1d93b01ef66187f495e8055d393/hb-rf-eth-recovery' -o hb-rf-eth-recovery
printf '%s  %s\n' '6b17d8b90ac3602f36d1202ebbd175f94092a9e4ec0c90d05d6a25d1ae5f8959' 'hb-rf-eth-recovery' | sha256sum -c -
```

Expected result:

```text
hb-rf-eth-recovery: OK
```

For manual comparison, the expected SHA-256 is:

```text
6b17d8b90ac3602f36d1202ebbd175f94092a9e4ec0c90d05d6a25d1ae5f8959
```

Transfer the verified file to temporary storage on the CCU:

```sh
scp -O hb-rf-eth-recovery root@192.168.1.50:/tmp/hb-rf-eth-recovery.new
```

`-O` selects the legacy SCP protocol for CCU SSH servers without SFTP support.
If your SCP client does not support this option, use its supported file-transfer
method while preserving the exact file contents and Unix line endings.

**Install only the worker file.** Do not copy `S98HBRFEthRecovery` into the
persistent hook directory, and do not overwrite the firmware's `S70ReGaHss`.
The worker contains the additional PID guard needed by older init scripts.
The native init-script fix in the PR is intended for future built images.

## 3. Check the old hook and wait for any recovery to finish

Back in the CCU SSH session:

```sh
chmod 600 /tmp/hb-rf-eth-recovery.new
sh -n /tmp/hb-rf-eth-recovery.new
sha256sum /tmp/hb-rf-eth-recovery.new
ls -l /var/run/hb-rf-eth-recovery.monit /var/run/hb-rf-eth-recovery.monit-suspended 2>/dev/null
grep 'hb-rf-eth-recovery:' /var/log/messages | tail -n 20
ps | grep '[h]b-rf-eth-recovery'
```

The syntax check must produce no error and the checksum must match the value
above. Missing journal files are normal. **Do not proceed while either journal
is nonempty or a recovery transaction is still active.** Wait for completion
and inspect Monit. Do not delete a journal to get past this check.

The next section stops an existing worker before replacing its executable.
This matters because the old worker keeps its old functions in memory; copying
a new file alone does not activate new behavior.

## 4. Back up, install atomically and start

Run this entire block on the CCU. It checks the staged file, records any existing
worker PIDs, creates a backup outside `rc.d`, stops the old hook, and then
installs and starts the new file. A subshell keeps its `set -eu` settings out of
your interactive SSH session.

```sh
(
    set -eu
    hook=/etc/config/rc.d/hb-rf-eth-recovery
    stage=/tmp/hb-rf-eth-recovery.new
    backup_root=/etc/config/hb-rf-eth-recovery-backups
    expected=6b17d8b90ac3602f36d1202ebbd175f94092a9e4ec0c90d05d6a25d1ae5f8959

    [ ! -e /bin/hb-rf-eth-recovery ]
    [ ! -e /etc/init.d/S98HBRFEthRecovery ]
    [ -e /var/status/startupFinished ]
    [ -s /etc/config/hb_rf_eth ]
    [ "$(cat /sys/class/hb-rf-eth/hb-rf-eth/is_connected)" = 1 ]
    [ ! -s /var/run/hb-rf-eth-recovery.monit ]
    [ ! -s /var/run/hb-rf-eth-recovery.monit-suspended ]
    [ "$(sha256sum "$stage" | cut -d' ' -f1)" = "$expected" ]
    sh -n "$stage"

    old_pids=''
    hook_real=$(readlink -f "$hook" 2>/dev/null || printf '%s' "$hook")
    for pidfile in /var/run/hb-rf-eth-recovery-service.pid /var/run/hb-rf-eth-recovery.pid; do
        [ -s "$pidfile" ] || continue
        old_pid=$(cat "$pidfile")
        case "$old_pid" in
            ''|*[!0-9]*|0|1) echo "STOP: invalid PID in $pidfile"; exit 1 ;;
        esac
        if kill -0 "$old_pid" 2>/dev/null; then
            command_line=$(tr '\000' ' ' <"/proc/$old_pid/cmdline")
            case "$command_line" in
                "/bin/sh $hook start "|"/bin/sh $hook run "|"/bin/sh $hook restart "|\
                "/bin/sh $hook_real start "|"/bin/sh $hook_real run "|"/bin/sh $hook_real restart ")
                    old_pids="$old_pids $old_pid" ;;
                *) echo "STOP: PID $old_pid belongs to another process"; exit 1 ;;
            esac
        fi
    done

    mkdir -p "$backup_root" /etc/config/rc.d
    backup_dir="$backup_root/$(date +%Y%m%d-%H%M%S)"
    mkdir "$backup_dir"
    if [ -e "$hook" ]; then
        cp -p "$hook" "$backup_dir/hb-rf-eth-recovery"
    else
        touch "$backup_dir/NO_PREVIOUS_HOOK"
    fi
    printf '%s\n' "$backup_dir" >"$backup_root/LAST"
    printf 'Backup directory: %s\n' "$backup_dir"

    if [ -x "$hook" ]; then
        "$hook" stop
    fi
    for old_pid in $old_pids; do
        attempt=0
        while kill -0 "$old_pid" 2>/dev/null; do
            [ "$attempt" -lt 30 ] || {
                echo "STOP: old worker $old_pid is still running"
                exit 1
            }
            sleep 1
            attempt=$((attempt + 1))
        done
    done
    [ ! -s /var/run/hb-rf-eth-recovery.monit ]
    [ ! -s /var/run/hb-rf-eth-recovery.monit-suspended ]

    cp "$stage" "$hook.new"
    chmod 755 "$hook.new"
    mv -f "$hook.new" "$hook"
    if ! "$hook" start; then
        echo "Start failed; use the rollback section and the backup above"
        exit 1
    fi
    [ -s /var/run/hb-rf-eth-recovery-service.pid ]
    new_pid=$(cat /var/run/hb-rf-eth-recovery-service.pid)
    case "$new_pid" in
        ''|*[!0-9]*|0|1) echo "STOP: invalid new worker PID"; exit 1 ;;
    esac
    kill -0 "$new_pid"
    sync
    printf 'Installed; worker PID: %s\n' "$new_pid"
)
```

Proceed only if the block reaches `Installed; worker PID: ...`. If it stops at
an error, keep the printed backup path and use the rollback section. The block
does not force-kill a stuck process or silently delete monitoring state.

If you previously had a recovery process without either expected PID file, or
under a different hook name, stop and investigate it separately before using
this installation procedure.

## 5. Verify startup and persistence

Run on the CCU:

```sh
/etc/config/rc.d/hb-rf-eth-recovery info
cat /var/run/hb-rf-eth-recovery-service.pid
ps | grep '[h]b-rf-eth-recovery'
grep 'hb-rf-eth-recovery:' /var/log/messages | tail -n 20
monit summary
```

Look for one worker command ending in `hb-rf-eth-recovery run`. The canonical
path can appear as `/usr/local/etc/config/rc.d/...`; that is normal because
`/etc/config` points into persistent storage.

With eligible hardware, the log should include:

```text
Background service started; waiting for supported HB-RF-ETH hardware
HB-RF-ETH detected; monitoring active (poll=2s, stability=30s, cooldown=300s)
```

Without eligible hardware, the background worker remains asleep and there is
no active-monitoring message. Recheck the hardware fields in section 1 if
standby is unexpected. `info` describes the service; it is not a live health
check and does not prove that RF devices communicate.

OpenCCU's normal add-on startup hooks launch the executable in
`/etc/config/rc.d` at boot. No cron job, `rc.local` addition or enable file is
required. The file is stored on persistent user storage. During a planned
reboot, verify the worker and logs again after startup; a reboot is not required
just to activate this installation.

### Check actual device data

Confirm more than process presence or `CONNECTED=1`:

1. Check that the relevant Monit services are `OK` and monitored.
2. Observe a known sensor producing a genuinely new reading.
3. Confirm the new value and its timestamp in the OpenCCU WebUI.
4. If Home Assistant is used, confirm it receives the same new value.
5. If appropriate, test a safe actuator selected by the owner.

The worker renews ReGa after successful RF reinitialization because an intact
interface file alone does not prove the WebUI is receiving current events.
Battery devices can report `CONFIG_PENDING` or lack values while configuration
or the next device transmission is outstanding. Their reporting intervals
vary. Do not declare every device healthy from the adapter status, repeatedly
restart the RF stack, clear flags by hand, or reset/re-pair devices as a first
response to that display. Investigate fresh device-level data separately.

## 6. Optional controlled link test

Perform this only during a maintenance period when interruption is acceptable.
Do not combine it with a gateway firmware update or another recovery test.

1. Keep a CCU SSH session open and watch the log:

   ```sh
   tail -f /var/log/messages
   ```

2. Briefly disconnect the HB-RF-ETH gateway's network cable, then restore it.
   Leave the CCU's own network connection intact so you can continue observing.
3. Expect a connection-lost message, then a continuous-stability wait after
   reconnection. Recovery starts after 30 seconds of stable connectivity,
   subject to the five-minute recovery cooldown.
4. Allow the RF stack and ReGa to complete startup. Monit checks are temporarily
   paused during controlled recovery, then restored. The wait can take several
   minutes; init commands have bounded timeouts, not a guarantee of instant
   device recovery.
5. Look for `Recovery completed; RF stack and previous Monit checks verified`,
   then repeat the actual-device checks above.

Use Ctrl+C to stop the log viewer; this does not stop the background worker.
If a second interruption occurs during cooldown, recovery remains pending and
is retried later. Avoid repeated test interruptions while diagnosing a failure.
The worker never requests an automatic full-system reboot.

## 7. Stop, disable or roll back

### Temporarily stop

```sh
/etc/config/rc.d/hb-rf-eth-recovery stop
```

The hook remains executable, so OpenCCU starts it again at the next boot.
If stop interrupts an active transaction, it waits for the current bounded
command and monitoring cleanup. Let it finish; do not immediately use `kill -9`.

### Disable until manually re-enabled

Wait for recovery to finish, then:

```sh
/etc/config/rc.d/hb-rf-eth-recovery stop
chmod 644 /etc/config/rc.d/hb-rf-eth-recovery
```

This prevents automatic startup through the executable add-on hook. To re-enable:

```sh
chmod 755 /etc/config/rc.d/hb-rf-eth-recovery
/etc/config/rc.d/hb-rf-eth-recovery start
```

### Restore the most recent pre-installation hook

Run this whole block on the CCU:

```sh
(
    set -eu
    hook=/etc/config/rc.d/hb-rf-eth-recovery
    backup_dir=$(cat /etc/config/hb-rf-eth-recovery-backups/LAST)
    [ -d "$backup_dir" ]
    if [ ! -f "$backup_dir/hb-rf-eth-recovery" ]; then
        echo "There was no previous hook to restore; use the removal steps"
        exit 1
    fi
    if [ -f "$hook" ]; then
        /bin/sh "$hook" stop
    fi
    [ ! -s /var/run/hb-rf-eth-recovery.monit ]
    cp -p "$backup_dir/hb-rf-eth-recovery" "$hook.new"
    mv -f "$hook.new" "$hook"
    if [ -x "$hook" ]; then
        "$hook" start
    fi
    sync
    echo "Previous hook restored; check its logs and Monit"
)
```

If the old hook used different operational behavior, its own limitations return
with it. Keep the CCU backup as well as this script backup.

### Remove the manual installation

To stop and remove the hook from automatic startup while retaining its file:

```sh
/bin/sh /etc/config/rc.d/hb-rf-eth-recovery stop
mkdir -p /etc/config/hb-rf-eth-recovery-backups
mv /etc/config/rc.d/hb-rf-eth-recovery /etc/config/hb-rf-eth-recovery-backups/removed-$(date +%Y%m%d-%H%M%S)
```

Inspect `monit summary` afterward. Do not remove the gateway configuration,
RF device database, or the production lock/journal files as part of uninstalling.

## 8. Updating the script or moving to built-in recovery

For a newer manual version, obtain its explicit commit and checksum, repeat
staging and backup, and stop the running old worker before replacement.
Do not reuse this guide's checksum for another revision.

Before upgrading to a firmware image that includes built-in recovery, remove
the manual hook using section 7 and retain its backup outside `rc.d`. Let the
firmware's `/bin/hb-rf-eth-recovery` and `S98HBRFEthRecovery` own startup. Leaving
an executable manual hook beside the built-in worker can prevent activation;
the two implementations share runtime ownership and must not be installed
in parallel. Follow the new firmware's migration instructions where supplied.

## Troubleshooting reference

| Symptom | Check |
| --- | --- |
| No worker PID file after start | Syntax, required commands, logs, and another worker already holding the lock |
| Worker runs but does not activate | Gateway configuration, detected hardware, safe mode and another legacy hook |
| Monit temporarily says `Not monitored` | Check whether a recovery is active; verify restoration after completion |
| Monit journal remains nonempty | Monitoring restoration failed; preserve the journal and diagnose Monit |
| WebUI values remain stale | Compare fresh direct device values with ReGa/WebUI and application timestamps |
| A device remains `CONFIG_PENDING` | Inspect that device's fresh status and transmission/configuration progress |
| Firmware already includes the service | Use built-in recovery; remove the manual hook before migration |

For a report, retain the script checksum, OpenCCU and gateway versions, relevant
log timestamps, Monit states, and a direct/WebUI comparison for a known device.
Redact passwords, tokens and device identifiers before sharing diagnostics.
