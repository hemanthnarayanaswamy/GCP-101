# Google Cloud Service Accounts

Service accounts are a special type of account used by applications and services. Non-human access to Google Cloud APIs and services is usually done via service accounts. They are created and managed within projects like most other resources. Because they are typically used by services, they don’t have an associated password and cannot log in via browser or cookies.

Authentication is done via private/public key pairs (either Google or customer-managed) or identity federation. 

![img](https://storage.googleapis.com/gweb-cloudblog-publish/images/WASA.max-2200x2200.png)

## Service account types
Some types of service accounts are built into Google Cloud services.

1. **User-managed**: Created by you and managed like all other resources. No IAM role is assigned by default. Can be used via key, VM association, or impersonation.
2. **Service default**: Created at API activation. Used by default when no customer service account is selected. For example, Compute Engine has a default service account for VMs. They have a fixed naming convention, and an editor IAM role is assigned at creation.
3. **Google-managed** (robots or service agents): Created at API activation. Used by Google Cloud services to perform actions on customer resources so they are created with specific IAM roles assigned. The Compute Engine robot account is an example of a Google-managed service account. 

## Service account credentials
There are different ways of managing and accessing service account credentials.

1. **Google-managed keys**: Both the public and private portions of the key pair are stored in Google Cloud, auto-rotated, and secured. They can be used by associating a service account with a VM or other compute service, or by impersonation from a different identity.
2. **User-managed keys**: You (as the customer) own both public and private portions and are responsible for rotating and securing them. Key pairs can be created from Google Cloud, or created externally and the public portion is uploaded to Google Cloud. 

It is a best practice to use short-lived credentials when you need to grant limited access to resources for trusted identities.

<table><thead><tr><th>Type</th><th>Who manages</th><th>Typical use cases</th><th>Privileges</th><th>Creation / Deletion</th></tr></thead><tbody><tr><td>User-managed service accounts</td><td>You (project admins/dev teams)</td><td>Custom applications, ETL pipelines, CI/CD, VM workloads — any workload identity you control</td><td>You define roles and permissions; best for least-privilege enforcement</td><td>Created and deleted by your team (manual or IaC)</td></tr><tr><td>Default service accounts</td><td>Auto-created by GCP when certain services are used</td><td>Used by Google Cloud services that need a service identity (e.g., Compute Engine default account)</td><td>Often granted broad permissions by default — can be overprivileged</td><td>Created automatically; deleting may break services that depend on them</td></tr><tr><td>Google-managed service accounts</td><td>Fully managed by Google</td><td>Internal support for Google-managed services (e.g., App Engine internal accounts)</td><td>Permissions are fixed by Google and not configurable by project admins</td><td>Creation and deletion controlled by Google</td></tr></tbody></table>

![img](https://docs.cloud.google.com/static/docs/authentication/images/authn-tree.svg)

## Service Accounts best practices

<table><thead><tr><th>Best practice</th><th>Why it matters</th></tr></thead><tbody><tr><td>Use user-managed service accounts per workload</td><td>Easier auditing, clearer ownership, and tighter permission scoping</td></tr><tr><td>Disable automatic default service account creation where feasible</td><td>Prevents unexpected over-privileged accounts and forces explicit identity creation</td></tr><tr><td>Grant the minimal permissions required</td><td>Limits blast radius from bugs or compromise (principle of least privilege)</td></tr><tr><td>Use structured naming and ownership</td><td>One account per app/environment simplifies incident response and audits</td></tr><tr><td>Enable audit logs and monitoring</td><td>Detect and attribute automated activity; correlate Cloud Audit Logs and app logs</td></tr></tbody></table>

- From a workflow perspective, the default service account is generous with permissions (i.e. Project Editor). It’s a good idea to create app-specific accounts, and only grant needed permissions.
- Service accounts can be used for selective applications to apply firewalls. For example: Open port 443 (HTTPS) for VMs for service account ‘webapp-fe’
- Create service accounts on dedicated projects for centralized management.
- A security risk related to user-managed keys is keys being compromised, either maliciously or by mistakenly publishing keys by embedding them in code. To help mitigate this risk, rotate keys frequently.
- VPC Service Controls help limit who can access Google Cloud services (which is what service accounts are ultimately for). For example: Access only permitted from on-prem IP ranges (when interconnecting). Implementing these access limitations can help minimize your attack surface. 
- Combine service accounts with a proactive approach by using Forseti to alert on old keys that need to be rotated.