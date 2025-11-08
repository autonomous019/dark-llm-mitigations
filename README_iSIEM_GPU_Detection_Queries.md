# ⚙️ GPU Abuse & Covert AI Training Detection — iSIEM Query Guide

**Audience:** SOC analysts, detection engineers, and incident responders.  
**Purpose:** Provide contextual SIEM queries and analytic logic for detecting illicit GPU usage, unauthorized LLM training, or compute abuse in cloud and hybrid environments.  
**Scope:** Defensive only — all queries focus on detection and alerting, not offensive or exploitative actions.

---

## 🧩 1. Detection Philosophy

GPU misuse for illicit AI workloads leaves *correlated traces* across:
- Cloud audit logs (instance creation, quota changes)
- Host telemetry (processes, GPU metrics)
- Network flows (large data transfers, unknown endpoints)
- Storage access logs (large uploads, new buckets)
- Billing/usage anomalies (spending spikes)

The following queries are intended for:
- **Proactive hunting**  
- **Real-time alerting**  
- **Anomaly detection baselining**

---

## ☁️ 2. Cloud Audit Log Queries

### Splunk — AWS CloudTrail / Azure Activity / GCP Audit Logs

```spl
index=cloud_audit_logs
(eventName="RunInstances" OR eventName="CreateInstance" OR eventName="Compute.Instances.insert")
| eval instanceType=coalesce(requestParameters.instanceType, json_extract(request, "$.protoPayload.request.machineType"))
| search instanceType IN ("p4d*", "p5*", "a100*", "h100*", "mi300*", "v100*")
| stats count BY userIdentity.arn, instanceType, sourceIPAddress, region
| where count > 0
```

**Context:**  
Flags new GPU instance provisioning. Filter by instance family names (A100, H100, MI300, V100, etc.).  
Use for baseline deviation — any creation event from a non-ML user identity is suspicious.

---

### Microsoft Sentinel — Cloud Activity
```kql
AuditLogs
| where OperationName has_any ("Create virtual machine", "Start virtual machine")
| where AdditionalDetails has_any ("GPU", "NVidia", "A100", "H100")
| summarize count() by Identity, IPAddress, Location
```
**Context:**  
Detects VM creation with GPU metadata.  
Ideal for multi-cloud or M365-linked tenants using Azure.

---

### Google Chronicle / GCP
```kql
metadata.event_type="GCE_VM_CREATE"
json_extract_scalar(metadata, "$.instance.machineType") =~ "(a100|h100|v100)"
| aggregate count() by principalEmail, zone, instanceName
```
**Context:**  
Detects creation of GPU-enabled Compute Engine instances across GCP projects.

---

## 💻 3. Host Process & GPU Utilization Queries

### Elastic / OSQuery Logs
```sql
SELECT
  hostname, process_name, cmdline, gpu_utilization, gpu_memory_used
FROM
  processes
WHERE
  process_name LIKE '%python%'
  AND cmdline LIKE '%torch%' OR cmdline LIKE '%tensorflow%' OR cmdline LIKE '%transformers%'
  AND gpu_utilization > 70
  AND duration_seconds > 600;
```

**Context:**  
Flags long-running Python processes with GPU load >70% for >10 min.  
Excellent for identifying unauthorized LLM training, mining, or inference jobs.

---

### Splunk (Sysmon + NVIDIA Telemetry)
```spl
index=sysmon sourcetype=nvidia_metrics
| stats avg(gpu_utilization) AS avgUtil, values(process_name) AS processes BY host
| where avgUtil > 80 AND NOT processes IN ("approved_ml_service", "model_training_daemon")
```
**Context:**  
Detects unapproved GPU-intensive workloads by average utilization across Sysmon/NVIDIA plugin logs.

---

## 🌐 4. Network & Data Exfiltration Queries

### Splunk — Large Uploads from GPU Hosts
```spl
index=network_traffic dest_port=443
| lookup asset_inventory host OUTPUT asset_type
| where asset_type="gpu" OR host LIKE "%gpu%"
| stats sum(bytes_out) AS outbound BY host, dest_ip
| where outbound > 5000000000  /* 5GB threshold */
```
**Context:**  
Flags GPU assets uploading >5GB in a 24-hour window — possible model checkpoint exfiltration.

---

### Elastic — Unexpected Data Egress
```kql
event.dataset: "network.flow"
AND network.direction: "outbound"
AND destination.domain NOT in ["approved-domain1.com", "approved-domain2.com"]
AND process.command_line: ("torch" OR "tensorflow" OR "transformers")
```
**Context:**  
Detects ML processes making outbound connections to non-approved domains.  
Correlate with GPU metrics for higher confidence.

---

## 📦 5. Storage & Model Artifact Detection

### Splunk — Cloud Storage Write Volume
```spl
index=cloud_storage_logs eventType=PutObject
| stats sum(bytesTransferred) AS upload_size BY userIdentity.arn, bucketName
| where upload_size > 1000000000
| search bucketName NOT IN ("approved-training-data", "backup-bucket")
```
**Context:**  
Detects large uploads to storage (e.g., S3) from non-standard accounts.  
Useful for identifying model checkpoints or data dumps.

---

### OSQuery — File System Hunt
```sql
SELECT path, size, atime, mtime
FROM file
WHERE path LIKE '%.pt' OR path LIKE '%.bin' OR path LIKE '%.ckpt'
AND size > 100000000;
```
**Context:**  
Searches local file systems for model files >100MB — common indicator of training runs.

---

## 💰 6. Billing & Cost Anomaly Queries

### Splunk — Cost Spike Detection
```spl
index=cloud_billing
| stats sum(cost) AS total_cost BY account, service
| where service IN ("EC2", "Compute Engine", "Azure VM")
| delta total_cost AS change
| where change > (total_cost * 2)
```
**Context:**  
Detects sudden doubling (or more) in compute cost — GPU campaigns often cause spikes in cloud bills.

---

## 🧠 7. Multi-Signal Correlation (Composite Detections)

Combine multiple weak signals into one **high-confidence detection**.

### Example (Splunk):
```spl
index=cloud_audit_logs OR index=sysmon OR index=network_traffic
| transaction host maxspan=1h
  startswith=(eventName="RunInstances" OR process_name="python")
  endswith=(dest_bytes>5000000000 OR filename="*.pt")
| stats count BY host, userIdentity.arn
```
**Context:**  
Correlates instance creation → GPU training → large upload within 1 hour.  
Strong indicator of unauthorized LLM activity.

---

## 🔍 8. Anomaly-Based Detection Ideas

- **GPU-hours anomaly model:**  
  Train a baseline of `gpu_hours` per user/project → alert on 3σ deviations.  
- **Cluster-level GPU contention:**  
  Detect sudden drops in available GPU quota (from Kubernetes metrics).  
- **Process lineage anomaly:**  
  Flag new Python processes spawned by non-interactive shells or service accounts.  
- **Dataflow anomaly:**  
  Alert on first-seen destination domain from GPU cluster subnet.  

---

## 🧾 9. Output & Escalation

- Send confirmed hits to **Tier-3 or DFIR** for artifact collection.  
- Cross-reference with **billing data** for compute impact.  
- Generate **cloud support tickets** if confirmed abuse involves provider infrastructure.  
- For critical detections, notify **law enforcement or CERT** if criminal misuse suspected.

---

## 🧭 10. Summary of Query Goals

| Query Category | Primary Objective | Outcome |
|----------------|-------------------|----------|
| Cloud Audit | Detect GPU provisioning | Early compromise identification |
| Host Telemetry | Catch illicit GPU workloads | Containment and triage |
| Network | Spot large exfil events | Stop model/data theft |
| Storage | Identify model file uploads | Evidence for forensics |
| Billing | Highlight cost anomalies | Detect campaigns at scale |

---

**Defensive Use Only — © 2025, autonomous019 & ChatGPT (OpenAI, model GPT-5)**  
_Last updated: November 7, 2025_
