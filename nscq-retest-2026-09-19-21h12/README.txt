NVSwitch attestation on an 8x H100 node: curl reaches NVIDIA's OCSP responder
from the same machine where nvattest cannot.
19 September 2026, 21:11 UTC, workload wrk-5l3ol8ubj71g, image ubuntu-24-04-lts-595.
Node rented and released for this run: 9.2 minutes.

WHY THIS RUN EXISTS

Earlier today, with the driver-matched NSCQ library from NVIDIA's repository,
switch attestation got past the NSCQ session and failed at certificate chain
validation with "SSL connect error (35)" against ocsp.ndis.nvidia.com.

NVIDIA then confirmed on nvtrust issue #153 that the RIM and OCSP endpoints are
meant to be reachable from the tenant guest, and that nvattest accepts
--rim-url, --ocsp-url and --nras-url.

A cheap run on a single-GPU confidential VM showed all three endpoints fully
reachable, but that was a different SKU. This run does the network test and the
attestation ON THE SAME EIGHT-GPU NODE, in the same session, so the comparison
is apples to apples.

RESULT: THE NETWORK IS FINE ON THIS NODE. THE TOOL IS NOT.

  1. Same node, same image, same minute, system curl:

       ocsp.ndis.nvidia.com        166.117.200.57, 166.117.113.121
         HTTP 405 in 0.115 s
       rim.attestation.nvidia.com  166.117.155.59, 166.117.56.89
         HTTP 403 in 0.114 s
       nras.attestation.nvidia.com 34.120.45.54
         HTTP 200 in 0.189 s

     405 and 403 are application answers to a bare GET. They prove DNS
     resolution, TCP, the TLS handshake and an HTTP exchange all completed.

  2. nvattest on that same machine, seconds later, with the endpoints passed
     explicitly:

       nvattest attest --device nvswitch \
         --ocsp-url https://ocsp.ndis.nvidia.com \
         --rim-url https://rim.attestation.nvidia.com \
         --nras-url https://nras.attestation.nvidia.com

       [nv_http.cpp:142] Fatal libcurl error code: SSL connect error (35)
       [nv_ocsp.cpp:163] Failed to perform OCSP check with url:
                         https://ocsp.ndis.nvidia.com
       [rim.cpp:265] Failed to generate certificate chain claims for RIM with
                     id NV_SWITCH_BIOS_5612_0002_890_96106D0001
       Error 002: Internal Error

  3. Passing the URLs explicitly changes nothing. The run without options
     fails identically.

WHAT THIS ESTABLISHES

  The failure is inside nvattest, not in the platform, not in egress policy,
  and not in DNS. A TLS connect error from the tool, on a host that system
  curl reaches in 115 milliseconds with OpenSSL 3.0.13 and the distribution CA
  bundle, points at the tool's own TLS stack or trust store.

  It also confirms the tool now gets FURTHER than before: it has switch
  evidence and is validating the RIM for a real switch BIOS identifier,
  NV_SWITCH_BIOS_5612_0002_890_96106D0001. Only the revocation check blocks it.

  We still claim nothing new about the fabric. The NVLink fabric on our 8-GPU
  nodes remains unverified on our public pages.

OPEN QUESTION PUT TO NVIDIA

  Does nvattest use the system trust store, or does it carry its own CA bundle
  or TLS stack? And is there anything in its network path that would ignore the
  environment's proxy or resolver settings, or pin a CA?

FILES

  run.log      raw terminal output of the whole run, unedited
  SHA256SUMS   checksums of the files in this folder

Licence CC BY 4.0. Sequence: nvswitch-2026-09-16 (original failure, wrong
cause), nscq-retest-2026-09-19 (uninstallable branch package),
nscq-retest-2026-09-19-10h11 (NSCQ works, OCSP fails),
egress-nvidia-2026-09-19-18h44 (endpoints reachable from a single-GPU VM),
this folder (same node, curl works, nvattest does not).
Issue: https://github.com/NVIDIA/nvtrust/issues/153
