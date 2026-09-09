# Google Cloud Resource Hierarchy (with Billing)

GCP automatically assigns the **Project Creator and Billing Account Creator** IAM roles to all users in domain. This allows any user to create projects and enable billing for the cost of resources.

![img](https://docs.cloud.google.com/static/billing/docs/images/resource-hierarchy-overview.png)

## Billing Accounts

**Billing Accounts** is mandatory for creating resources in a project:
- Billing Account contains the payment details.
- Every Project with active resources should be associated with a Billing Account. 

Billing Account can be associated with one or more projects. 
* You can have multiple billing accounts in an Organization. A startup can have just one billing account while a large enterprise can have a separate billing account for each department. 

Two Types of Billing Account:
1. **Self Serve**: Billed Directly to Credit Card or Bank Account. 
2. **Invoiced**: Generate invoices (Used by large enterprises).

## Managing Billing - Budget, Alerts and Exports

* Setup a **Cloud Billing Budget** to avoid surprises and configure **Alerts** to recevice email alerts at different usage thresholds.
* Export your billing data to `BigQuery`. Export your usage and cost data to a BigQuery dataset, and use the dataset for detailed analyses. You can also visualize your exported data in tools such as Data Studio.
* *Review anomalies for your projects*. Anomalies are spikes or deviations in usage costs that differ from your expected spend, when compared to historical spending patterns. The Anomalies dashboard displays all cost anomalies associated with your projects, within the linked billing account.

## Optimize and Control Costs
1. **Use AI-powered features to monitor and optimize costs**. Cloud Billing uses AI to provide insights into your spending, such as AI-driven cost forecasts, anomaly detection, the AI Cost Summary Agent, and Gemini Cloud Assist insights in Billing Reports and the FinOps hub.
2. **Create a spend cap budget to automatically pause usage and cost accrual**. Spend cap budgets help reduce unexpected or runaway costs for eligible services. When a spend cap budget is enforced, usage of your specified services is automatically paused until you manually lift the spend cap, stopping charges for the service in the project where you set the spend cap budget.
3. **View the FinOps hub for recommendations and utilization insights**. With the FinOps hub, you can monitor and communicate your current savings, explore recommended opportunities to optimize costs, gain utilization insights for potential wasted usage, and plan your optimization goals.
4. **Sign up for `Committed use discounts` (CUDs)**. If your workloads have predictable resource needs, you can purchase a Google Cloud commitment, which gives you discounted prices in exchange for your commitment to use a minimum level of resources for a specific term.

Two types of relationships govern the interactions between organizations, Cloud Billing accounts, and projects: ownership and payment linkage.
- **Ownership** refers to IAM permission inheritance.
- **Payment linkages** define which Cloud Billing account pays for a given project.

![img](https://docs.cloud.google.com/static/billing/docs/images/access-control-org.png)

In the diagram, the Organization node has ownership over Projects 1, 2, and 3, meaning that it is the IAM permissions parent of the three projects.

The Cloud Billing account is linked to Projects 1, 2, and 3, meaning that it pays for costs incurred by the three projects.

## Resource Hierarchy and IAM Policy

![img](https://d33wubrfki0l68.cloudfront.net/be8814e34d568f920e3f080a79aa36bf236c7e35/0e36b/gcpimages/02-architecture/00-policy-role-resource.png)

* IAM policy can be set at any level of the hierarchy. 
* Resources inherit the policies of ALL PARENTS. 
* The effective policy for a resource is the union of the policy on that resource and its parents
* ***You can’t restrict policy at lower level if permission is given at an higher level***