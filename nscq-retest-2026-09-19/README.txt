NSCQ retest on an 8x H100 node, after NVIDIA's reply on nvtrust issue #153
19 September 2026, workload wrk-83vbfp7ysyc8, image ubuntu-24-04-lts-595
Node rented and released for this run: 8.1 minutes.

WHY: on 16 September we reported that NVSwitch attestation fails from inside a
TDX guest (NSCQ_RC_WARNING: RDT init failure, Code 1) with libnvidia-nscq
615.71.09 installed by hand. NVIDIA's steven-bellock asked, reasonably, why we
had installed a different version from the driver and suggested trying 595.71.05.

RESULT: INCONCLUSIVE, and for a new reason worth knowing.

  1. The image ships driver 595.71.05 and NO NSCQ package at all.

  2. The branch-matched package exists but resolves to a different point
     release, from Ubuntu's archive rather than NVIDIA's:
       libnvidia-nscq-595
         Installed: (none)
         Candidate: 595.58.03-0ubuntu0.24.04.1
           500 http://us.archive.ubuntu.com/ubuntu noble-updates/multiverse

  3. It cannot be installed on the stock image:
       The following packages have unmet dependencies:
        nvidia-kernel-common : Conflicts: nvidia-kernel-common
        nvidia-kernel-common-595-server : Conflicts: nvidia-kernel-common
       E: Error, pkgProblemResolver::Resolve generated breaks

  4. With no NSCQ present, nvattest fails at the same place, now for the
     obvious reason:
       [switch/evidence.cpp:178 get_evidence] [error] Failed to initialize NSCQ
       Error 600: NSCQ Initialization Failed

WHAT THIS CHANGES FOR OUR PUBLIC CLAIMS: nothing yet. We still do not know
whether the NVLink fabric is attestable by a tenant, so we keep writing that it
is NOT verified on our 8-GPU nodes, and we keep saying the eight GPUs ARE
attested individually (10 September, NRAS, ppcie_mode False).

WHAT IT ADDS: a tenant following NVIDIA's own guide has no supported path. The
image has the driver without NSCQ, the matching NSCQ will not install, and the
one that does install comes from another branch. That is now the open question
with NVIDIA, asked publicly on the issue.

Full unedited session output: run.log
