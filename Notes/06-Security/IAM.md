# Cloud IAM Google Cloud

Cloud IAM helps define who can do what and where on Google Cloud. It provides fine-grained access control and visibility for centrally managing cloud resources.

> `IAM` stands for **Identity and Access Management**.
> `IAM` = `Identity` (Who) + `Role` (What they can do) + `Resource` (Where they can do it)

## Identity & Access management: Authentication with Cloud Identity

In security, the three “A”s of controlling access are `Authentication` (Who is the user?), `Authorization` (What is the user allowed to do?), and `Auditing` (What are they doing?)

1. ***Authentication (AuthN)***: Authentication is the process of identifying a user through a private form of verification (for example, a password, a certificate, a key, and so on). In Google Cloud, `Cloud Identity` performs authentication.
2. ***Authorization (AuthZ)***: Authentication on its own provides no set of permissions; authorization is used to set permissions that a user is allocated post authentication. In Google Cloud, `Cloud IAM` is used for authorization (and Cloud Identity for assigning admin roles with broad default permissions).
3. ***Auditing***: Auditing is about monitoring the resources accessed or modified by a particular identity.  In Google Cloud, `Cloud Audit Logging` helps with auditing resources and the Reports API helps auditing in Cloud Identity operations. You can also integrate the logs with prominent SIEM systems.

![img](https://github.com/priyankavergadia/GCPSketchnote/blob/main/images/IAMAuthentication.jpg?raw=true)

---

## What is Cloud Identity ? 
In Google Cloud, an *identity* (also called a principal or member) represents who is requesting access.

<table>
<thead>
<tr>
<th>Type</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td><strong>Google Accounts</strong></td>
<td>Individual user accounts (like <code>user@gmail.com</code>)</td>
</tr>
<tr>
<td><strong>Service Accounts</strong></td>
<td>Non-human accounts used by apps or services</td>
</tr>
<tr>
<td><strong>Google Groups</strong></td>
<td>Group of users managed together</td>
</tr>
<tr>
<td><strong>Google Workspace Accounts</strong></td>
<td>Corporate or organizational accounts</td>
</tr>
<tr>
<td><strong>Cloud Identity Domain</strong></td>
<td>Identity managed without Gmail or Workspace</td>
</tr>
</tbody>
</table>

Cloud Identity is the identity provider (IdP) for Google Cloud. It also is the Identity-as-a-Service (IDaaS) solution that powers Google Workspace. It stores and manages digital identities for Google Cloud users.

Aside from username and password, there are two frequently used authentication options:
1. *2-Step Verification (`2SV`) with Google authentication:* Adds a second factor for authentication in addition to a username and password. However,  not all the 2SV methods are the same. SMS, backup codes, one-time passwords (OTP), and mobile push prompts provide additional protection but they are still phishable.
2. *SSO authentication with a third-party identity provider*: You can also delegate authentication using SSO to a third-party SAML 2.0 identity provider, such as Okta, Ping, Active Directory Federation Services (AD FS), or Azure AD. This method normally means faster Google Cloud onboarding and less disruption if you are already using a compatible IdP.  

---

## Cloud IAM Google Cloud

Once you have identified who a user is (authenticated them) using Cloud Identity, the next step is to define what they can do on Google Cloud (authorize them) so they can access the resources they are permitted to use. Access control for Google Cloud resources is managed by 
1.`Cloud IAM Policies` for humans 
2.`Service Accounts` for non-humans (applications and services)

![iam](https://storage.googleapis.com/gweb-cloudblog-publish/images/image1_copy_3.max-2000x2000.jpg)

#### What are IAM Permissions ?
Access is granted by assigning Roles — and roles contain Permissions. Each permission allows performing a specific operation on a resource. 
Permissions are the most granular level of IAM — the actions that can be performed.

```ini
compute.instances.create → Create a VM instance
compute.instances.list → List all instances
compute.instances.start → Start a VM
compute.instances.stop → Stop a VM
```
Permissions are always grouped inside a Role -- you can't assign a single permission directly to a user. 

#### What are IAM Roles ?
A Role is a collection of permissions.
When you grant a role to a user, you’re granting all the permissions contained in that role.

```ini
Compute Admin Role → Can create, delete, and modify VMs
Storage Admin Role → Can manage storage buckets and objects
```
Roles are what you assign to users, service accounts, or groups to control access.

#### Types of IAM Roles

1. **Basic Roles**: These are the original, broad roles that apply across all GCP services:
- *Owner* Complete control over all resources (can manage billing, IAM, etc.)
- *Editor* Can view and modify resources but not manage IAM
- *Viewer* Read-only access across all services

2. **Predefined Roles**: These are ready-made roles created and maintained by Google. They offer fine-grained access control — limited only to specific services or tasks.

```ini
roles/compute.admin → Full access to Compute Engine
roles/compute.viewer → Read-only access to Compute Engine
roles/compute.networkAdmin → Manage networking components only
```
3. **Custom Roles**: You can create your own roles by combining specific permissions that suit your needs. If you want a role that can start and stop VMs but cannot delete them, you can create a custom role with:
```ini
compute.instances.start
compute.instances.stop
```
Custom roles help you maintain tight security control while keeping flexibility.

<table>
<thead>
<tr>
<th>Concept</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td><strong>Identity (Principal)</strong></td>
<td>Who is requesting access</td>
</tr>
<tr>
<td><strong>Role</strong></td>
<td>What they can do</td>
</tr>
<tr>
<td><strong>Permission</strong></td>
<td>The individual action allowed</td>
</tr>
<tr>
<td><strong>Resource</strong></td>
<td>Where they can perform the action</td>
</tr>
</tbody>
</table>

## Google Cloud IAM Policy

Google Cloud Platform (GCP) Identity and Access Management (IAM) policies define who (principals) has what access (roles) to which Google Cloud resources

## Core Components of an IAM Policy
1. **Principal (Who)**: Users, Google groups, service accounts, or entire domains.
2. **Role (What)**: A collection of individual permissions (e.g., roles/storage.admin).
3. **Resource (Which)**: The GCP asset being accessed, such as a project, folder, organization, or Cloud Storage bucket.
4. **Binding**: Connects principals to a role.
5. **Condition** (Optional): Logical expressions adding context-based constraints, like time or device attributes

Binding principals can be:
- an org domain, granting the role to all org members
- a Workspace/Cloud Identity user
- a Workspace/Cloud Identity group
- a service account (described later)

### IAM Conditions
IAM policies can also be bound to conditions based on resource and request attributes. 

This allows for the following use cases:
- Time-limited access; for example: only allow access during working hours
- Access to a subset of resources; for example: grant access only to VMs prefixed with ‘webapp-frontend-’
- Network address space; for example: only allow access from the corporate network

![iam conditions](https://storage.googleapis.com/gweb-cloudblog-publish/images/What_are_IAM_conditions.max-2200x2200.png)

IAM Conditions also enable granular control on which roles can be assigned or revoked. IAM Conditions also support secure tags. 
Tags are access-controlled key/value resources defined at the organization level, which can be associated with hierarchy nodes (organization, folders, projects). Once tags are associated with a node, they can be set in IAM Conditions to scope role assignment to relevant nodes.

### How IAM Policies Work (The Hierarchy)

Google Cloud resources are organized hierarchically like this:

![img](https://media2.dev.to/dynamic/image/width=800%2Cheight=%2Cfit=scale-down%2Cgravity=auto%2Cformat=auto/https%3A%2F%2Fdev-to-uploads.s3.amazonaws.com%2Fuploads%2Farticles%2Fo738d2t8ogwymt98coyr.png)

You can attach an IAM Policy at any of these levels:
- Organization level
- Folder level
- Project level
- Resource level (for some services like Storage, Compute Engine)

Policies are inherited down the hierarchy. That means:
- If you give someone a role at the organization level, they’ll automatically have that access for all folders, projects, and resources inside it.

```ini
Company (Organization)
 └── Department B (Folder)
      └── Team B (Folder)
           └── Product 1 (Project)
                └── Development VM (Resource)

If you set a policy at each level, the effective IAM policy for the “Development VM” is a combination (union) of all the policies above it:

Effective Policy = Company Policy + Department B Policy + Team B Policy + Product 1 Policy

So if the company allows viewer access and the project gives editor access, the final (effective) permissions include both viewer and editor.
```
### What's Inside an IAM Policy ?
An IAM Policy is a collection of Role Bindings.

A binding simply connects:
- One or more Members (Principals)/Groups
- To a Role
- On a specific Resource

![img](https://media2.dev.to/dynamic/image/width=800%2Cheight=%2Cfit=scale-down%2Cgravity=auto%2Cformat=auto/https%3A%2F%2Fdev-to-uploads.s3.amazonaws.com%2Fuploads%2Farticles%2F5o1qjbnfoay7h35qef0u.png)

### Main Types of GCP IAM Policies

1. ***ALLOW POLICIES*** The standard mechanism to grant principals specific roles and permissions on a resource.
2. ***DENY POLICIES*** Explicitly override allow rules to ensure certain principals can never use specific permissions, regardless of what roles they hold.
3. ***Principal Access Boundary (`PAB`) policies***: Restrict the resources a principal is generally eligible to access across an organization.

### Cloud IAM best practices

The **Principle of least privilege (`PoLP`)** is a security practice that gives users, programs, and systems only the minimum access rights and resources they need to do their jobs, and nothing more

![img](https://storage.googleapis.com/gweb-cloudblog-publish/images/CIBP_1.max-2200x2200.png)

When using Cloud IAM, you should map ***IAM policies to functional identities using groups***.

* Use individual identity groups as recipients of functional sets of IAM roles, with clear permission scopes and boundaries (org, folder, project, resource).
* Use groups to mirror on-premises workflows (networking, DevOps, etc.) or map to new cloud-specific workflows.
* Minimize the points where IAM policies are applied by using folders.

#### Principle of Least Privilege & Role Management
1. **Avoid primitive roles**: Never use broad basic roles like Owner, Editor, or Viewer in production environments.
2. **Use predefined or custom roles**: Choose specific predefined roles or build custom roles tailored to exact operational needs.

#### Identity & Group Configuration
1. **Grant roles to groups**: Assign permissions to Google Groups instead of individual user accounts to streamline onboarding and offboarding. 
2. **Enforce corporate identity**: Use single sign-on (SSO) and multi-factor authentication (MFA) via Google Cloud IAM or federated identity providers instead of personal accounts.
3. **Control resource hierarchy scopes**: Apply policies carefully at the organization or folder level, keeping in mind that permissions inherit downward to child project

#### Service Account Security
1. **Avoid user-managed keys**: Do not create or upload service account keys unless absolutely required; rely on GCP-managed keys or Workload Identity.
2. **Limit service account privileges**: Prevent service accounts from holding higher privileges than the users or workloads invoking them.
3. **Use temporary credentials**: Implement the Service Account Credentials API and Credential Access Boundaries to downscope access tokens.

#### Auditing & Monitoring
1. **Regularly review policies**: Audit access using Cloud Audit Logs, IAM Policy Analyzer, and role recommendations to revoke unused permissions.