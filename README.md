# SOC Network Architecture | Traffic Flow & Trust Boundaries

**Conceptual architecture exercise** · **Diagramming / security design** · **@sharpleynate**

> **Scope disclosure:** This repository documents a proposed SOC architecture. I created and annotated the diagrams to explain traffic flows, trust boundaries, monitoring components, and potential failure points. I did **not** deploy or configure the depicted Vultr instances, Elastic Stack, Fleet Server, osTicket, or adversary infrastructure as part of this project. The diagrams are design illustrations, not evidence of an operational SOC.

## Overview

This project explores how a small, hypothetical Security Operations Center (SOC) could be organized in a Vultr cloud environment. It focuses on the distinction between **endpoint event generation**, **Elastic Agent log collection**, **Fleet agent management**, **Elasticsearch storage and search**, **Kibana analyst investigation**, and **osTicket incident tracking**.

The proposed environment includes Windows and Ubuntu monitored hosts, a Fleet Server, Elasticsearch/Kibana, osTicket, an analyst workstation, and external simulated adversary components (Kali Linux and a hypothetical Mythic C2 server).

## Diagrams

### 1. Proposed SOC architecture
<img width="1495" height="856" alt="01-architecture" src="https://github.com/user-attachments/assets/94a1074d-72a9-4f52-95bd-f6c04e7ec829" />

### 2. Telemetry and management flow
![Telemetry and agent management](diagrams/02-telemetry-flow.png) ![Uploading 01-architecture.png…]()


### 3. Trust boundaries and access
![Trust boundaries](diagrams/03-trust-boundaries.png)

### 4. Failure modes and visibility
![Failure modes](diagrams/04-failure-visibility.png)

Editable Graphviz `.dot` source and scalable `.svg` files are included in `diagrams/`.

## Component responsibilities

| Component | Conceptual responsibility |
|---|---|
| Windows Server | Generates Windows security events, including failed RDP logons (often Event ID 4625). |
| Ubuntu Server | Generates Linux system and SSH authentication logs. |
| Elastic Agent | Runs on monitored hosts, collects configured telemetry, and forwards it to an output such as Elasticsearch. |
| Fleet Server | Coordinates enrollment, agent check-ins, and centrally managed policies; it is **not** normally the log-forwarding hop. |
| Elasticsearch | Receives, indexes, stores, and searches telemetry. |
| Kibana | Provides the web interface for searches, detections, investigations, and Fleet configuration. |
| osTicket | Represents a separate incident/case-tracking workflow; integration is **proposed**, not implemented. |
| Kali Linux | Represents an external attacker workstation for authorized simulations. |
| Mythic C2 | Represents hypothetical attacker command-and-control infrastructure, not a deployed C2 instance. |

## Example event path (conceptual)

1. A simulated failed RDP login occurs against Windows.
2. Windows records the authentication failure locally.
3. An Elastic Agent configured to collect the relevant Windows event logs reads the event.
4. The agent forwards the event to Elasticsearch.
5. An analyst searches or investigates the event in Kibana.
6. A case could be documented in osTicket manually, or through a **separately configured** integration.

**Log path:** `Windows → Elastic Agent → Elasticsearch ↔ Kibana → Analyst`

**Management path:** `Kibana (Fleet UI) → Fleet Server ↔ Elastic Agents`

These are different flows. The Ubuntu host is a separate monitored endpoint, not a required intermediary for Windows logs. Kibana queries Elasticsearch; it is not the primary log-storage destination.

## Trust boundaries and proposed safeguards

The diagram separates external systems, the cloud edge, and the private VPC. A VPC is a network boundary, **not proof of security enforcement**.

- Restrict RDP and SSH to approved management paths such as a VPN or tightly scoped allowlist; do not intentionally expose a domain controller's RDP to the open internet.
- Restrict access to Kibana and administrative services with authentication and appropriate network controls.
- Treat Kali/C2 communication as **hypothetical adversary traffic**; do not confuse it with trusted Elastic telemetry.
- Treat firewall rules, VPNs, network segmentation, TLS, and role-based access as **design recommendations**, not implemented controls.
- The reference image includes an example private subnet; this redesign intentionally avoids asserting that any particular CIDR was actually configured.

## Visibility and failure scenarios

| Scenario | Expected effect / limitation |
|---|---|
| Fleet Server unavailable | Existing agents can often continue under cached policies and forward telemetry if their output remains reachable; centralized policy changes are disrupted. |
| Kibana unavailable | Analyst web access and investigations are interrupted, while agent ingestion into healthy Elasticsearch may continue. |
| Elasticsearch unavailable | Central ingestion and searches fail; agents may retry or buffer, but full recovery depends on queue capacity, source retention, and collection settings. |
| Monitored host unavailable | That host stops generating new live telemetry; the rest of the SOC components need not fail. |
| Missing 30 minutes of telemetry | Investigate outage, agent/network problems, log rotation, and possible tampering. A gap alone is **not** proof of compromise. |

## Diagram legend

- **Green:** endpoint telemetry, storage, and search flow
- **Amber dashed:** agent-management/control plane
- **Blue:** analyst and administrative access
- **Red dashed:** hypothetical attacker path or exposure risk
- **Purple dashed:** proposed or manual incident-ticket workflow
- **Dashed VPC outline:** conceptual trust/network boundary, not an enforced firewall rule

## Learning outcomes

- Distinguished the data plane from the agent-management plane.
- Mapped monitored endpoints to log collection, indexing, and investigation interfaces.
- Identified internet-to-cloud and cloud-to-private-network trust boundaries.
- Illustrated why public RDP/SSH exposure increases risk.
- Examined how SIEM outages can create telemetry gaps and monitoring blind spots.
- Practiced documenting security architecture without misrepresenting conceptual work as hands-on deployment.
