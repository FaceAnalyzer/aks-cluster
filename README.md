# AKS cluster for Faceanalyzer

This repo contains configuration for setting up an AKS cluster to which Faceanalyzer application can be deployed.

## Ingress-nginx

Ingress-nginx is a reverse proxy. It enables external access to ingresses in cluster.

1. Navigate to `ingress-nginx/`
2. Run `helmfile apply`

# Cert-manager

Cert-manager issues Let's Encrypt certificates for TLS.

1. Navigate to `cert-manager/`
2. Run `helmfile apply`
