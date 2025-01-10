# Kubernetes Certificate Management - Revision Notes

## 1. **Certificate Authority (CA)**
- **Purpose**: Acts as the root of trust in a Kubernetes cluster.
- **Files**:
  - `ca.key`: Private key of the CA, used to sign certificates.
  - `ca.crt`: Public certificate of the CA, distributed to all components to verify certificates signed by the CA.
- **Command to create CA**:
  ```bash
  openssl req -x509 -new -nodes -key ca.key -subj "/CN=Kubernetes-CA" -days 10000 -out ca.crt
  ```
  - **Self-signed**: The CA signs its own certificate since it is the root of trust.

---

## 2. **Component Certificates**
- Each component (e.g., API server, kubelet) requires its own certificate and private key for secure communication.

### **Why Use CSRs for Components?**
- Ensures certificates are signed by the CA, creating a centralized trust system.
- Allows all components to trust each other via the CA.
- Avoids the complexity of managing self-signed certificates for each component.

### **Steps to Create Certificates for Components**
1. **Generate a Private Key**:
   ```bash
   openssl genrsa -out <component>.key 2048
   ```
   Example: `kubelet.key`, `apiserver.key`.

2. **Create a Certificate Signing Request (CSR)**:
   ```bash
   openssl req -new -key <component>.key -out <component>.csr -subj "/CN=<component-name>"
   ```
   Example: `/CN=kubelet`, `/CN=kube-apiserver`.

3. **Sign the CSR Using the CA**:
   ```bash
   openssl x509 -req -in <component>.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out <component>.crt -days 365
   ```
   - This creates a signed certificate (e.g., `kubelet.crt`, `apiserver.crt`).

---

## 3. **Where Certificates Are Stored and Configured**
### **API Server**
- **Files**:
  - `apiserver.key`: Private key.
  - `apiserver.crt`: Certificate signed by the CA.
  - `ca.crt`: Used to verify client certificates.
- **Default Path**: `/etc/kubernetes/pki/`
- **Configuration** (in `kube-apiserver.yaml`):
  ```yaml
  --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
  --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
  --client-ca-file=/etc/kubernetes/pki/ca.crt
  ```

### **Kubelet**
- **Files**:
  - `kubelet.key`: Private key.
  - `kubelet.crt`: Certificate signed by the CA.
  - `ca.crt`: Used to verify the API server’s certificate.
- **Default Path**: `/var/lib/kubelet/` or `/etc/kubernetes/pki/`
- **Configuration** (in `kubelet.config.yaml`):
  ```yaml
  client-certificate: /var/lib/kubelet/kubelet.crt
  client-key: /var/lib/kubelet/kubelet.key
  certificate-authority: /etc/kubernetes/pki/ca.crt
  ```

---

## 4. **How Secure Communication Happens**
### **Example: Kubelet and API Server**
1. **Mutual TLS Setup**:
   - The kubelet acts as a client.
   - The API server acts as a server.

2. **Authentication Process**:
   - Kubelet connects to the API server over HTTPS.
   - API server sends its certificate (`apiserver.crt`).
     - Kubelet verifies it using the CA certificate (`ca.crt`).
   - Kubelet presents its own certificate (`kubelet.crt`).
     - API server verifies it using the CA certificate (`ca.crt`).

3. **Encrypted Communication**:
   - Once both parties authenticate, all communication is encrypted over the TLS channel.

---

## 5. **Key Files Summary**
| Component       | File                           | Purpose                                      |
|------------------|--------------------------------|----------------------------------------------|
| **CA**          | `ca.key`                       | Signs all certificates.                     |
|                  | `ca.crt`                       | Verifies certificates across the cluster.   |
| **API Server**   | `apiserver.key`                | Private key for the API server.             |
|                  | `apiserver.crt`               | Certificate for the API server.             |
| **Kubelet**      | `kubelet.key`                 | Private key for kubelet.                    |
|                  | `kubelet.crt`                 | Certificate for kubelet.                    |

---

## 6. **Why Use a CA and CSRs?**
- **Centralized Trust**: The CA ensures all components trust each other.
- **Scalability**: New components can request certificates without needing manual trust setup.
- **Security**: Private keys are kept local to each component, and certificates are centrally verified by the CA.

---

## 7. **Testing and Validation**
1. **Inspect Component Certificates**:
   ```bash
   openssl x509 -in <component>.crt -text -noout
   ```
   - Verify the issuer matches the CA.
   - Confirm the CN reflects the component’s identity.

2. **Test Communication**:
   - Check logs to ensure secure connections (e.g., `kubelet` and `API server`).
   - Verify components are accepting certificates signed by the CA.

---

This guide summarizes the CA and certificate-based security in Kubernetes. It ensures you understand the purpose, process, and configuration for securing communication between cluster components.

