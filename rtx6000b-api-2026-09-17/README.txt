VoltageGPU evidence bundle: single-GPU RTX PRO 6000 Blackwell, Intel TDX guest, PUBLIC API PATH ONLY
Captured 2026-09-17 12:28 UTC. Same SKU as ../rtx6000b-2026-09-17/, different question:
can a customer, or a customer's coding agent, get both proofs with nothing but an API key
and the public documentation, never opening the dashboard? Yes, and one thing was wrong on
the way, fixed the same day (see "What we found").

The exact sequence (base https://api.voltagegpu.com, header X-API-Key)
  1. GET  /api/confidential/vm/tiers            rtx6000b-small, 3.80 $/h, 10 available,
                                                attestation.verified true, verifiedOn 2026-09-17
  2. POST /api/volt/ssh-keys                    the tenant's ed25519 public key, HTTP 201
  3. POST /api/confidential/vm/deploy           {"name":"twoproofs-...","resource_name":"rtx6000b-small","hardstop_hours":1}
                                                HTTP 200, podId cmu5i01bz001vl9042ag1ps0w, workload wrk-xhnrun04sbj8,
                                                one hour (3.80 $) charged upfront
  4. GET  /api/volt/pods                        polled every 15 s until the VM had an address
  5. ssh -p 20000 ubuntu@157.254.50.65           the commands below, 19 seconds wall clock
  6. scp the bundle out, verify it here          RESULT: VERIFIED (verify-outside-the-vm.txt)
  7. POST /api/volt/pods/{podId}/stop           HTTP 200, prepaidRefund 3.20 of 3.80
                                                (0.84 h unused), total cost of the run 0.60 $

Inside the VM (Ubuntu 24.04, kernel 6.8.0-110-generic, /dev/tdx_guest present,
systemd-detect-virt kvm, driver 595.71.05, nvidia-smi conf-compute CC State ON, Ready)
  curl -sS https://bootstrap.pypa.io/get-pip.py | python3 - --user --break-system-packages
  python3 -m pip install --user --break-system-packages "voltage-verify[attest]"
  export PATH=$HOME/.local/bin:$PATH
  voltage-verify manifest --challenge auto -o manifest.json
      challenge ac7258d5020853718cb97ee58b43e81736e766c8032c0592c903b9c732437389
  sudo env PATH=$PATH PYTHONPATH=$(python3 -c 'import site;print(site.getusersitepackages())') \
    voltage-verify attest --manifest manifest.json --mode single-gpu -o bundle.json
      NRAS: Attestation Successful, 4940-byte TDX quote, bundle 121 960 bytes
  voltage-verify verify bundle.json --challenge ac7258...7389      RESULT: VERIFIED

Outside the VM, on a Windows laptop (voltage-verify 0.1.1 from PyPI)
  voltage-verify verify bundle.json --challenge ac7258...7389 --hwmodel GB20X --gpus 1
      RESULT: VERIFIED                      (full output in verify-outside-the-vm.txt)
  voltage-verify verify bundle.json --challenge 0000...0000
      RESULT: NOT VERIFIED (manifest.challenge)   a replayed bundle is rejected
  voltage-verify verify bundle.json --offline --challenge ac7258...7389
      RESULT: VERIFIED                      Intel collateral embedded in the bundle, no network

What we found
  GET /api/volt/pods used to return, for a VM, the container gateway command
  "ssh wrk-...@ssh.voltagegpu.com". A VM is not behind that gateway: the command answered
  "Permission denied (publickey)" and an API-key user had no other way to learn the address.
  Fixed on 2026-09-17: the listing now returns provider_status, ssh_ready, ssh_host, ssh_port,
  ssh_user (ubuntu) and a working ssh_command for VMs. The commands returned in
  attestation.verifyInsideTheVm also started with "pip install", which fails because the image
  ships without pip; they now start with get-pip.

Files
  manifest.json              the tenant's manifest with the random challenge
  bundle.json                TDX quote + NVIDIA NRAS tokens bound to that challenge
  SHA256SUMS                 computed inside the VM before download, re-checked after
  verify-outside-the-vm.txt  the verification run on the laptop, not on the VM

Checksums (sha256, computed in the VM)
  see SHA256SUMS
