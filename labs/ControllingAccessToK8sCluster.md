![Alt text](https://github.com/cloudbrdesign/eks-nuggets/blob/main/labs/images/Controlling%20access%20to%20the%20kubernetes%20cluster.png)

**The Scenario**
----------------

Imagine you want to tell Kubernetes to create a new Pod --- a basic unit of work. What actually happens behind the scenes? This guide walks through the stages a request goes through, from initial contact to secure execution.

* * * * *

**🚪 1. Establishing a Secure Connection (Transport Security)**
---------------------------------------------------------------

### **✅ Vanilla Kubernetes**

Before Kubernetes even checks who you are, it makes sure your communication is private and authentic.

-   **API Server Address**: You connect to the API server at a specific address (e.g., https://my-kubernetes-cluster:6443). The https:// prefix means the connection is secured using TLS (Transport Layer Security).

-   **Server Certificate (API Server's ID Card)**: The server presents a TLS certificate to prove its identity. If this certificate is signed by a known Certificate Authority (CA), your client trusts it automatically. If it's a private CA (common in private clusters), your ~/.kube/config must include the CA's certificate.

-   **Client Certificate (Optional)**: Some clusters may require you to present a client certificate, adding another layer of trust.

-   **Secure Channel Established**: Once identities are validated, an encrypted channel is established, protecting your request from tampering or eavesdropping.

**2\. Proving Your Identity (Authentication)**
----------------------------------------------

### **✅ Vanilla Kubernetes**

Now that the channel is secure, Kubernetes needs to know **who you are**.

-   **Supported Authentication Methods**:

    -   Client certificates

    -   Static passwords

    -   Bearer tokens (especially for service accounts)

Kubernetes checks each method in turn until one succeeds. If none match, the request is rejected with a 401 Unauthorized.

-   ✅ **Success**: You're identified by a username and optional groups.

-   ❌ **Failure**: You receive a 401 Unauthorized error.

* * * * *

**🔐 3. Are You Allowed? (Authorization)**
------------------------------------------

### **✅ Vanilla Kubernetes**

Next, Kubernetes checks if the authenticated user is allowed to perform the requested action.

-   **What's Evaluated**:

    -   Your **username**

    -   Your **groups**

    -   The **action** (create, delete, get, etc.)

    -   The **resource** (e.g., Pod in a specific namespace)

-   **Policy Check**: Kubernetes checks its authorization modules (RBAC, ABAC, etc.). If at least one says "yes," the request proceeds. If none allow the action, a 403 Forbidden error is returned.

* * * * *

**🧰 4. Final Filters & Mutations (Admission Control)**
-------------------------------------------------------

### **✅ Vanilla Kubernetes**

Even if you're authenticated and authorized, Kubernetes runs your request through **admission controllers** for final checks.

-   **What They Do**:

    -   Validate that your request meets certain requirements

    -   Mutate the request to add default values or labels

    -   Enforce security policies (e.g., disallow privileged containers)

-   **Strict Evaluation**:

    -   They run in order.

    -   A single rejection means your request is denied.

    -   They only act on **write** requests (create/update/delete).

* * * * *

**🧪 5. Validation & Storage**
------------------------------

-   **Schema Validation**: The request is checked against the Kubernetes API schema for that resource type.

-   **State Update**: If valid, the request is written to the etcd store --- the single source of truth for the desired state of your cluster.

* * * * *

**🧾 6. Auditing (Tracking All Requests)**
------------------------------------------

Kubernetes keeps a record of every API interaction --- successful or not.

-   **Audit Logs Include**:

    -   Who made the request

    -   What action was attempted

    -   When it happened

    -   The outcome (success or failure)

This forms a powerful tool for security, debugging, and compliance.

**EKS-Specific Access Control (Kubernetes on AWS)**
===================================================

While the access control flow is similar in EKS, there are key differences in **authentication and identity mapping**, thanks to AWS IAM integration.

* * * * *

**🔐 Secure Connection (Same as Vanilla)**
------------------------------------------

-   **Managed API Server**: AWS manages TLS, certificates, and the control plane.

-   **TLS Encryption**: Secured by default.

-   **Access Setup**: You typically run aws eks update-kubeconfig to configure kubectl with the correct endpoint and IAM-based token provider.

* * * * *

**🧑‍💻 Authentication via AWS IAM**
------------------------------------

### **🔄 How It Works**

1.  You run a kubectl command.

2.  kubeconfig triggers aws eks get-token.

3.  This command:

    -   Uses your local AWS credentials

    -   Contacts AWS STS to get a temporary signed token

4.  The token is passed to the API server.

### **🧩 Webhook Authentication**

-   The EKS control plane uses a webhook to validate the token with AWS STS.

-   If valid, it returns the corresponding IAM identity.

* * * * *

**Mapping IAM to Kubernetes Identity**
--------------------------------------

### **Option 1: **

### **aws-auth**

### ** ConfigMap**

This ConfigMap maps IAM users/roles to Kubernetes usernames and groups.

<pre lang="markdown">
```yaml
mapUsers: |
  - userarn: arn:aws:iam::123456789012:user/Alice
    username: alice
    groups:
      - developers

mapRoles: |
  - rolearn: arn:aws:iam::123456789012:role/AdminRole
    username: eks-admin
    groups:
      - system:masters
```
</pre>

### **Option 2: EKS Access Entries (Recommended)**

-   Managed via EKS APIs

-   IAM identities are mapped via **access policies**

-   Scalable for large orgs and IaC workflows

-   No need to manually edit ConfigMaps

| Stage              | Vanilla Kubernetes                  | AWS EKS                                    |
|--------------------|--------------------------------------|---------------------------------------------|
| Authentication     | Client certs, tokens, passwords      | IAM identity + STS token + webhook          |
| Identity Mapping   | Direct Kubernetes user/group         | `aws-auth` ConfigMap or EKS access entries  |
| Authorization      | RBAC, ABAC, Webhook, etc.            | Same — standard Kubernetes authorization    |


