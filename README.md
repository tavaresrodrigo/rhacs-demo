# RHACS demo

This repository provides a ready-to-use setup for a **Red Hat Advanced Cluster Security (RHACS)** demo. Before diving into the hands-on exercises, this section introduces the **RHACS architecture** to clarify how the platform observes, analyzes, and enforces security controls across the Kubernetes lifecycle.

## Red Hat Advanced Cluster Security architecture overview

Red Hat Advanced Cluster Security is built around a **centralized control plane** that continuously collects and analyzes security data from one or more K8s/OpenShift clusters. The core component, **Central**, provides the user interface, API, policy engine, and data store. All configuration, policy definition, alerting, and audit workflows are managed centrally, allowing security teams to define controls once and apply them consistently across environments.

The **Sensor** communicates with the Kubernetes API server and forwards metadata about deployments, pods, namespaces, and runtime activity to Central. The **Collector**, running on each node, observes low-level process and network activity to enable runtime threat detection. For vulnerability management, RHACS uses an integrated **Scanner** to analyze container images and correlate known vulnerabilities with deployed workloads.

### Key architecture components

- **Central**
  - Policy management, UI, API, and data storage
  - Aggregates and correlates security data from all clusters
- **Sensor**
  - Watches Kubernetes resources and lifecycle events
  - Sends workload and configuration data to Central
- **Collector**
  - Monitors runtime process and network activity on nodes
  - Enables runtime threat detection and enforcement
- **Scanner**
  - Scans container images for vulnerabilities
  - Supports CVE severity thresholds and risk acceptance

![RHACS architecture](/images/acs-architecture-scannerv4.png)

---

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
├─ workloads/
│  ├─ privileged.yaml
│  ├─ forbidden-cmd.yaml
│  ├─ run-as-root/
│  ├─ capabilities/
│  ├─ hostpath/
│  ├─ registries/
│  ├─ resources/
│  └─ runtime/
└─ images/
```

---

## Quick Start

Create the demo namespace which will be used in this demo:

```bash
$ oc new-project acsdemo
```

## Blocking the privileged container creation

Setting `securityContext.privileged: true` gives a container almost the same level of access as a process running directly on the host, effectively disabling most container isolation mechanisms.

A privileged container can access host devices, modify kernel settings, load kernel modules, and interact with the host filesystem and network stack, which means a compromise inside the container can quickly become a host compromise.

```bash
[...]
securityContext:
  privileged: true
[...]
```

Deploy a single non-compliant workload which contains a privileged container:

```bash
oc apply -f workloads/privileged/yaml
```

Observe in ACS:

- Admission decision (allowed or blocked)
- Violation generated
- Enforcement action, if enabled

### Enforcing the policy

By default, out-of-the-box policies are configured with the **Inform** method. Change the policy action to **Inform and enforce** at the **Deployment** stage.

Go to `<Platform Configuration> <Policy Management> <Policy Behavior> <Actions>` to enable enforcement in the Privileged Container Policy.

![Enforcing policy](/images/privileged.png)

After changing the mitigation method, recreate the workload:

```bash
oc delete -f workloads/privileged/yaml
oc apply -f workloads/privileged/yaml
```

Observe the command output and describe what happened.

---

## Blocking forbidden commands at runtime (netcat)

Allowing containers to execute tools such as `netcat` introduces a significant runtime risk, as these utilities can be used to open listening ports, establish reverse shells, or exfiltrate data. When combined with an interactive shell, tools like `nc` enable attackers to maintain persistence and pivot to other services or nodes.

In production environments, the presence of network and exploitation utilities inside application containers is rarely justified and often indicates malicious activity or a policy violation.

```bash
[...]
command:
  - /bin/sh
  - -c
  - nc -lvnp 4444
[...]
```

Deploy a non-compliant workload that executes a forbidden command (`netcat`) at runtime:

```bash
oc apply -f workloads/forbidden-cmd.yaml
```

Observe in ACS:

- Runtime violation generated
- Events in the `acsdemo` namespace
- Process and command details captured
- Response action applied, if enabled (alert, kill pod, block exec, quarantine)

### Enforcing the policy

Change the policy action to **Inform and enforce** at the **Runtime** stage.

Go to `<Platform Configuration> <Policy Management> <Policy Behavior> <Actions>` and enable enforcement for the policy **Netcat Execution Detected**.

Recreate the workload and check the events in the namespace:

```bash
$ oc get events | grep -i stackrox
10m   Warning   StackRox enforcement   pod/netcat-test   A pod (netcat-test) violated StackRox policy "Netcat Execution Detected" and was killed
```

Go to `<Violations> <Active>`, search for **Netcat Execution Detected**, click the policy name, and review the violation details:

![netcat test](/images/netcat.png)

---

## Blocking containers running as root

Running containers as the root user (`UID 0`) weakens the security boundary between the application and the container runtime. If a vulnerability is exploited in a root container, the attacker gains elevated privileges inside the container, which increases the likelihood of container escape, host compromise, or abuse of mounted volumes and credentials.

In most application workloads, running as root is unnecessary. Enforcing non-root execution reduces the blast radius of an attack and aligns with Kubernetes and OpenShift security best practices.

```bash
[...]
securityContext:
  runAsUser: 0
  allowPrivilegeEscalation: true
[...]
```

Deploy a non-compliant workload that runs as the root user:

```bash
oc apply -f workloads/run-as-root.yaml
```

Observe in ACS:

- Admission decision (allowed or blocked)
- Violation generated for running as root
- Enforcement action, if enabled

### Enforcing the policy

Change the policy action to **Inform and enforce** at the **Deployment** stage.

Go to `<Platform Configuration> <Policy Management> <Policy Behavior> <Actions>` and enable enforcement for the policy **Container Runs as Root User** (or equivalent).

Recreate the workload after enabling enforcement:

```bash
oc delete -f workloads/run-as-root.yaml
oc apply -f workloads/run-as-root.yaml
```

Observe the command output and describe what happened.

---

## Visualizing network communication with NetworkPolicy graph

This exercise illustrates the Network Graph view in ACS. By deploying two workloads in the same namespace and selectively allowing traffic between them, you can observe how ACS builds a clear picture of allowed and denied network paths.

NetworkPolicies are a key control for reducing lateral movement. RHACS does not enforce NetworkPolicies itself, but it **analyzes, visualizes, and correlates** them with runtime activity, helping teams understand actual communication patterns versus intended policy.

---

### Scenario description

- Two workloads are deployed in the `acsdemo` namespace:
  - `frontend` (client)
  - `backend` (server)
- A NetworkPolicy allows traffic **only from frontend to backend** on a specific port
- All other ingress traffic to the backend is denied by default

This setup allows ACS to show:
- Allowed communication paths
- Denied or absent paths
- Policy-driven segmentation in the network graph

**Deploy the backend and frontend workload**

```bash
$ oc apply -f workloads/np-backend.yaml
$ oc apply -f workloads/np-frontend.yaml
```

**Deploy the networkPolicies**

```bash
$ oc apply -f workloads/np.yaml
```