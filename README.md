# RHACS demo

This repository provides a ready-to-use setup for a Red Hat Advanced Cluster Security (RHACS) demo. 

## Objective

Demonstrate how RHACS enforces security across the container lifecycle by deploying **intentionally non-compliant workloads** and observing detection, enforcement, and audit behavior at:


- Build / admission time
- Runtime
- Vulnerability management
- Compliance and governance


## Prerequisites

- OpenShift or Kubernetes cluster
- Red Hat Advanced Cluster Security installed
- Central and Secured Cluster connected
- `oc` or `kubectl` configured

## Scope of the Demo

### Build / Admission Controls
- Prohibit privileged containers
- Block running as root whenever applicable
- Allowed and denied Linux capabilities
- HostPath, HostNetwork, and HostPID restrictions with approved exceptions
- Image registry allowlisting (approved vs denied registries)
- CVE acceptance criteria (severity thresholds)
- Mandatory CPU and memory requests / limits
- Benchmarks referenced (CIS Kubernetes)

### Runtime Security
- Forbidden commands and processes:
  - Interactive shells
  - netcat
  - curl / wget (scoped by namespace or workload)
- Policy scope:
  - Global
  - Per namespace
  - Per workload
- ACS response actions:
  - Alert only
  - Kill pod
  - Block exec
  - Quarantine workload
  
### Audit and Integrations
- Violations recorded in ACS audit logs
- Policy categories and tags for classification
- Events prepared for routing to external systems (SIEM, webhook, etc.)

## Repository Structure

```
rhacs-demo
├─ README.md
├─ docs/
│  ├─ 00-prereqs.md
│  ├─ 10-demo-runbook-2h.md
│  ├─ 20-reset-cleanup.md
│  └─ 30-audit-and-integrations.md
├─ namespaces/
│  ├─ demo-ns.yaml
│  └─ restricted-ns.yaml
├─ workloads/
│  ├─ privileged/
│  ├─ run-as-root/
│  ├─ capabilities/
│  ├─ hostpath/
│  ├─ registries/
│  ├─ resources/
│  └─ runtime/
├─ policies/
│  ├─ build/
│  └─ runtime/
└─ scripts/
```

---

## Quick Start

Create the demo namespace:

```bash
$ oc new-project acsdemo
```

Deploy a single non-compliant workload:

```bash
oc apply -f workloads/privileged/privileged-pod.yaml
```

Observe in ACS:
- Admission decision (allowed or blocked)
- Violation generated
- Enforcement action, if enabled

---

## Demo Flow (2 Hours)

1. **Architecture Overview**  
   Central, Sensor, Collector, and policy lifecycle

2. **Build and Admission Controls**  
   Privileged containers, root user, Linux capabilities, host access

3. **Supply Chain Security**  
   Registry allowlisting and image trust

4. **Vulnerability Management**  
   CVE severity thresholds and acceptance criteria

5. **Resource Governance**  
   Enforcing CPU and memory requests and limits

6. **Runtime Threat Detection**  
   Forbidden commands, process execution, response actions

7. **Audit and Integrations**  
   Violation records, categorization, and external routing

Detailed steps and timing are documented in:

```
docs/10-demo-runbook-2h.md
```

---

## Policy Management

Policies can be:
- Created directly in the ACS UI
- Imported and exported using `roxctl`

Policies are organized under:
- `policies/build/`
- `policies/runtime/`

Secrets and API tokens must not be committed to this repository.

---

## Cleanup

Remove all demo resources:

```bash
./scripts/90-cleanup.sh
```

---

## Important Notes

- All workloads in this repository are **intentionally insecure**
- Use **demo or lab clusters only**
- Policies may be switched between **alert-only** and **enforce** mode during the demo
- Runtime enforcement actions should be demonstrated gradually

---
