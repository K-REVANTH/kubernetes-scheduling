# Kubernetes Scheduling Walkthrough (Scenario-Based)

This repository demonstrates Kubernetes scheduling concepts using real cluster experiments.

Cluster setup used:

- ip-172-31-15-1 → env=prod  
- ip-172-31-8-107 → env=dev  
- master → gpu=true, disk=ssd  

---

# SCENARIO 0: Setup (Mandatory)

```bash
kubectl create namespace scheduling-demo
kubectl get nodes
```

Label nodes:

```bash
kubectl label node ip-172-31-15-1 env=prod
kubectl label node ip-172-31-8-107 env=dev
kubectl label node master gpu=true
kubectl label node master disk=ssd
```

Verify:

```bash
kubectl get nodes --show-labels
```

---

# SCENARIO 1: NodeSelector Success

```yaml
nodeSelector:
  env: prod
```

Result:
- Pod scheduled on `ip-172-31-15-1`

---

# SCENARIO 2: NodeSelector Failure (Pending)

```yaml
nodeSelector:
  env: staging
```

Result:
- Pod → Pending

Debug:

```bash
kubectl describe pod <pod-name>
```

Reason:
- No node matches label

---

# SCENARIO 3: NodeSelector AND Condition

```yaml
nodeSelector:
  gpu: "true"
  disk: ssd
```

Result:
- Pod runs only on **master**

Definition:
- Multiple labels = AND

---

# SCENARIO 4: Taints Blocking Scheduling

Master has taint:

```
NoSchedule
```

Result:
- Pod cannot be scheduled on master

Fix:

```bash
kubectl taint nodes master node-role.kubernetes.io/control-plane:NoSchedule-
```

---

# SCENARIO 5: Node Affinity (Required Success)

```yaml
operator: In
values:
- prod
```

Result:
- Pod runs on prod node

---

# SCENARIO 6: Node Affinity Failure

```yaml
values:
- staging
```

Result:
- Pod → Pending

Reason:
- No matching node

---

# SCENARIO 7: Preferred Affinity (Soft Rule)

```yaml
preferredDuringSchedulingIgnoredDuringExecution
```

Result:
- Pod schedules even if condition not met

Definition:
- Soft constraint

---

# SCENARIO 8: Multiple Preferred (Scoring)

Weights:

- env=prod → 50  
- disk=ssd → 30  
- gpu=true → 20  

### Observation:

- prod node score = 50  
- master score = 50  

Result:
- Pod scheduled on **master**

### Key Insight:

- Tie → scheduler uses internal logic  
- Not deterministic  

---

# SCENARIO 9: Exists Operator

```yaml
operator: Exists
key: gpu
```

Result:
- Pod runs on master

---

# SCENARIO 10: NotIn Operator

```yaml
operator: NotIn
values:
- dev
```

Result:
- Avoids dev node

---

# SCENARIO 11: DoesNotExist Operator

```yaml
operator: DoesNotExist
key: test
```

Result:
- Pod runs on any node

---

# SCENARIO 12: Gt Operator

```yaml
operator: Gt
values:
- "2"
```

Label:

```bash
kubectl label node ip-172-31-15-1 cpu=4
```

Result:
- Pod runs on that node

---

# SCENARIO 13: Lt Operator

```yaml
operator: Lt
values:
- "10"
```

Result:
- Condition satisfied → pod runs

---

# SCENARIO 14: OR Logic

```yaml
nodeSelectorTerms:
- env=prod
- env=dev
```

Result:
- Pod runs on either prod OR dev

---

# SCENARIO 15: AND Logic

```yaml
gpu Exists AND disk=ssd
```

Result:
- Pod runs only on master

---

# SCENARIO 16: Anti-Affinity (Required - Strict)

```yaml
podAntiAffinity:
  requiredDuringSchedulingIgnoredDuringExecution
```

Rule:
- Do NOT place pods with same label on same node

Result:
- Pods spread across nodes

---

# SCENARIO 17: Anti-Affinity Spreading

Deploy multiple pods:

Result:
- Each pod goes to different node

---

# SCENARIO 18: Anti-Affinity Failure (Important)

Deploy more pods than nodes:

Result:
- Last pod → Pending

Reason:

```
didn't match pod anti-affinity rules
```

---

# SCENARIO 19: Anti-Affinity Preferred (Soft)

```yaml
preferredDuringSchedulingIgnoredDuringExecution
```

Result:
- Pods try to spread
- Never Pending

---
