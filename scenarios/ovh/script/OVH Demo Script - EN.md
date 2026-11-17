# OVHcloud Network Operations — End-to-End Demo Script

Elastic Observability + AI-assisted incident response on OVHcloud infrastructure

Primary fault (HITL)

CH-01: vRack VLAN Segmentation

Background fault (Auto)

CH-12: LB Health Check Flap

Error code

VRACK-VLAN-SEGMENTATION

Duration

\~12 minutes

Before starting: inject **CH-01 (vRack VLAN Segmentation)** in Cascading Failure mode from the Chaos Console. Also inject **CH-12 (LB Health Check Flap)** in Calibration Drift mode — this one auto-remediates in the background and will be revealed at the end.

---

## Act 1 — The Business Impact

Application Cockpit

In today's environment, every application has its own architecture — from legacy monoliths to cloud-native microservices, deployed on-premises, in the cloud, or across hybrid environments. Observing all of them through a single pane of glass is already a challenge; but correlating technical signals with business impact while evaluating incident budget burn is critical for any business. \
\
Elastic's unique ability to ingest any type of data into its Search AI database makes it the ideal foundation for a true 360-degree view. \
\
Looking at our application cockpit, which blends technical and business signals to show the end-to-end status of our OVHcloud network operations, we can see our application SLO is burning fast and directly impacting the business. The vRack cross-DC latency has spiked, backend health is dropping, and our uptime SLO is in breach. \
\
As an SRE, our job is to analyze the incident, execute the remediation, and deliver a root cause analysis — all while executives keep a real-time eye on the SLO.

Point out the KPI tiles: **vRack cross-DC latency** spiked from \~5ms to 100ms+, **Backend health %** dropped below 60%, **Uptime %** is in SLO breach. The trend charts show the moment the fault was injected.

---

## Act 2 — Finding the Needle

Discover

Navigate to **Discover**. In the ES|QL editor, type the query from **Tab 1** below.

Tab 1 — Data source

Tab 2 — ML Categorisation

```
FROM logs.otel.ovhnet, logs.otel.ovhnet.*, logs.ecs.ovhnet, logs.ecs.ovhnet.*
// Focus on ERROR log messages
```

```
FROM logs.otel.ovhnet, logs.otel.ovhnet.*, logs.ecs.ovhnet, logs.ecs.ovhnet.*
| WHERE log.level == "ERROR"
| STATS count=COUNT() BY category=CATEGORIZE(body.text)
| SORT count DESC
```

In today's environment, an SRE journey can start from anywhere in Elastic. Let's see how Elastic empowers SREs to find the needle in the haystack among the massive volume of logs received every day, thanks to Elastic's agentic capabilities. \
\
Elastic brings deep data exploration across different signals through Kibana and ES|QL, our query language. While there is always a learning curve with any new solution, Elastic's agentic capabilities lower that barrier by translating natural language into queries — making data exploration accessible to everyone. \
\
Let's see this in action. Using the ES|QL query in **Tab 1**, I'm scoping to just the OVH network namespace — filtering out any other scenarios running on the platform. I can immediately see a sudden spike in log volume.

Focus on ERROR log messages

Thanks to Elastic's AI capabilities, I simply express my intent and the assistant translates it into the right ES|QL filter. We can see a sharp increase in ERROR messages — but how can I tell which specific error is driving this spike? \
\
That's where Machine Learning log categorization kicks in. Switching to **Tab 2**, I use ES|QL's `CATEGORIZE()` function to automatically group error logs by pattern and sort them by count — pinpointing the exact category behind the surge.

Switch to the **Tab 2** query in the ES|QL editor — or click **"Machine Learning Categorisation"** in Discover.

Great — we can now clearly see which category is causing the spike. Let's dive into the investigation.

Hit the **expand button** on a trace containing the error.

The `body.text` field shows unstructured data where our error message is located.

Expand the message to reveal the full error.

`VRACK-VLAN-SEGMENTATION` appears to be the reason behind the spike. The log tells us there's a VLAN tagging mismatch between datacenters — instances in GRA11 can't reach instances in SBG5 over the vRack private network, with 85% packet loss. \
\
But what does it actually mean, and where is it coming from? The answer lies in the application traces, where data is correlated across the entire application — making the transaction readable from end to end. \
\
What if we could ask our agent to perform a first-level error analysis?

---

## Act 3 — AI-Assisted Analysis

AI Assistant

Click **"Explain this log entry"**.

Thanks to the Elastic AI Context Engine, the agent has full context and specialized skills to run a detailed analysis — determining an initial root cause from the unstructured error message, assessing the impact, and proposing the right next steps. \
\
The agent identifies that the `VRACK-VLAN-SEGMENTATION` error is caused by a VLAN tagging mismatch between the GRA and SBG datacenters. It sees the cascade: the vRack segmentation is causing OVHcloud Load Balancer backends in SBG to become unreachable, MKS pods are entering CrashLoopBackOff because cross-DC services are down, and Gateway NAT is rerouting outbound traffic through the public internet as a fallback. \
\
This gives us a solid first-level analysis. Now, let's turn this into an official case to kick off a formal incident.

Click **"Start conversation"**.

We now enter a conversational experience where we can interact directly with our OVHcloud Network Analyst agent, which carries over the previous analysis context to act on our requests. In this case, we want to create a dashboard to track this specific ERROR message so we have clear evidence once the issue is resolved. Let's ask it to build that for us.

Create a dashboard to track progress of this VRACK-VLAN-SEGMENTATION issue

Once created, click the **dashboard preview**.

Thanks to the agent, we now have a dedicated dashboard to track the resolution of our vRack VLAN segmentation issue — showing error rate over time, affected datacenters, and cross-DC latency trends.

Save the dashboard.

---

## Act 4 — Case Creation & Runbook

Human-in-the-Loop

We can now turn this context and dashboard into an actionable case to track the resolution and build the final RCA. Let's ask the agent to handle this for us.

Create an observability case and link the dashboard to track the resolution. Find the appropriate remediation actions from the internal runbook.

Click **"Go to case"**.

All the findings have been collected and consolidated into a single, comprehensive case — containing not only our investigation details, but also the exact remediation steps extracted from our internal runbook. \
\
The runbook prescribes verifying VLAN IDs across all datacenters using the OVH CLI and OpenStack commands, checking the vRack configuration for consistency, and if needed, re-applying the VLAN tagging via the OVH API. The runbook states that this remediation requires manual approval — the vRack is the backbone of our private networking, so any reconfiguration must be validated by an SRE.

---

## Act 5 — The Reveal: Auto-Remediation Was Already Running

Auto-Remediate

Up until this point, we've seen how Elastic's AI capabilities empower SREs to dramatically speed up mean time to detection and resolution. But what if the system could heal itself? \
\
That's actually what was happening in the background while we were investigating: a second incident — `LB-HEALTHCHECK-FLAP` — was detected, triaged, and analyzed automatically, complete with a full root cause analysis and an evidence-backed confidence score. The HAProxy health checks were flapping due to backend readiness probes failing during the vRack disruption. The system identified this as a self-healing condition: once the flapping subsided, backends would auto-recover through HAProxy's built-in rise/fall mechanism. \
\
All of this was packaged inside a case, simply waiting for the SRE to approve the runbook remediation for the primary vRack issue. \
\
Sounds too good to be true? Let's take a look.

Click the **email notification** in your inbox → click on the **Case** (show briefly the content) → click on **"conversation"** → **validate the remediation**.

The workflow has now executed the remediation plan based on the internal runbook and tracked progress — keeping the case continuously updated until full resolution. Per runbook guidelines, this remediation enforced human validation with a "human-in-the-loop" check before triggering the vRack reconfiguration fix.

Go back to **case or cockpit** and watch the resolution happen. The vRack cross-DC latency drops back to normal, backend health recovers to 100%, uptime SLO stops burning.

---

## Act 6 — Closing

In this demonstration, you've seen how Elastic's native agentic capabilities empower SREs to go all the way from finding the needle in the logs to full incident resolution on OVHcloud infrastructure — while keeping human control over critical remediation. \
\
From vRack private networking to HAProxy load balancing, from Anti-DDoS VAC scrubbing to OVHcloud Connect hybrid connectivity — Elastic observes it all through a single platform, correlates signals across services and datacenters, and uses AI to accelerate every step of the incident lifecycle.

---

## Appendix — Demo Flow Timeline

**T-0:00** — Inject CH-01 (vRack VLAN Segmentation, HITL) + CH-12 (LB Health Check Flap, Auto) from Chaos Console

**T+0:30** — Open Application Cockpit, show SLO burning and KPI impact

**T+1:30** — Navigate to Discover, run Tab 1 ES|QL query on ovhnet namespace, show log spike

**T+3:00** — Run Tab 2 ES|QL (CATEGORIZE) → identify VRACK-VLAN-SEGMENTATION

**T+4:00** — Expand log entry, show body.text, click "Explain this log entry"

**T+5:30** — AI analysis: root cause, cascade impact, next steps

**T+6:30** — Start conversation → ask agent to create tracking dashboard

**T+8:00** — Ask agent to create case with runbook remediation

**T+9:30** — Reveal: auto-remediation of CH-12 was running in background

**T+10:30** — Email notification → approve HITL remediation for CH-01

**T+11:30** — Watch resolution on cockpit: metrics recover, SLO stops burning

**T+12:00** — Closing remarks

---

## Appendix — Key OVH-to-GCP Mapping for Presenters

If you've delivered the GCP version before, here's how the beats map:

| Beat | GCP version | OVH version |
| --- | --- | --- |
| Primary fault | `VPC-ROUTE-LIMIT-EXCEEDED` | `VRACK-VLAN-SEGMENTATION` |
| Auto-remediate | CH-16 Backend Unhealthy | CH-12 LB Health Check Flap |
| Root cause | VPC peering 250-route limit | vRack VLAN tagging mismatch across DCs |
| CLI in runbook | `gcloud compute networks peerings list-routes` | `ovh-cli vrack get` / `openstack network list` |
| KPI impact | CDN cache hit rate, Cloud Armor blocks | vRack cross-DC latency, Backend health % |
| Cascade | VPC → LB → NAT | vRack → LB → MKS |
| AI agent | Google Cloud Network Analyst | OVHcloud Network Analyst |