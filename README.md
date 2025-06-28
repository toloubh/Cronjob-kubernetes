# Cronjob-kubernetes
CronJobs are a fundamental part of Linux/Unix automation, providing a straightforward way to schedule tasks to run automatically at specific times or intervals.

# What is a CronJob in Kubernetes?
A CronJob in Kubernetes is a resource used to run scheduled tasks within a Kubernetes cluster.
It automates recurring jobs, such as database backups, log rotation, or batch processing, by executing commands or scripts at specified intervals using the cron syntax.
CronJobs are a long-standing feature of Linux and UNIX systems. 
Unlike a standard Kubernetes Job, which runs once and exits, a CronJob schedules jobs to run at fixed times, dates, or intervals.
Kubernetes manages these jobs, ensuring they execute as defined and handling retries if necessary.
Kubernetes also follows job history, concurrency policies, and resource limits, making CronJobs reliable for scheduled automation.

# Kubernetes CronJobs benefits and use cases
The main benefit of using a CronJob in Kubernetes is that it allows you to automate recurring tasks, such as backups, data synchronization, batch processing, and maintenance jobs. To save manual effort, you should use CronJobs everywhere they are the most appropriate option.

When should you use Kubernetes CronJobs?
Some common use cases for CronJobs in K8s include:

    • Data backup — Schedule periodic backups of application or database data, ensuring important information is regularly saved to persistent storage or an external location.
    • Database maintenance — Automate tasks such as database cleanup, reindexing, or data migration to maintain performance and integrity.
    • Log rotation and cleanup — Rotate and manage application log files to prevent excessive growth, which can impact system performance and storage.
    • Certificate renewal — Automate SSL/TLS certificate renewal to ensure applications always use up-to-date and valid certificates for secure communication.
    • Data synchronization — Periodically synchronize data between different systems or databases to keep information updated across services.
    • Scheduled reports — Generate and send periodic reports (daily, weekly, or monthly) to users or stakeholders.
    • Batch processing — Run batch jobs for processing large data volumes at scheduled intervals, such as nightly ETL (Extract, Transform, Load) processes or data imports/exports.
    • Scheduled cleanup — Automate the removal of temporary files, outdated data, or unused resources to optimize system performance and reduce costs.
    • Maintenance tasks — Schedule routine application maintenance tasks, such as database schema updates, software updates, or health checks.
    • Resource scaling — Adjust application resources based on demand, scaling up during peak hours and down during off-peak times.
    • Security scanning — Conduct regular security scans and vulnerability assessments to identify and mitigate risks.
    • Monitoring and alerts — Automate system health checks, collect logs and metrics, and trigger alerts based on collected data.
    • Content publishing — Schedule content updates or publishing for websites, blogs, or content management systems.
    • Compliance audits —Automate compliance checks and audits to ensure adherence to regulatory requirements.
    • Cache invalidation — Schedule cache invalidation or purges to ensure applications serve the latest data to users.
# How to schedule pods restart
I would like to restart the pods of my cluster every morning at 8.00 AM.
You have to setup RBAC, so that the Kubernetes client running from inside the cluster has permissions to do needed calls to the Kubernetes API.

```
---
# Service account the client will use to reset the deployment,
# by default the pods running inside the cluster can do no such things.
kind: ServiceAccount
apiVersion: v1
metadata:
  name: deployment-restart
  namespace: <YOUR NAMESPACE>
---
# allow getting status and patching only the one deployment you want
# to restart
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployment-restart
  namespace: <YOUR NAMESPACE>
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    resourceNames: ["<YOUR DEPLOYMENT NAME>"]
    verbs: ["get", "patch", "list", "watch"] # "list" and "watch" are only needed
                                             # if you want to use `rollout status`
---
# bind the role to the service account
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deployment-restart
  namespace: <YOUR NAMESPACE>
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: deployment-restart
subjects:
  - kind: ServiceAccount
    name: deployment-restart
    namespace: <YOUR NAMESPACE>
```

And the cronjob specification itself:
```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: deployment-restart
  namespace: <YOUR NAMESPACE>
spec:
  concurrencyPolicy: Forbid
  schedule: '0 8 * * *' # cron spec of time, here, 8 o'clock
  jobTemplate:
    spec:
      backoffLimit: 2 # this has very low chance of failing, as all this does
                      # is prompt kubernetes to schedule new replica set for
                      # the deployment
      activeDeadlineSeconds: 600 # timeout, makes most sense with 
                                 # "waiting for rollout" variant specified below
      template:
        spec:
          serviceAccountName: deployment-restart # name of the service
                                                 # account configured above
          restartPolicy: Never
          containers:
            - name: restart-pod
              image: bitnami/kubectl # probably any kubectl image will do,
                                     # optionaly specify version, but this
                                     # should not be necessary, as long the
                                     # version of kubectl is new enough to
                                     # have `rollout restart`
          command:
            - /bin/sh
            - -c
            - |
              kubectl rollout restart deployment/<YOUR DEPLOYMENT NAME>
```
