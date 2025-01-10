# Kubernetes Certificate Management - Revision Notes

## 1. Why do we need a CA in Kubernetes?
- A CA (Certificate Authority) is the root of trust in a Kubernetes cluster.
- It ensures secure communication between components by signing their certificates.
- Without a CA, how would different components verify each other’s identity?
- This means that the cluster CA is self signed and this is now being used as an authority to issue certificates for other components.
- Remember that ca.key is the secret key used to sign certificates of other components and is privately stored while ca.crt is the public counterpart of ca.key and it establishes trust
- because any certificate signed by ca.key can be verified by ca.crt 

## 2. What is the purpose of `ca.key` and `ca.crt`?
- `ca.key` is the private key used to sign all other certificates.
- `ca.crt` is the CA’s public certificate, shared with all components to verify the certificates signed by the CA.
- Think of `ca.key` as the secret seal of approval and `ca.crt` as the badge everyone trusts.

### How do you create the CA?
```bash
openssl genrsa -out ca.key 2048

openssl req -x509 -new -nodes -key ca.key -subj "/CN=Kubernetes-CA" -days 10000 -out ca.crt
```
- Here, the CA is self-signed because it’s the root of trust.

---

## 3. Why don’t we directly create certificates for components like the CA?
- If every component self-signed its certificate, there would be no central trust.
- By using a CSR (Certificate Signing Request), components ask the CA to validate and sign their certificate.
- This keeps everything centralized and ensures mutual trust across the cluster.

## 4. How do we create certificates for components like the kubelet or API server?

### Step 1: Generate the private key for the component
```bash
openssl genrsa -out <component>.key 2048
```
- Example: `kubelet.key` or `apiserver.key`

### Step 2: Create a CSR
```bash
openssl req -new -key <component>.key -out <component>.csr -subj "/CN=<component-name>"
```
- Example: `/CN=kubelet` or `/CN=kube-apiserver`

### Step 3: Sign the CSR using the CA
```bash
openssl x509 -req -in <component>.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out <component>.crt -days 365
```
- This gives you the signed certificate, e.g., `kubelet.crt` or `apiserver.crt`.
- Now the component’s identity is verified by the CA.

- When a component CSR (e.g., kubelet.csr) is submitted, the CA uses:
    ca.key: To sign the certificate and create component.crt.
    ca.crt: To embed its public identity into component.crt for verification.
  This establishes that the certificate is trusted by the CA.

- Verification of certificates The ca.crt is distributed to all cluster components (API server, kubelet, etc.).
  Components use it to verify that any presented certificate (e.g., apiserver.crt or kubelet.crt) is signed by the trusted CA.

---

## 5. Where do these certificates and keys go?
### API Server
- Files:
  - `apiserver.key`: Private key.
  - `apiserver.crt`: Certificate signed by the CA.
  - `ca.crt`: Used to verify client certificates.
- Configuration (in `kube-apiserver.yaml`):
  ```yaml
  --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
  --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
  --client-ca-file=/etc/kubernetes/pki/ca.crt
  ```

### Kubelet
- Files:
  - `kubelet.key`: Private key.
  - `kubelet.crt`: Certificate signed by the CA.
  - `ca.crt`: Used to verify the API server’s certificate.
- Configuration (in `kubelet.config.yaml`):
  ```yaml
  client-certificate: /var/lib/kubelet/kubelet.crt
  client-key: /var/lib/kubelet/kubelet.key
  certificate-authority: /etc/kubernetes/pki/ca.crt
  ```

---

## 6. How does secure communication happen between components?
### Example: Kubelet and API Server
1. **Mutual TLS Setup**:
   - The kubelet connects to the API server over HTTPS.
   - The API server sends its certificate (`apiserver.crt`).
     - The kubelet verifies it using `ca.crt`.
   - The kubelet sends its own certificate (`kubelet.crt`).
     - The API server verifies it using `ca.crt`.

2. **Encrypted Communication**:
   - After verification, communication is encrypted over the TLS channel.

---

## 7. Why is using a CA and CSRs important?
- Centralized Trust: The CA ensures all components trust each other.
- Scalability: New components can request certificates without extra manual steps.
- Security: Each component keeps its private key private, and the CA ensures authenticity.

---

## 8. How can I test or validate this setup?
### Inspect a certificate
```bash
openssl x509 -in <component>.crt -text -noout
```
- Check if the **Issuer** matches the CA and the **CN** reflects the component’s identity.

### Check logs for secure connections
- Look at the kubelet and API server logs to confirm they are communicating securely.
- Errors often indicate certificate mismatches or trust issues.

---

This guide covers why we need a CA, how it works, and how certificates are created, signed, and used in Kubernetes. It’s a complete overview for understanding secure communication in the cluster.

