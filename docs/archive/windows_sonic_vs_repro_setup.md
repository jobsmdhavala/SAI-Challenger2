# Windows and Ubuntu Quick Setup: SONiC VS + SAI-Challenger

## Goal

Use this runbook to get a reproducible lab quickly on Windows or Ubuntu.

- Windows recommended path: Path A (QEMU SONiC VM + client container)
- Ubuntu recommended path: Path B first, then Path A

Related notes:
- docs/summary.txt
- docs/summary_commands.txt
- docs/summary_usecases.txt

## Windows Quick Start (Path A)

### Terminal 1: start SONiC VS VM and keep it running

```powershell
& "C:\Program Files\qemu\qemu-system-x86_64.exe" `
  -m 6144 -smp 4 `
  -drive file="C:\Users\mdhavala\Downloads\sonic-vs.qcow2",if=virtio,format=qcow2 `
  -netdev user,id=mgmt,hostfwd=tcp:127.0.0.1:2222-:22,hostfwd=tcp:127.0.0.1:6379-:6379 `
  -device e1000,netdev=mgmt `
  -nographic
```

Wait for SONiC boot to complete (about 2 to 3 minutes).

### Terminal 2: start client and SSH forwarder

```powershell
cd C:\Users\mdhavala\Downloads\SAI-Challenger2
$repo = (Get-Location).Path

docker run -d --name sc-client-run `
  --cap-add=NET_ADMIN `
  --add-host host.docker.internal:host-gateway `
  -v "${repo}:/sai-challenger" `
  plvisiondevs/sc-client:bookworm-latest

docker run -d --name sonic-ssh-fwd `
  -p 127.0.0.1:22:22 `
  alpine/socat `
  tcp-listen:22,reuseaddr,fork `
  tcp:host.docker.internal:2222
```

### Connectivity check

```powershell
docker exec -i sc-client-run bash -lc "apt-get update >/dev/null 2>&1; apt-get install -y netcat-openbsd >/dev/null 2>&1 || true; nc -vz host.docker.internal 22; nc -vz host.docker.internal 6379"
```

### First test

```powershell
docker exec -i sc-client-run bash -lc "cd /sai-challenger && pytest -v tests/api/test_vlan.py -k test_vlan_create --testbed=sainpu_sonic_vs"
```

## Ubuntu Quick Start

Ubuntu is usually stable for native veth-based paths.

### Path B1: SAIVS standalone API sanity

```bash
cd ~/SAI-Challenger2

./build.sh -a trident2 -t saivs
./run.sh -a trident2 -t saivs
./exec.sh -t saivs pytest --testbed=saivs_standalone -v -k "test_l2_basic"
```

### Path B2: legacy SAI-PTF umbrella

```bash
cd ~/SAI-Challenger2/tests
pytest --testbed=saivs_thrift_standalone ../usecases/sai-ptf/SAI/ptf/saifdb.py -k FdbAttributeTest -v
```

### Optional Path A on Ubuntu (SONiC VM)

Use the same model as Windows Path A and run tests with testbed sainpu_sonic_vs.

## Cleanup

### Windows

```powershell
docker rm -f sonic-ssh-fwd sc-client-run 2>$null
```

Stop QEMU using Ctrl+C in Terminal 1.

### Ubuntu

```bash
docker ps -q | xargs -r docker stop
```

## Troubleshooting

- If Redis or SSH is unreachable, verify SONiC VM is fully booted.
- If dataplane hangs on Windows, switch to Path A and avoid local SAIVS dataplane path.
- If path B tests fail on Ubuntu, verify submodules and testbed names first.
