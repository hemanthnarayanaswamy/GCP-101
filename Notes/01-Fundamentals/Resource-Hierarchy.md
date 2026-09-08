# Google Cloud Resource Hierarchy

The Google Cloud resource hierarchy provides a structured way to organize your cloud resources.

The hierarchy consists of the organization (root) at the top, followed by folders (optional) for grouping, and then projects, which contain the actual service resources like Compute Engine virtual machines and storage buckets.

> Organization > Folder > Project > Resources

![img](https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcRqJ6K5wqxJsZBxwt6-QVvykVNcCQB3wcvs0nvG01zkU2ff3qm50jzv2Q0&s=10)

### 1. Organization

- Every google workspace account has only one organization associated with it. 
- We can provide roles for users at the organizational level, these roles are inherited by all projects and folders that are present in the organization.

### 2. Folders

- Folders provide an additional boundary and also separate one project from the other.
- Folders contain multiple projects and other sub-folders.
- If we grant a role at the folder level then this role will be inherited by all the projects and sub-folders that exist in that parent folder.

### 3. Projects

- Project is the core organizational component of Google Cloud.
- The resources we are using belong to one specific project.
- We can enable billing and also set billing alerts at the project level
- Track resources and their usage at the project level

In Google Cloud, the project is a global entity and every project has 3 identifying attributes
* **PROJECT ID** - Globally unique, chosen by you, Immutable. 
* **PROJECT NAME** - Uniqueness not Required, chosen by you, Mutable.
* **PROJECT NUMBER** - Globally unique, Assigned by GCP, Immutable.

We can change the project ID during the creation of the project, once the project got created we cannot change the project ID.
If we delete a project and again try to create a new project with the same name then this time GCP will assign a different project ID.
Project name is always in lowercase.

The main container for all resources (VMs, Cloud Storage, Databases, etc.).

Each project has this entities:
- A unique Project ID
- Billing association
- IAM policies

![img](https://d33wubrfki0l68.cloudfront.net/eaddeba5e864fe63444fe247f7a7277b427e42c2/ed88b/gcpimages/02-architecture/resource-hierarchy-overview.png)