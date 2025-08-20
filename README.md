# Interview_questions_scenario

✅ **Quick Headline (to remember)**
✅ **Bullet points (what to say in interview)**
✅ **Mini real-time story (to show impact)**

---

# **Terraform & IaC**

**1. Handling Terraform drift in production**
👉 **Headline:** Detect → Review → Correct → Apply

* Use `terraform plan -refresh-only` to detect drift.
* Example: EC2 tags manually changed by ops team.
* Never blindly apply → always review plan.
* If conflict → decide if Terraform state should override or accept infra change.
* Best practice → use CI/CD pipeline with `terraform plan` PR checks + remote backend (S3 + DynamoDB).

🟢 **Story:** “We once found that ops had manually resized an RDS instance in prod. `terraform plan` flagged drift. Instead of blindly applying, I reviewed with DB team and updated Terraform code to match. This avoided accidental DB downtime.”

---

**2. If EC2 creation failed mid-apply**
👉 **Headline:** Target + Re-apply

* Use `terraform apply -target=aws_instance.my_ec2`.
* Or `terraform taint` if resource is corrupted → forces re-create.
* Always store state remotely (S3/Dynamo) to avoid mismatch.

🟢 **Story:** “In prod, one EC2 failed due to subnet mismatch. Instead of re-applying whole infra, I targeted just that resource, fixed subnet\_id, and applied cleanly.”

---

**3. Structuring Terraform modules for multi-AZ**
👉 **Headline:** DRY + Parametrize + For\_each

* Use a VPC module (networking).
* Pass list of subnets per AZ.
* Use `for_each` to create EC2 per AZ.
* Keep separate `prod`, `stage`, `dev` workspaces.

🟢 **Story:** “We used modules for VPC, EKS, and RDS. For EC2s, we looped over AZs to ensure HA without writing duplicate code.”

---

**4. Secrets in Terraform**
👉 **Headline:** Never hardcode → Use Vault/SSM

* Use AWS SSM Parameter Store / Secrets Manager.
* Or HashiCorp Vault integration.
* Mask secrets in `terraform.tfvars`.

🟢 **Story:** “We had DB passwords in plain tfvars — security flagged it. Moved to AWS Secrets Manager and pulled dynamically during apply. Passed audit cleanly.”

---

**5. Multiple teams same Terraform state**
👉 **Headline:** Remote state + Locking

* Store state in S3 backend with DynamoDB locking.
* Use separate workspaces/envs.
* Use Terragrunt for team collaboration.

🟢 **Story:** “Two teams deploying same infra caused race conditions. Added DynamoDB state lock — fixed overlapping apply issues.”

---

# **AWS & Networking**

**6. EC2 in private subnet → push to S3**
👉 **Headline:** Role + Gateway + Policy

* EC2 → IAM Role with S3 write policy.
* Route → NAT Gateway (if internet needed) OR VPC Endpoint (best).
* S3 Bucket policy restricts only that role.

🟢 **Story:** “We set up a VPC Gateway Endpoint for S3. That cut NAT traffic cost by \~40% and removed public exposure.”

---

**7. Public server in private subnet (mistake)**
👉 **Headline:** 2 Fixes → Move or Re-map

* If possible → stop instance → move ENI to public subnet.
* Else → attach Elastic IP + NAT/ALB.
* Don’t hack security groups; fix subnet design.

🟢 **Story:** “Someone deployed a web server in private subnet. Instead of rebuilding, we migrated ENI to public subnet and re-attached Elastic IP — app came back online in minutes.”

---

**8. EKS pods access S3 without static creds**
👉 **Headline:** IRSA (IAM Role for Service Accounts)

* Create IAM role with S3 policy.
* Map to K8s ServiceAccount.
* Pods use role automatically.

🟢 **Story:** “We replaced AWS creds in pods with IRSA. This eliminated secret rotation headaches and passed security review.”

---

**9. ALB vs NLB**
👉 **Headline:** ALB = HTTP brain, NLB = TCP muscle

* ALB = Layer 7 → routing by hostname/path.
* NLB = Layer 4 → raw performance, static IPs.
* NLB supports TCP/UDP; ALB doesn’t.

🟢 **Story:** “We used ALB for microservices with routing. For gaming TCP traffic, we used NLB. Wrong choice here can kill performance.”

---

**10. HA in AWS using AZs**
👉 **Headline:** Spread + Autoheal + DR

* Spread EC2/EKS nodes across ≥2 AZs.
* Use Auto Scaling groups.
* RDS Multi-AZ / Aurora.
* S3 cross-region replication for DR.

🟢 **Story:** “We designed 99.99% uptime infra by spreading EKS + RDS across 3 AZs. During AZ outage, traffic shifted automatically.”

---

# **Kubernetes (EKS)**

**11. Compute engine behind EKS**
👉 **Headline:** Worker = EC2 (or Fargate)

* Control plane = AWS managed (not visible).
* Worker nodes = EC2 ASG OR Fargate.
* Networking = CNI plugin (ENIs in VPC).

---

**12. Challenges in K8s v1.33 upgrade**
👉 **Headline:** API Deprecation + PreferClose

* Watch for deprecated APIs (Ingress v1beta1 gone long ago).
* PreferClose scheduling can affect pod placement (closer nodes chosen → imbalance).
* Need to test workloads with new scheduler.

---

**13. Upgrades & PodDisruptionBudget**
👉 **Headline:** Drain smartly

* `kubectl drain` respects PDB.
* If PDB too strict (minAvailable=100%), no upgrade possible.
* Adjust PDB temporarily.

🟢 **Story:** “During node upgrade, PDB blocked drain. We temporarily reduced PDB from 100% → 80%, upgraded nodes, then restored. Zero downtime.”

---

**14. StatefulSet vs DaemonSet vs Deployment**

* Deployment → stateless apps (e.g., web app).
* StatefulSet → DB, Kafka, apps needing persistent ID/storage.
* DaemonSet → 1 pod per node (e.g., logging agent, monitoring agent).

🟢 **Story:** “We ran Fluentd as DaemonSet, Kafka as StatefulSet, and microservices as Deployments.”

---

**15. Pods running, service unreachable**
👉 **Checklist:**

* `kubectl describe svc` → check selectors.
* `kubectl get ep` → endpoints exist?
* SG/NACL allow traffic?
* DNS resolving?

🟢 **Story:** “App pods were running but service label selector was wrong (`app=frontend` vs `app=fe`). Corrected selector → traffic started flowing.”

---

# **CI/CD & Automation**

**16. Multi-env pipeline in Jenkins**
👉 **Headline:** Params + Branch/Env strategy

* Use Jenkins pipeline parameters (`ENV=dev/prod`).
* Deploy to namespace/env based on param.
* Store values in Helm values.yaml per env.

---

**17. Helm rollback if deploy fails**
👉 **Headline:** Detect + Rollback

* Use `helm upgrade --atomic`.
* If fail → auto rollback to last good version.
* Jenkins monitors Helm exit code.

🟢 **Story:** “One prod deploy failed due to bad config. Helm atomic rollback restored last good state in <1 min.”

---

**18. Automated task with Ansible/Python**
👉 **Headline:** Replace repetitive → Script it

* Ansible: Automated weekend EC2 stop/start by tags.
* Python: Automated EKS node health check + alert to Slack.

🟢 **Story:** “We saved 20+ hours/month by automating EC2 shutdown with Ansible. Finance team loved the cost savings.”

---

# **Monitoring & Observability**

**19. Use case: Fluentd, Prometheus, Grafana**

* Fluentd → ship logs to S3/Elasticsearch.
* Prometheus → collect pod metrics (CPU/mem).
* Grafana → visualize + alert on dashboards.

---

**20. Grafana Data Source**
👉 **Headline:** Input for Dashboards

* Examples: Prometheus, CloudWatch, Loki, Elasticsearch, MySQL.
* I’ve used 3–4 in production (Prometheus, CloudWatch, Loki, Elastic).

---

**21. CloudWatch vs Prometheus/Grafana**
👉 **Headline:** Native vs Open-source power

* CloudWatch → native AWS metrics/logs, easy, but \$\$\$.
* Prometheus/Grafana → fine-grained metrics, flexible queries, but needs infra.

🟢 **Story:** “We used CloudWatch for AWS infra monitoring, but for app-level metrics we needed Prometheus + Grafana because queries were more powerful.”

---

✅ That’s your **“cheat sheet”**:

* Headlines → memory hooks
* Bullets → interview structure
* Stories → show real experience

---

👉 Great question 👍 Let’s go step by step and make this **interview-ready** with both **process explanation** and a **real-time story** you can tell.

---

## 🌍 Terraform Drift Handling in Production

### Headline: **Detect → Review → Decide → Correct → Apply**

---

### 🔍 Step 1: Detect drift

* Command:

  ```bash
  terraform plan -refresh-only
  ```
* What happens:

  * Terraform refreshes the **state file** with the **real infrastructure** from the cloud provider (e.g., AWS).
  * It does **not plan to change anything** in infra; it just shows differences between **Terraform state** and **actual resources**.
  * Example: An EC2 instance type changed from `t3.medium` → `t3.large` manually.

---

### 📝 Step 2: Review drift

* Carefully look at what has drifted:

  * **Minor metadata** (tags, SG rules) → maybe accidental/manual ops change.
  * **Critical resources** (RDS resize, ASG changes) → could be intentional by another team.

---

### ⚖️ Step 3: Decide

Now you have **two choices**:

#### ✅ If you want to **keep the manual change**:

1. **Update Terraform code** to reflect the new reality.
   Example: Change `instance_type = "t3.large"` in Terraform config.
2. Run `terraform plan` again → drift is gone.
3. Apply safely with CI/CD pipeline.

👉 This ensures **Terraform state = real infra = code**.

---

#### ❌ If you do **NOT want to keep the manual change**:

1. Keep Terraform code as-is.
2. Run normal `terraform plan` → Terraform will propose to revert infra back to the original config.
3. Review → Apply → Infra reverts to match Terraform code.

👉 This enforces **Infra should always follow code**.

---

### 🚀 Step 4: Apply via pipeline

* Never run `terraform apply` directly in prod.
* Use CI/CD pipeline with:

  * Remote backend (S3 + DynamoDB lock).
  * PR checks showing `terraform plan` output.
  * Peer review before merge.

---

## 🔥 Real-World Story (Interview Style Answer)

*"In one case, our ops team manually resized an RDS instance in production during a performance issue. A week later, when I ran `terraform plan -refresh-only`, it flagged a drift. Instead of blindly applying, I discussed with the DB team and confirmed the change was intentional. I updated the Terraform code to reflect the new size, so our IaC stayed in sync with reality. This avoided accidental downtime if Terraform had tried to revert it. Since then, we enforced a policy: any infra change must go through Terraform PRs, with drift detection as part of our CI/CD checks."*

---

✅ So your **structured interview answer** would be:

1. Detect drift with `terraform plan -refresh-only`.
2. Review the plan output.
3. If we want to **keep the manual change** → update Terraform code.
4. If we don’t want it → apply Terraform to override manual change.
5. Always go through PR review + remote backend for safe apply.

---


