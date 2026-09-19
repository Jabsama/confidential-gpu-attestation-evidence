NSCQ retest on an 8x H100 node, with the driver-matched library from NVIDIA's
own repository, following yf23's answer on nvtrust issue #153.
19 September 2026, 10:11 UTC, workload wrk-fgj2k14vkc18, image ubuntu-24-04-lts-595.
Node rented and released for this run: 8.6 minutes.

WHAT CHANGED SINCE OUR PREVIOUS RUNS

On 16 September we reported that NVSwitch attestation fails from inside a TDX
guest with NSCQ_RC_WARNING: RDT init failure (Code 1). We had installed
libnvidia-nscq 615.71.09 by hand, because the image ships no NSCQ package.

On 19 September, earlier, we reported that the branch-suffixed package
libnvidia-nscq-595 cannot be installed on the stock image: it resolves to
595.58.03 from Ubuntu's archive and conflicts with nvidia-kernel-common.

NVIDIA then pointed out that the correct package is libnvidia-nscq WITHOUT the
branch suffix, from NVIDIA's network repository, pinned to the driver version.
That is what this run does.

RESULT: THE NSCQ BLOCKER IS GONE. A NEW AND DIFFERENT ONE APPEARS.

  1. The NVIDIA repository does carry a library matching the driver exactly:
       libnvidia-nscq/unknown 595.71.05-1ubuntu1 amd64
     alongside 595.91.07, 595.58.03, 610.x, 615.x and others.

  2. It installs cleanly on the stock image, no conflict:
       ii  libnvidia-nscq  595.71.05-1ubuntu1  amd64
       /usr/lib/x86_64-linux-gnu/libnvidia-nscq.so.2.0 -> libnvidia-nscq.so.595.71.05

  3. nvattest attest --device nvswitch NO LONGER fails at NSCQ session
     creation. There is no RDT init failure in this run. The tool reaches
     "Attesting Switches" and proceeds to build attestation report claims,
     which means switch evidence is collected from inside the TDX guest.

  4. It now fails much later, at certificate chain validation, because the
     online revocation check cannot reach NVIDIA's OCSP responder:

       [nv_http.cpp:142] Fatal libcurl error code: SSL connect error (35)
       [nv_ocsp.cpp:163] Failed to perform OCSP check with url:
                         https://ocsp.ndis.nvidia.com
       [switch/evidence.cpp:336] Failed to validate attestation report
                                 certificate chain
       Error 002: Internal Error

WHAT THIS DOES AND DOES NOT ESTABLISH

  It does NOT establish that the NVLink fabric is attested. It is not. The run
  ends in failure and we claim nothing new about the fabric.

  It DOES establish that our earlier explanation was wrong in its cause. We
  said NSCQ could not open a session from inside a TDX guest. With the
  version-matched library from NVIDIA's repository, it can. The remaining
  blocker is reaching an external OCSP responder over TLS from the guest, which
  looks like a network or trust-store question rather than an architectural
  one.

  The next precise test, which we have not run yet, is whether
  https://ocsp.ndis.nvidia.com is reachable at all from inside this image, and
  whether nvattest can be pointed at a cached or offline revocation path.

OUR PUBLIC CLAIMS AFTER THIS RUN

  Unchanged: the eight GPUs of an 8x H100 node attest individually in Protected
  PCIe mode (10 September 2026, NRAS, ppcie_mode False). The NVLink fabric is
  still NOT verified and we still do not claim it.

  Corrected: we no longer say the fabric is unattestable from inside a guest
  because NSCQ refuses to initialise. That statement was based on a mismatched
  library and it is retired.

FILES

  run.log      raw terminal output of the whole run, unedited
  SHA256SUMS   checksums of the files in this folder

Licence CC BY 4.0. Related: nvswitch-2026-09-16 (the original failure),
nscq-retest-2026-09-19 (the uninstallable branch package),
ppcie-8x-h100-2026-09-10 (the successful per-GPU attestation on the same node
type). Issue: https://github.com/NVIDIA/nvtrust/issues/153
