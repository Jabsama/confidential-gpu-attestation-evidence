NVSwitch attestation attempt from inside a Confidential VM
8x H100 node (h100-xlarge), workload wrk-a2xmzy14yevl, host 199.73.48.132
16 September 2026, tool: nvattest 1.2.0 (apt package "nvattest")

WHY: on 10 September the Python SDK (nv-attestation-sdk) attested all 8 GPUs
but rejected the switch evidence with errorCode 4005 INVALID_EVIDENCE. The
infrastructure operator reported that `nvattest attest --device nvswitch`
works on their 8x H200, same Hopper HGX generation, four LS10 switches per
node. This run tests that claim from inside the tenant guest.

RESULT 1, NVSwitch: NOT ATTESTABLE FROM THE GUEST TODAY.
  libnvidia-nscq is absent from the ubuntu-24-04-lts-595 image. After
  installing it (615.71.09) the failure moves one layer down:
    [switch/nscq_client.cpp:195] Failed to create NSCQ session:
        NSCQ_RC_WARNING: RDT init failure (Code 1)
    [switch/evidence.cpp:178] Failed to initialize NSCQ
    Error 600: NSCQ Initialization Failed
  The four switches ARE present: [10de:22a3] on the bus, /dev/nvidia-nvswitch0
  through 3 in the guest. The blocker is the NSCQ session, not the hardware.
  Open question with the operator: is switch attestation host-side only?

RESULT 2, GPU via the CLI: BLOCKED BY EGRESS, NOT BY THE PLATFORM.
    [nv_http.cpp:142] Fatal libcurl error code: SSL connect error (35)
    [nv_ocsp.cpp:163] Failed to perform OCSP check with url:
        https://ocsp.ndis.nvidia.com
    [rim.cpp:265] Failed to generate certificate chain claims for RIM with id
        NV_GPU_DRIVER_GH100_595.71.05
    GPU attestation failed! Error 002
  Same intermittent egress problem measured on 16 September from h100-small
  (14 failures in 40 OCSP requests). A 5 shot curl loop against
  rim.attestation.nvidia.com during this session: four HTTP 200, one timeout
  at 12 s. The Python SDK in REMOTE mode still succeeds, because NRAS performs
  the revocation checks on its own side.

WHAT THIS CHANGES FOR OUR PUBLIC CLAIMS:
  Keep writing that NVSwitch is NOT verified on 8 GPU nodes. We now know the
  precise reason, which is better than "it failed": the tenant cannot open an
  NSCQ session. Do not claim the NVLink fabric is attested.
  The 8 GPUs themselves remain attested (10 September, NRAS, ppcie_mode False,
  token 17522 bytes), see ../ppcie-8x-h100/.

Cost of this run: 3 USD of provider credit, node up 12 minutes.
