# Install cert-manager
```
helm repo add jetstack https://charts.jetstack.io --force-update
helm upgrade --namespace cert-manager --create-namespace --install cert-manager jetstack/cert-manager  --version v1.16.2 -f values.yaml
```
(Ensure you have created cgrcred secret in cert-manager namespace first! )

# Create root CA

Create CA:

```
kubectl apply -f - << EOF
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-selfsigned-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: my-selfsigned-ca
  secretName: root-secret
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
    group: cert-manager.io
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: my-ca-issuer
spec:
  ca:
    secretName: root-secret
EOF
```

# Share the CA cert with trust-manager
Install trust-manager to share CA certs with other namespaces than cert-manager and mount it into pods without risk of accidentally mounting the CA key:

```
helm upgrade trust-manager jetstack/trust-manager --install --namespace cert-manager  --wait -f trust.yaml


kubectl apply -f - <<EOF
apiVersion: trust.cert-manager.io/v1alpha1
kind: Bundle
metadata:
  name: example-bundle
spec:
  sources:
  - useDefaultCAs: true
  - secret:
      name: "root-secret"
      key: "tls.crt"
  target:
    configMap:
      key: "trust-bundle.pem"
EOF
```

Check your trust manager setup:
Get the CA cert:
```
kubectl get -n cert-manager secret root-secret -o"jsonpath={.data['tls\.crt']}" | base64 -d
```
Grab about 5 characters from a line in the output and then:
```
kubectl get configmap example-bundle -o "jsonpath={.data['trust-bundle\.pem']}" | grep <5chars>
```
You should see a similar line to the one from the cert.



# UNINSTALL cert-manager

If you want to uninstall cert-manager:
```
# Delete any of these:
kubectl get Issuers,ClusterIssuers,Certificates,CertificateRequests,Orders,Challenges --all-namespaces

# Once all the objects are gone:
helm uninstall cert-manager -n cert-manager

# And delete the CRDS:
kubectl delete crd \
  issuers.cert-manager.io \
  clusterissuers.cert-manager.io \
  certificates.cert-manager.io \
  certificaterequests.cert-manager.io \
  orders.acme.cert-manager.io \
  challenges.acme.cert-manager.io
```
