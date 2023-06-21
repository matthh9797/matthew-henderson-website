---
title: GCP Cloud SQL
weight: 210
menu:
  notes:
    name: Cloud SQL
    identifier: notes-gcp-cloudsql
    parent: notes-gcp
    weight: 10
---

<!-- Deletion -->
{{< note title="Delete an Instance" >}}
By default Cloud SQL instances are created with deletion protection. To delete an instance you must first remove deletion protection using the gcloud cli. Run the following code in gcloud with your instance name to delete a Cloud SQL instance.

```bash
gcloud sql instances patch [INSTANCE_NAME] --no-deletion-protection
gcloud sql instances delete [INSTANCE_NAME]
```
{{< /note >}}
