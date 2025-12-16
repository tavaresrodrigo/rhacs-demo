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
├─ workloads/
│  ├─ privileged.yaml
|  ├─ forbidden-cmd.yaml
│  ├─ run-as-root/
│  ├─ capabilities/
│  ├─ hostpath/
│  ├─ registries/
│  ├─ resources/
│  └─ runtime/
└─ scripts/
```

---

## Quick Start

Create the demo namespace which will be used in this demo:

```bash
$ oc new-project acsdemo
```

## Blocking the Privileged container creation

Setting securityContext.privileged: true gives a container almost the same level of access as a process running directly on the host, effectively disabling most container isolation mechanisms.

A privileged container can access host devices, modify kernel settings, load kernel modules, and interact with the host filesystem and network stack, which means a compromise inside the container can quickly become a host compromise.

```bash
[...]
    securityContext:
      privileged: true
[...]
```


Deploy a single non-compliant workload which contains a privileged container.

```bash
oc apply -f workloads/privileged/yaml
```

Observe in ACS:

- Admission decision (allowed or blocked)
- Violation generated
- Enforcement action, if enabled

### Enforcing the policy

By default the out of the box policies are configured with **Inform** method to address violations, here we are going to change it to **Inform and enforce** at the **Deployment** stage.

Go to <Platform Configuration><Policy Management><Policy Behavior><Actions> and enable the enforcement.

![Enforcing policy](/images/privileged.png)

After changing the mitigation method in the policy lets recreate our workload. 

```bash
oc delete -f workloads/privileged/yaml
oc apply -f workloads/privileged/yaml
```

Observe the command output and escribe what happened.

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
- Get the events in the acsdemo namespace
- Process and command details captured
- Response action applied, if enabled (alert, kill pod, block exec, quarantine)

### Enforcing the policy

Again lets change the policy action to **Inform and enforce** at the **Runtime** stage.

Go to `<Platform Configuration> <Policy Management> <Policy Behavior> <Actions>` and enable enforcement for the policy called **Netcat Execution Detected**

Recreate the workload and check for the events in the namespace.

Search for StackRox events in the acsdemo namespace:

```bash
$ oc get events | grep -i stackrox
10m         Warning   StackRox enforcement   pod/netcat-test      A pod (netcat-test) violated StackRox policy "Netcat Execution Detected" and was killed
2m53s       Warning   StackRox enforcement   pod/netcat-test      A pod (netcat-test) violated StackRox policy "Netcat Execution Detected" and was killed
```

Go to <Violations><Active>, search for the Netcat Execution Detected policy and observe the number of enforced, click the policy name and confirm the details about the violation event:

![netcat test](/images/netcat.png)
