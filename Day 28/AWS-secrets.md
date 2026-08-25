apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: catalog-spc
  namespace: catalog
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "$SECRET_NAME"  
        objectType: "secretsmanager"
        jmesPath:
          - path: username
            objectAlias: username
          - path: password
            objectAlias: password
    usePodIdentity: "true"
  secretObjects:
    - secretName: catalog-secret
      type: Opaque
      data:
        - objectName: username
          key: username
        - objectName: password
          key: password

--

# 2. `apiVersion`

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
```

This tells Kubernetes which API/version you're using for the **SecretProviderClass** resource.

`SecretProviderClass` is a custom Kubernetes resource provided by the Secrets Store CSI Driver.

---

# 3. `kind`

```yaml
kind: SecretProviderClass
```

You're creating a `SecretProviderClass`.

Think of this as a **configuration/instruction telling the CSI driver where and how to retrieve secrets**.

It isn't the actual secret.

---

# 5. AWS provider

```yaml
spec:
  provider: aws
```

This tells the Secrets Store CSI Driver:

> Use AWS as the external secret provider.

In your case, the actual secret is stored in **AWS Secrets Manager**.

---

# 6. `objects`

This is the most important section.

```yaml
parameters:
  objects: |
    - objectName: "$SECRET_NAME"
      objectType: "secretsmanager"
```

You're telling the AWS provider:

> Go to AWS Secrets Manager and retrieve the secret whose name is `$SECRET_NAME`.

For example, if:

```text
SECRET_NAME=prod/catalog/database
```

then effectively you're asking AWS for:

```text
prod/catalog/database
```

### Important point about `$SECRET_NAME`

Kubernetes itself does **not automatically substitute `$SECRET_NAME`** here.

If this YAML is being processed by something like a Helm chart, CI/CD pipeline, `envsubst`, etc., that system may replace it.

For example:

```text
$SECRET_NAME
       ↓
prod/catalog/database
```

If you directly run:

```bash
kubectl apply -f file.yaml
```

don't assume `$SECRET_NAME` will be expanded.

---
---

# 8. `jmesPath`

Now suppose your AWS secret is:

```json
{
  "username": "cataloguser",
  "password": "MyPassword123"
}
```

You don't necessarily want the entire JSON object exposed as one value.

You want:

```text
username → cataloguser
password → MyPassword123
```

That's what this does:

```yaml
jmesPath:
  - path: username
    objectAlias: username

  - path: password
    objectAlias: password
```

### First one

```yaml
- path: username
  objectAlias: username
```

Means:

> From the JSON secret, find the `username` field and expose it using the alias `username`.

So:

```text
AWS Secret:

{
  "username": "cataloguser",
  "password": "MyPassword123"
}

             ↓

username = cataloguser
```

### Second one

```yaml
- path: password
  objectAlias: password
```

Means:

> Extract the `password` field and call the resulting object `password`.

So:

```text
password = MyPassword123
```

---

# 9. `usePodIdentity`

```yaml
usePodIdentity: "true"
```

This is an important EKS security piece.

You're saying:

> Use EKS Pod Identity to obtain AWS credentials for this pod.

The application pod doesn't need an AWS access key and secret key hardcoded in the Deployment.

Instead:

```text
Pod
 │
 │ Pod Identity
 ▼
IAM Role
 │
 │ GetSecretValue
 ▼
AWS Secrets Manager
```

The IAM role should have permission similar to:

```json
{
  "Effect": "Allow",
  "Action": [
    "secretsmanager:GetSecretValue"
  ],
  "Resource": "arn:aws:secretsmanager:region:account:secret:prod/catalog/database-*"
}
```

This is much better than putting AWS credentials inside your Kubernetes YAML.

---


/////

Absolutely. This `SecretProviderClass` is doing something very common in EKS: **reading a JSON secret from AWS Secrets Manager and making its individual fields available as a Kubernetes Secret**.

Let's go line by line.

### 1. What is this doing overall?

Your flow is:

```text
AWS Secrets Manager
        │
        │ GetSecretValue
        ▼
EKS Pod Identity
        │
        ▼
Secrets Store CSI Driver
        │
        ▼
SecretProviderClass
        │
        ├──────────────► mounted secret files
        │
        └──────────────► Kubernetes Secret
                              │
                              ▼
                         catalog-secret
                              │
                              ▼
                         Application Pod
```

Your AWS Secrets Manager secret probably looks something like:

```json
{
  "username": "cataloguser",
  "password": "MyPassword123"
}
```

---



---

# 4. Metadata

```yaml
metadata:
  name: catalog-spc
  namespace: catalog
```

You're creating a resource called:

```text
catalog-spc
```

inside the Kubernetes namespace:

```text
catalog
```

Your Deployment will later refer to this name:

```yaml
secretProviderClass: catalog-spc
```



# 7. `objectType`

```yaml
objectType: "secretsmanager"
```

This says the secret comes from:

```text
AWS Secrets Manager
```

rather than, for example, AWS Parameter Store.


# 10. `secretObjects`

Now we get to an important part:

```yaml
secretObjects:
  - secretName: catalog-secret
    type: Opaque
```

This tells the Secrets Store CSI Driver:

> After retrieving the secret from AWS, create/synchronise a Kubernetes Secret called `catalog-secret`.

So you'll eventually have:

```bash
kubectl get secret -n catalog
```

and see something like:

```text
catalog-secret
```

---

# 11. `data`

You have:

```yaml
data:
  - objectName: username
    key: username

  - objectName: password
    key: password
```

This maps the objects retrieved from AWS to keys in the Kubernetes Secret.

Think of it like:

```text
AWS Secrets Manager
        │
        ▼
username ─────────────► username
password ─────────────► password
                         │
                         ▼
                  catalog-secret
```

So the Kubernetes Secret conceptually contains:

```yaml
username: <base64 encoded cataloguser>
password: <base64 encoded MyPassword123>
```

---

# 12. Then your Deployment can use it

For example:

```yaml
env:
  - name: DB_USERNAME
    valueFrom:
      secretKeyRef:
        name: catalog-secret
        key: username

  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: catalog-secret
        key: password
```

Now your application sees:

```text
DB_USERNAME=cataloguser
DB_PASSWORD=MyPassword123
```

without those values being hardcoded in the Deployment.

---

# 13. Complete flow

This is the part I would remember for an EKS interview:

```text
                AWS
       ┌────────────────────┐
       │  Secrets Manager   │
       │                    │
       │ prod/catalog/db    │
       │                    │
       │ {                  │
       │  username: xxx,    │
       │  password: xxx     │
       │ }                  │
       └─────────┬──────────┘
                 │
                 │ GetSecretValue
                 │
                 ▼
       ┌────────────────────┐
       │  EKS Pod Identity  │
       │                    │
       │     IAM Role       │
       └─────────┬──────────┘
                 │
                 ▼
       ┌────────────────────┐
       │ Secrets Store CSI   │
       │ Driver + AWS ASCP  │
       └─────────┬──────────┘
                 │
                 ▼
       ┌────────────────────┐
       │ SecretProviderClass│
       │                    │
       │ catalog-spc        │
       └─────────┬──────────┘
                 │
                 ▼
       ┌────────────────────┐
       │ Kubernetes Secret  │
       │                    │
       │ catalog-secret     │
       │                    │
       │ username: xxx      │
       │ password: xxx      │
       └─────────┬──────────┘
                 │
                 │ secretKeyRef
                 ▼
       ┌────────────────────┐
       │    Catalog Pod     │
       │                    │
       │ DB_USERNAME        │
       │ DB_PASSWORD        │
       └────────────────────┘
```

### One important interview point

Your `secretObjects` section **does not by itself cause the Kubernetes Secret to exist immediately**.

The `SecretProviderClass` is used when a pod mounts the Secrets Store CSI volume. The CSI driver retrieves the secret, and the `secretObjects` configuration synchronises it into a Kubernetes Secret.

Also, the Kubernetes Secret is still stored in the Kubernetes API server. So if your goal is **"don't store the secret in Kubernetes at all"**, you can use the CSI-mounted files and have the application read them directly, instead of using `secretObjects`.

**Interview answer:**

> "I use AWS Secrets Manager as the source of truth. EKS Pod Identity provides the pod with an IAM role, the Secrets Store CSI Driver with the AWS provider retrieves the secret, `jmesPath` extracts individual JSON fields, and `secretObjects` synchronises those values into a Kubernetes Secret so the application can consume them through `secretKeyRef`."
