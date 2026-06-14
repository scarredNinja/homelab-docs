---
project_id: Homelab-2025
status: Archived
phase: Archive
tags:
  - archive
---
tls:
  stores:
    default:
      defaultCertificate:
        certFile: /certs/local-ca.crt
        keyFile: /certs/local-ca.key

  options:
    default:
      minVersion: VersionTLS12
      sniStrict: true
      cipherSuites:
        - TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
        - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
        - TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305
        - TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305
        - TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
        - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256

  certificates:
    # Your internal wildcard certificate
    - certFile: /certs/internal-wildcard.crt
      keyFile: /certs/internal-wildcard.key
      stores:
        - default
