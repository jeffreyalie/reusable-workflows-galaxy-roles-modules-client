	 	  
	 	

## For automation:

1. For GHA /.gitea/workflows/\*.yml  
   1.  add in org or repo level secrets  
   2. define in vault (openbao)  
2. For manual  
   1. Export all in environment

## Secrets variable: 

LXD\_ADDRESS

LXD\_CLIENT\_CERT

LXD\_CLIENT\_KEY

MINIO\_ACCESS\_KEY

MINIO\_SECRET\_KEY

MINIO\_ENDPOINT

ANSIBLE\_SSH\_PRIVATE\_KEY

ANSIBLE\_SSH\_PUBLIC\_KEY

LXD\_TRUST\_PASSWORD

##  Secret values: 

* LXD\_ADDRESS  
  * ip \-4 addr  
  * 10.0.0.162

  

* LXD\_CLIENT\_CERT  
  * cat \~/lxd-certs/gitea-runner.crt

* LXD\_CLIENT\_KEY  
  * cat \~/lxd-certs/gitea-runner.key

* MINIO\_ACCESS\_KEY  
  * Minio admin username:  
  * minioadmin

* MINIO\_SECRET\_KEY  
  * Minio admin password:  
  * minioadmin

* MINIO\_ENDPOINT  
  * Minio API server address  
  * http://10.248.42.22:9000

* ANSIBLE\_SSH\_PRIVATE\_KEY  
  * cat \~/.ssh/ansible\_id\_ed25519

* ANSIBLE\_SSH\_PUBLIC\_KEY  
  * cat \~/.ssh/ansible\_id\_ed25519.pub

* LXD\_TRUST\_PASSWORD  
  * Via LXD web UI

## How to get: LXD\_CLIENT\_CERT and LXD\_CLIENT\_KEY

### **Step 1 — Generate a Client Certificate**

Run this on your LXD host:

bash

\# Create a directory for the cert

mkdir \-p \~/lxd-certs && cd \~/lxd-certs

\# Generate client cert \+ key (valid 10 years)

openssl req \-x509 \-newkey ec \\

  \-pkeyopt ec\_paramgen\_curve:secp384r1 \\

  \-sha384 \-keyout gitea-runner.key \\

  \-out gitea-runner.crt \\

  \-days 3650 \-nodes \\

  \-subj "/CN=gitea-runner"

---

### **Step 2 — Trust the Certificate in LXD**

bash

\# Add the cert as a trusted client

lxc config trust add \~/lxd-certs/gitea-runner.crt \--name gitea-runner

Or via **LXD Web UI:**

Settings → Trusted clients → Add certificate → paste gitea-runner.crt content

Verify it's trusted:

bash

lxc config trust list

---

### **Step 3 — Add Secrets to Gitea**

You need **3 secrets** this time:

| Secret Name | Value | How to get it |
| ----- | ----- | ----- |
| `LXD_CLIENT_CERT` | Content of `gitea-runner.crt` | `cat ~/lxd-certs/gitea-runner.crt` |
| `LXD_CLIENT_KEY` | Content of `gitea-runner.key` | `cat ~/lxd-certs/gitea-runner.key` |
| `LXD_ADDRESS` | Your host IP | `hostname -I | awk '{print $1}'` |
| `ANSIBLE_SSH_PUBLIC_KEY` | Your ansible pub key | *(already planned)* |

## How to get ANSIBLE\_SSH\_PUBLIC\_KEY:

### **Step-by-Step: Adding the Secret**

**Step 1 — Generate your SSH key pair** (if you haven't already):

```bash  
ssh-keygen \-t ed25519 \-C "ansible" \-f \~/.ssh/ansible\_id\_ed25519  
# Press Enter twice for no passphrase (recommended for automation)
```
This creates two files:

* `~/.ssh/ansible_id_ed25519` → **private key** (keep this safe, never share)  
* `~/.ssh/ansible_id_ed25519.pub` → **public key** (this goes into Gitea)

**Step 2 — Copy the public key content:**

```bash  
cat \~/.ssh/ansible\_id\_ed25519.pub
```
You'll see something like:
```bash  
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... ansible
```
Copy that entire line.

**Step 3 — Open your Gitea repository in the browser:**

http://\<your-gitea-host\>/your-username/lxd-vm-terraform

**Step 4 — Navigate to:**

Settings → Secrets and Variables → Actions → New Secret

**Step 5 — Fill in the form:**

| Field | Value |
| ----- | ----- |
| **Name** | `ANSIBLE_SSH_PUBLIC_KEY` |
| **Value** | Paste the full public key line |

Click **Add Secret**.

---

### **How It Flows into Terraform**

In the workflow file (`.gitea/workflows/terraform.yml`), the secret is referenced like this:

yaml  
env:  
  TF\_VAR\_ansible\_ssh\_public\_key: ${{ secrets.ANSIBLE\_SSH\_PUBLIC\_KEY }}

Gitea injects it as an environment variable at runtime → Terraform picks it up as `var.ansible_ssh_public_key` → it gets templated into `cloud-init/user-data.yaml` → cloud-init writes it to `/home/ansible/.ssh/authorized_keys` inside the VM.

---

### **Important Points**

* **Public key is NOT sensitive** technically, but keeping it in Gitea Secrets is clean practice and avoids cluttering your code.  
* **Never put the private key** (`ansible_id_ed25519`) anywhere near Gitea or Terraform.  
* The private key stays on the machine that will run Ansible playbooks against the VM. You'd SSH in like:

bash  
 ssh \-i \~/.ssh/ansible\_id\_ed25519 ansible@\<VM-IP\>

* Secrets are **masked in pipeline logs** — Gitea will never print the value even if you accidentally `echo` it.

