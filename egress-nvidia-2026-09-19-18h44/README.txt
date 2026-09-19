Can a tenant TDX guest reach NVIDIA's attestation endpoints? Measured, not assumed.
19 September 2026, 18:44 UTC, workload wrk-q7wu4wopvsao, image ubuntu-24-04-lts-595.
Machine rented and released for this run: 2.4 minutes, 0.07 dollars.

WHY THIS RUN EXISTS

Earlier today, with the driver-matched NSCQ library installed, NVSwitch
attestation on an 8x H100 node got past the NSCQ session and collected switch
evidence, then failed at certificate chain validation:

  [nv_http.cpp:142] Fatal libcurl error code: SSL connect error (35)
  [nv_ocsp.cpp:163] Failed to perform OCSP check with url:
                    https://ocsp.ndis.nvidia.com

NVIDIA then confirmed on nvtrust issue #153 that the RIM and OCSP endpoints ARE
meant to be reachable from the tenant guest, and that nvattest accepts
--ocsp-url, --rim-url and --nras-url.

That made the next question a network question, and a network question does not
need an NVSwitch. So we answered it on the smallest confidential VM available,
1.69 dollars an hour instead of 24.72, rather than rent an eight-GPU node to
run curl.

RESULT: ALL THREE ENDPOINTS ARE FULLY REACHABLE FROM A CONFIDENTIAL VM

  General internet egress works:
    https://www.google.com -> HTTP 200 in 0.098 s

  ocsp.ndis.nvidia.com
    DNS: 166.117.113.121, 166.117.200.57
    Connected on port 443, TLSv1.3, TLS_AES_128_GCM_SHA256, X25519, RSASSA-PSS
    HTTP/2 stream opened

  rim.attestation.nvidia.com
    DNS: 166.117.56.89, 166.117.155.59
    Connected on port 443, TLSv1.3, TLS_AES_128_GCM_SHA256, X25519, RSASSA-PSS
    HTTP/2 stream opened

  nras.attestation.nvidia.com
    DNS: 34.120.45.54
    Connected on port 443, TLSv1.3, TLS_AES_256_GCM_SHA384, X25519, RSASSA-PSS
    HTTP/2 stream opened

  Client stack: curl 8.5.0, OpenSSL 3.0.13, system CA bundle present at
  /etc/ssl/certs/ca-certificates.crt

WHAT THIS RULES OUT, AND WHAT IT LEAVES

  Ruled out: a blanket egress policy blocking NVIDIA's attestation endpoints
  from confidential VMs on this platform. From this SKU, in this image, they
  resolve and complete a TLS 1.3 handshake without help.

  Still open, and the next thing we will measure on an eight-GPU node itself:
  whether that node has the same egress path as this one, and whether nvattest
  uses the system trust store or one of its own. A TLS error 35 at connect time
  with a working system curl points at one of those two, not at the platform.

  We have claimed nothing new about the NVLink fabric. It remains unverified on
  our public pages.

FILES

  run.log      raw terminal output of the whole run, unedited
  SHA256SUMS   checksums of the files in this folder

Licence CC BY 4.0. Sequence: nvswitch-2026-09-16 (original failure),
nscq-retest-2026-09-19 (uninstallable branch package),
nscq-retest-2026-09-19-10h11 (NSCQ works, OCSP fails), this folder.
Issue: https://github.com/NVIDIA/nvtrust/issues/153
