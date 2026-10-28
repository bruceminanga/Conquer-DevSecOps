# 🛑 Stop Using Deployments for Everything! Mastering Kubernetes Jobs & CronJobs



When we think of Kubernetes, we usually picture highly available, always-on web services and APIs. If a Pod crashes, Kubernetes automatically spins it back up in an infinite loop of self-healing.

**But what happens when you have a task that is designed to finish?**

Think about database schema migrations, nightly backup exports, batch report generation, or machine learning model training. If you run these inside a standard Deployment, Kubernetes treats container termination as a failure and restarts the task infinitely.

Enter the dedicated batch processing primitives: **Jobs** and **CronJobs**. 🦸

---

## 💡 Why We Need Kubernetes Jobs

A **Job** is purpose-built for finite, run-to-completion workloads. It coordinates Pods to ensure a specified number terminate successfully with an exit code of `0`.

* **Guaranteed Run-to-Completion:** Recognizes that "Done" means finished—it will not infinitely reboot a successfully exited container.
* **Built-in Retry Limits:** If a worker node crashes mid-execution, the Job controller automatically reschedules and retries up to your configured `backoffLimit`.
* **Parallel Processing (`parallelism` & `completions`):** Need to process 10,000 files? Run 10 parallel workers concurrently. With `completionMode: Indexed`, each Pod receives a unique index (`JOB_COMPLETION_INDEX`), making it easy to partition static work queues.
* **Instant Resource Deallocation:** The moment a Job completes, its compute footprint (CPU and RAM) is returned to the cluster pool.

---

## ⏰ Why We Need Kubernetes CronJobs

A **CronJob** manages the lifecycle of a Job on a recurring time-based schedule.

* **Eliminates Dedicated Cron VMs:** Removes single-point-of-failure Linux cron servers by distributing scheduled executions across cluster worker nodes.
* **Concurrency Policies:** Prevent race conditions. If yesterday’s backup is still running, you can instruct Kubernetes to `Forbid` overlapping runs or terminate the older run.
* **Execution History & Auditing:** Retains logs and metadata for a defined window of successful and failed past runs.

---

## 🛠️ The Hands-On Runbook

### 1️⃣ Running a One-Time Batch Job

We will execute a finite mathematical task: calculating Pi to 2,000 decimal places. It runs, writes output, and stops cleanly.

Create `job-pi.yaml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: calculate-pi
spec:
  backoffLimit: 4             # Retry up to 4 times on failure before failing the Job
  activeDeadlineSeconds: 120  # Hard timeout: kills Job if it runs longer than 2 minutes
  ttlSecondsAfterFinished: 300 # Automatic cleanup: deletes Job 5 minutes after completion
  template:
    spec:
      containers:
      - name: pi-calculator
        image: perl:5.34.0
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never   # For Jobs: Must be Never or OnFailure (Never Always!)
```

#### Apply and Inspect:

```bash
# 1. Submit the Job
kubectl apply -f job-pi.yaml

# 2. Watch execution to completion
kubectl get pods -l job-name=calculate-pi -w

# 3. Read the output from the completed pod
kubectl logs -l job-name=calculate-pi
```

> ⚠️ **Two Critical Job Guardrails:**
> * **`activeDeadlineSeconds`:** Prevents hanging jobs. Without this, a deadlocked script or hanging database query can run indefinitely, consuming cluster resources silently.
> * **`ttlSecondsAfterFinished`:** Completed jobs do not delete themselves by default. Without a TTL, thousands of stale Pods and Jobs accumulate in `Completed` state and clutter `etcd`.

---

### 2️⃣ Running a Scheduled CronJob

Create `cronjob-hello.yaml` to run a task on a repeating schedule:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-every-minute
spec:
  schedule: "* * * * *"          # Standard cron syntax (runs every minute)
  timeZone: "America/New_York"   # Supported in K8s 1.27+ (avoids manual UTC conversion)
  concurrencyPolicy: Forbid      # Options: Allow, Forbid, Replace
  startingDeadlineSeconds: 60    # Bounds schedule lookback if controller was offline
  successfulJobsHistoryLimit: 3  # Keep audit trail for last 3 successful runs
  failedJobsHistoryLimit: 1      # Keep logs for the last failed run
  suspend: false                 # Set to true to freeze runs without deleting the manifest
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox:1.36
            command: ["/bin/sh", "-c", "date; echo 'Batch task completed successfully!'"]
          restartPolicy: OnFailure
```

Apply and monitor:

```bash
kubectl apply -f cronjob-hello.yaml
kubectl get cronjob hello-every-minute
```

> ⚠️ **The 100-Missed-Schedules Gotcha:**  
> If the `kube-controller-manager` goes down or a CronJob schedule is misconfigured such that more than 100 scheduled executions are missed, Kubernetes **stops scheduling that CronJob entirely** without a loud error. Setting `startingDeadlineSeconds: 60` fixes this by restricting the historical window Kubernetes checks when recovering.

---

## 🔥 Operational Pro-Tips from the Trenches

### 1. Trigger an Immediate Ad-Hoc Run
If you need to run a scheduled CronJob out-of-band for testing, do not alter the schedule. Spawn an ad-hoc Job directly from the CronJob template:

```bash
kubectl create job --from=cronjob/hello-every-minute manual-debug-run-01
```

### 2. Pause a CronJob During Maintenance
To temporarily pause scheduled runs during a maintenance window without deleting the object:

```bash
# Freeze execution
kubectl patch cronjob hello-every-minute -p '{"spec":{"suspend":true}}'

# Resume execution
kubectl patch cronjob hello-every-minute -p '{"spec":{"suspend":false}}'
```

---

## 📋 Concurrency Policy Decision Matrix

| Policy                  | Behavior                                                     | Best Use Case                                                           |
| :---------------------- | :----------------------------------------------------------- | :---------------------------------------------------------------------- |
| **`Allow`** *(Default)* | Concurrent runs execute simultaneously.                      | Stateless, independent batch queue processors.                          |
| **`Forbid`**            | Skips the new run if the previous run is still active.       | Database backups, heavy index rebuilds, shared resource tasks.          |
| **`Replace`**           | Terminates the running Job and starts a new one immediately. | Tasks where only the most recent data matters (e.g., metric snapshots). |

