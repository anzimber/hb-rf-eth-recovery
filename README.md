# HB-RF-ETH recovery for OpenCCU

Automatic recovery for an already configured RPI-RF-MOD connected through
HB-RF-ETH and used for both BidCos and HmIP. The worker starts at boot and
sleeps in the background when the supported hardware is not detected.

Follow the [manual installation guide](MANUAL_INSTALLATION.md) for prerequisites,
verified downloads, installation, checks, rollback and removal.
For Home Assistant add-ons, start with the guide's
[direct OpenCCU SSH setup](MANUAL_INSTALLATION.md#home-assistant-add-on-enable-direct-openccu-ssh-access).

Recovery restarts RF services and ReGa, temporarily interrupting communication
and potentially running CCU automations. Do not install beside a built-in
recovery service or another recovery controller.

## Contents

- `hb-rf-eth-recovery`: the persistent startup hook and recovery worker.
- `MANUAL_INSTALLATION.md`: installation on existing OpenCCU firmware.
- `checksums.sha256`: SHA-256 of the bundled worker.
- `tests/`: isolated BusyBox ash regression tests; no CCU connection required.

This is a standalone distribution of the worker from
[OpenCCU PR #4519](https://github.com/OpenCCU/OpenCCU/pull/4519), based on
[commit f04ae8555cebdeb90f9dcf40ba996cdfe4dc2e88](https://github.com/anzimber/OpenCCU/commit/f04ae8555cebdeb90f9dcf40ba996cdfe4dc2e88).
The worker is unchanged from that revision. The tests only adapt its local path.
Firmware integration and the native ReGa init-script change stay in the
OpenCCU contribution; they are not needed as manual installation files.
This project does not indicate inclusion in an official OpenCCU release.

## Development

Install Python 3, BusyBox with ash, and util-linux `flock`, then run:

```sh
python3 -m unittest discover -s tests -v
busybox ash -n hb-rf-eth-recovery
sha256sum -c checksums.sha256
```

Tests use temporary fixtures and stubbed services. They do not establish radio
reception or fresh readings from every physical device; verify those on the CCU
as described in the manual.

## License

Apache License 2.0; see [LICENSE](LICENSE). Source provenance is linked above.
