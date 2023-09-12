# Cert-manager and Routes

This repository demonstrates how to use [cert-manager] to automatically provision [LetsEncrypt] certificates for secure Routes.

[cert-manager]: https://cert-manager.io/

## Overview

We take advantage of the fact that OpenShift has support for [Ingress] resources; it handles then by translating them to an equivalent [Route] resource. As part of this process, it will extract certificates and keys from the secret associated with an Ingress resource and inject them into the Route.

[Ingress]: https://kubernetes.io/docs/concepts/services-networking/ingress/
[Route]: https://docs.openshift.com/container-platform/4.13/rest_api/network_apis/route-route-openshift-io-v1.html#route-route-openshift-io-v1

## The plan

1. Create a certificate issuer.

    An [Issuer] manages [Certificate] resources and issues certificates into Secret resource. For this demo, we're using an issuer that can solve [http-01] type challenges:

    ```
    apiVersion: cert-manager.io/v1
    kind: Issuer
    metadata:
      name: demo-issuer
    spec:
      acme:
        server: https://acme-staging-v02.api.letsencrypt.org/directory
        privateKeySecretRef:
          name: issuer-account-key
        solvers:
        - http01:
            ingress:
              ingressClassName: openshift-default
    ```

    Note that this issue is configured to use the LetsEncrypt staging servers by default to avoid triggering any rate limits; that means the resulting certificates will not be trusted. Once you're confident things work as you want you can switch over to the production servers.

    [issuer]: https://cert-manager.io/docs/concepts/issuer/
    [certificate]: https://cert-manager.io/docs/concepts/certificate/
    [http-01]: https://letsencrypt.org/docs/challenge-types/#http-01-challenge

1. Create an Ingress resource. Annotate it with the name of the issuer using the `cert-manager.io/issuer` annotation:

    ```
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: demo-server
      annotations:
        cert-manager.io/issuer: demo-issuer
    spec:
      tls:
      - hosts:
          - demo-server.shift.oddbit.com
        secretName: demo-server-certificate
      rules:
      - host: demo-server.shift.oddbit.com
        http:
          paths:
          - backend:
              service:
                name: demo-server
                port:
                  name: http
            path: /
            pathType: Prefix
    ```

1. The annotation will cause cert-manager to create a certificate. Shortly after you submit the Ingress resource you should see:

    ```
    NAME                      READY   SECRET                    AGE
    demo-server-certificate   False   demo-server-certificate   10s
    ```

    Wait a moment (typically seconds, not minutes), and you will see that the certificate is ready:

    ```
    NAME                      READY   SECRET                    AGE
    demo-server-certificate   True    demo-server-certificate   4m43s
    ```

1. Creating the Ingress resource will cause the OpenShift ingress operator to create a corresponding Route resource. The Ingress operator will extract the TLS key and certificate from the secret referenced in the Ingress and inject them into the appropriate places in the Route:

    ```
    apiVersion: route.openshift.io/v1
    kind: Route
    metadata:
      name: demo-server-f5mzt
    spec:
      host: demo-server.shift.oddbit.com
      path: /
      port:
        targetPort: http
      tls:
        certificate: |
          -----BEGIN CERTIFICATE-----
          ...
          -----END CERTIFICATE-----
        insecureEdgeTerminationPolicy: Redirect
        key: |
          -----BEGIN RSA PRIVATE KEY-----
          ...
          -----END RSA PRIVATE KEY-----
        termination: edge
      to:
        kind: Service
        name: demo-server
        weight: 100
      wildcardPolicy: None
    ```

At this point you should be able to access your service via https at the given hostname.

## Deploying this repository

**Ensure you update `ingress.yaml` with a hostname that you control.**

Deploy the manifests:

```
kubectl apply -k .
```
