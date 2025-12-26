# CV_expansion

The Complete Call+ Story - Polished Version
Here's how you should narrate your Call+ experience from start to finish, incorporating all the improvements:

Opening: Setting the Context
Interviewer: "Tell me about your work on the Call+ platform at Yubi."
You:
"I joined Yubi during a critical period—they had just acquired Spocto, a debt collection platform, and I was responsible for leading the technical integration of Spocto's Call+ product into Yubi's ecosystem. This was particularly important because we were simultaneously onboarding large PSU clients, which eventually contributed to deals worth over ₹200 Crores.
Call+ is a digital loan collection platform that enables banks to manage their loan recovery operations. Banks send us allocation data—basically lists of loan accounts that need collection activity—and our platform enables call center agents to reach out to borrowers, track payment promises, and maintain detailed audit trails of all interactions."

Part 1: Understanding the System Architecture (Current State)
Interviewer: "Walk me through the current architecture."
You:
"Let me take you through the end-to-end flow, from data ingestion to reporting.
Data Ingestion Flow:
We receive allocation data from approximately 10 banks via SFTP drops. Each bank is whitelisted to a dedicated server for security isolation and to avoid creating a single point of failure. The data comes in Excel format, typically once a month, with volumes ranging from 3-5 lakh allocations per bank, totaling around 40-50 lakh records monthly.
Once we receive the data, it goes through a dedicated data pipeline service that sanitizes and normalizes it. Different banks send data in different formats—one might use 'loan_account_id' while another uses 'loan-account-no'—so we maintain a mapping configuration per bank to transform all incoming data into a common schema.
After sanitization, the data flows to our Central Data Service, or CDS. This is a microservice built on Node.js with MongoDB as its database. CDS does two critical things:

It assigns each allocation a unique spocto_id, which becomes the UUID for that record across our entire ecosystem
It publishes allocation records to platform-specific Kafka topics based on configuration—if an allocation is enabled for Call+ and Field+ (our field collection product), it gets published to both topics

The Call+ service consumes from its dedicated Kafka topic, and that's how allocations land in the Call+ database. We have retry logic with exponential backoff and dead letter queues on the CDS side to handle any publish failures, and similar error handling on the Call+ consumer side.
Hierarchical Data Assignment:
Once data is in Call+, it flows through a four-tier hierarchy: Department Heads → Supervisors → Team Leads → Agents.
The assignment works in two distinct phases:
Phase 1 - Batch-level assignment (DH → Supervisor → TL):
Higher-level roles work at the batch level. When a DH assigns 5 lakh allocations to a Supervisor, we track this in a separate batch_assignments table with columns like batch_id, assigned_to_user_id, and role. This doesn't touch individual allocation records yet—it's purely for workload distribution and visibility.
Phase 2 - Record-level assignment (TL → Agent):
When a Team Lead assigns work to agents, they select specific ranges of spocto_ids—say, 500 allocations to Agent A and 500 to Agent B. At this point, we update the assigned_to field on the individual allocation records. Only after an allocation is tagged to an agent can it be called.
Visibility is hierarchical—you can see all allocations assigned to anyone in your vertical below you. An agent only sees their assigned allocations, a TL sees all allocations for their agents, and so on up the chain.
The Calling Workflow:
When an agent logs in, they see their assigned allocations. Each allocation has a complete profile—loan type, outstanding amount, due date, previous interaction history—but with PII masked for privacy.
When the agent clicks 'Call,' we make an API call to VC Dialer, our third-party telephony provider, passing the phone number and agent ID. VC Dialer returns a call_id (a combination of agent_id, phone number, and timestamp) which we store. We record the button-click timestamp as the call start time.
The calls are manual—agents speak with borrowers, and all calls are automatically recorded by VC Dialer. After the call ends, the agent selects a disposition from a predefined list of 25 options—things like 'Payment Promised,' 'Callback Requested,' 'Wrong Number,' etc. Some dispositions require additional fields—for example, 'Payment Promised' requires a promise-to-pay date.
When the agent submits the disposition, we record the submission timestamp as call end time. This data gets written to the Call+ database and simultaneously published as an event to Kafka for CDS to consume and maintain its copy.
Now, I should note—our call start and end times are based on UI interactions, not actual telephony events, so they include dial time and ringing. We reconcile with actual call data from VC Dialer during our end-of-day sync.
Call Recording Sync:
VC Dialer stores all call recordings. We run a scheduled job after 7 PM—when calling operations have ended for the day—to pull recordings from VC Dialer using the call_id as the key. We store these in S3 and update our database with the S3 URLs.
We do this EOD rather than real-time for two reasons: First, we don't want to overwhelm VC Dialer's servers during business hours when agents are actively calling. Second, exact call durations and recordings aren't time-critical—they're used for quality checks and audits, not real-time operations. This approach keeps costs down without impacting user experience.
360-Degree View:
From any allocation profile, users can click to see a 360-degree view. This makes an API call to CDS, which returns all activities across the entire product ecosystem—not just Call+ actions, but also field visits, SMS campaigns, payment updates, everything. This gives complete context on the collection efforts for that borrower.
This is an eventually consistent view—if an agent just updated a disposition and immediately checks the 360 view, there might be a small lag before CDS reflects the latest update. We accept this trade-off because blocking reads for consistency would hurt performance, and the lag is typically under a few seconds.
Report Generation:
When a user requests a report—say, 'all calls made in March for Bank X with Payment Promised disposition'—the request is published to a Kafka topic. A dedicated Report Generation microservice consumes from this topic.
The service queries the Call+ database, chunks the results into 50,000-record batches, writes each chunk to a file, flushes memory, and moves to the next chunk. This prevents the memory exhaustion issues we had earlier with the monolithic system.
Once the report is complete, it's uploaded to S3, a presigned URL is generated (valid for 24 hours for security), and the frontend polls a view on our database to see when the status changes from 'queued' to 'completed.' The user can then download the report, and optionally have it emailed.
We can generate 4 reports in parallel. Additional requests queue up. A full report with 50 lakh records takes about 3 minutes. For large reports, we require explicit confirmation from the user—they must check a box and provide a reason—to prevent accidental full-data exports that could strain the system."

Part 2: The PSU Integration Challenge
Interviewer: "You mentioned PSU clients. What were the specific challenges there?"
You:
"PSU clients brought two major dimensions of complexity: strict technical requirements and on-premise deployment constraints.
Technical Requirements:
These were government entities with non-negotiable SLAs—99.99% uptime, which translates to less than 5 minutes of downtime per month. They also needed comprehensive audit trails, data residency guarantees, and security certifications.
At the time I joined, the platform wasn't ready for this. It was a monolithic Node.js application with Python scripts bolted on for report generation, running on traditional VMs with stateful components. Any failure could cascade and bring down the entire system.
On-Premise Deployment:
Unlike our other clients who used our cloud-hosted solution, PSUs required on-premise deployment within their infrastructure. This meant we couldn't rely on AWS services like S3, and we had to work within their infrastructure constraints—limited compute resources, no external monitoring tools, restricted network access.
The first PSU client came onboard while we were mid-migration to the new architecture. This actually worked in our favor—by the time their deployment was complete, we had the new stateless, containerized architecture in place. This became our reference implementation for pitching to other PSUs."

Part 3: The Architecture Revamp
Interviewer: "Walk me through what you changed and why."
You:
"The revamp happened in phases, and I'll break it down by the problems we solved.
Problem 1: Stateful Components Causing Instability
The original monolith stored state in several problematic ways:

Session management and call state: Stored in local server memory, which meant requests were tied to specific servers via sticky sessions. If a server went down, all active sessions were lost.
File processing: Temporary files and generated reports were stored on local filesystems. If the server crashed mid-generation, you lost the work and had no way to resume.
Application-level caching: Each server instance had its own cache, leading to inconsistent data and cache invalidation nightmares.
Background job state: Long-running tasks tracked their progress in memory, so any crash meant starting over.

The solution:

Externalized sessions and call state to Redis, enabling any instance to handle any request
Moved file storage to S3 (or S3-compatible storage for on-prem), making it instance-independent
Replaced local caches with shared Redis caches
Used Kafka for job queuing with database-tracked state, so jobs could be picked up by any worker

Problem 2: Report Generation Killing the System
This was our biggest pain point. The Python report generation process would load entire datasets—sometimes 50 lakh records—into memory. This would exhaust available RAM and crash the server, taking down the entire platform.
Additionally, there were no mandatory filters on report requests. Users could accidentally click 'Generate Report' without selecting any filters, triggering a full-data export that would bring everything down.
The solution:

Service isolation: Extracted report generation into its own microservice. Now if it crashes, it only affects reports, not the calling operations.
Chunking: Implemented streaming with 50k record chunks. Load chunk → write to file → flush memory → next chunk. Memory usage stays constant regardless of total report size.
Mandatory filters: Made filter selection required in the UI (button stays disabled until filters are selected). On the backend, if no filters are present, we require explicit full-report confirmation with a text justification. This prevents accidental full exports.
Resource management: The report service can generate 4 reports in parallel with queuing for additional requests. This prevents resource exhaustion while maintaining reasonable throughput.

Problem 3: Monolithic Deployment and Scaling
With everything in one application, we couldn't scale components independently. If calling volume spiked, we had to scale the entire monolith, including the report generation we didn't need more of.
The solution:
We decomposed the monolith into focused microservices:

Call+ API Service (Node.js): Handles all API requests from the frontend—allocations, profiles, dispositions, user management
Report Generation Service (Python): Dedicated to report creation, listening to Kafka for requests
CDS Consumer Service: Consumes Kafka events and keeps CDS in sync
Integration Services: Separate services for third-party integrations (VC Dialer, SFTP handlers)

Each service has its own database/datastore, communicates via Kafka or REST APIs, and can be scaled independently based on load.
Problem 4: Containerization and Orchestration
After breaking into microservices, we containerized each service with Docker. Benefits included:

Minimal dependencies: Each container has only what it needs—the Node.js container doesn't have Python libraries, keeping it lean
Independent scaling: We can run 10 instances of the API service but only 2 of the report service
Deployment isolation: Deploy a new version of reports without touching the API
Environment parity: Same container runs in dev, staging, production, and on-prem

We deployed on Kubernetes for orchestration, which handled:

Automatic restarts on failures
Load balancing across instances
Zero-downtime deployments with rolling updates
Resource quotas to prevent any service from monopolizing resources

Migration Strategy:
We took an incremental approach, not big bang:

Phase 1: Decomposed the monolith into services while keeping them on VMs
Phase 2: Containerized each service one at a time
Phase 3: Moved to Kubernetes orchestration
Phase 4: Externalized all state (Redis, S3, Kafka)

This let us derisk each change and maintain service during the transition. Critical because PSU clients were onboarding in parallel—we couldn't afford extended downtime."

Part 4: On-Premise Deployment Specifics
Interviewer: "How did the on-prem deployment differ from your cloud setup?"
You:
"The on-prem deployments had several constraints we had to adapt to:
Storage:
We couldn't use AWS S3, so we deployed MinIO—an S3-compatible object storage solution that runs on their infrastructure. This gave us the same API interface, so minimal code changes were needed. We just swapped the endpoint configuration.
Messaging:
Kafka was deployed on-premise on their servers. We had to work with their ops team to size the cluster appropriately and configure retention policies based on their storage constraints.
Orchestration:
Kubernetes was deployed on their infrastructure. We provided the deployment manifests and worked with their team on capacity planning. The challenge was they had fixed resources—we couldn't just scale horizontally like in cloud—so we had to be more thoughtful about resource allocation.
Monitoring:
This was a pain point. They didn't allow external monitoring tools for security reasons, so we couldn't use our New Relic or cloud-based Grafana. Instead:

We set up a self-hosted Grafana + Prometheus stack on their infra
Implemented structured logging to local files
Built a log aggregation pipeline for troubleshooting
During initial deployment, we literally had to travel to their location to view logs and debug issues

Load Testing & Capacity Planning:
We stress-tested the on-prem setup to validate it could handle their expected load—up to 10 lakh allocations. The testing, which was conducted by our DevOps team in collaboration with their IT team, validated:

Allocation ingestion throughput
Concurrent agent capacity (how many agents could call simultaneously without degradation)
API response times under load (targeting 95th percentile under 500ms)
Report generation performance with maximum expected data volumes

The system started showing degradation as we approached the 10 lakh record mark, but that was acceptable given their infrastructure constraints and expected usage patterns.
Multi-Tenancy:
Initially, we maintained separate code branches for each PSU client because they had unique customization requests. This became a maintenance nightmare—bug fixes had to be cherry-picked across branches.
We later refactored to a config-driven approach. All customizations are now controlled through configuration files and feature flags, with a single codebase. This made deployments and bug fixes dramatically easier—one fix applies to everyone, but behavior is customized per client through config."

Part 5: Outcomes and Impact
Interviewer: "What were the results?"
You:
"The impact was significant across multiple dimensions:
Reliability:
We went from frequent outages—sometimes multiple times a week when reports crashed the system—to 99.99% uptime. That's less than 5 minutes of downtime per month. This was critical for PSU clients who had this as a contractual requirement.
Business Impact:
The platform's reliability and our ability to deploy on-premise enabled us to close deals with multiple PSU clients, contributing to over ₹200 Crores in contract value. These were high-value, multi-year contracts that became a significant revenue stream for Yubi.
Operational Efficiency:

Report generation no longer blocks other operations
We can scale services independently based on actual load
Deployments went from risky, downtime-requiring events to routine rolling updates with zero downtime
On-call burden reduced dramatically—fewer 3 AM pages for system crashes

Scalability:
The new architecture handles 40-50 lakh allocations per month comfortably, with headroom to grow. Individual services can be scaled based on need—during month-end when banks send fresh allocations, we scale up ingestion workers without touching other components.
Technical Debt Reduction:
Moving from a monolith to well-defined microservices made the codebase much more maintainable. New engineers can contribute to specific services without understanding the entire system. Testing became easier—unit tests for services in isolation, integration tests for inter-service contracts."

Part 6: Handling Follow-Up Technical Questions
Now let me prepare you for likely follow-up questions on the critical areas:

Deep Dive Preparation: The 5 Critical Areas
1. Kafka Partitioning & Ordering Guarantees
Core Concepts:
What is a partition in Kafka?
A partition is an ordered, immutable sequence of messages within a topic. Topics are split into multiple partitions for parallelism and scalability. Each partition is hosted on one broker and has a leader that handles all reads and writes.
What is a partition key?
A partition key is a value you specify when producing a message to Kafka. Kafka uses this key to determine which partition the message goes to by applying a hash function: partition = hash(key) % number_of_partitions
Why does it matter?

Ordering guarantee: Kafka only guarantees message order within a single partition, not across partitions
Related messages together: Using the same key for related messages ensures they land in the same partition and are processed in order
Load distribution: Different keys distribute across partitions for parallel processing

Your Call+ Scenario:
Current problem:
You're not using a partition key when publishing disposition updates to CDS. This means:
Time T1: Agent updates allocation A123 → disposition = "Callback Requested" 
         → Goes to Partition 0

Time T2: Agent corrects allocation A123 → disposition = "Payment Promised"
         → Goes to Partition 1 (round-robin)

Your single CDS consumer reads from both partitions.
It might process Partition 1 (T2) before Partition 0 (T1)
Result: Final state is "Callback Requested" (wrong!)
The fix:
javascript// When publishing to Kafka
producer.send({
  topic: 'cds-updates',
  messages: [{
    key: allocation.spocto_id,  // This is the partition key
    value: JSON.stringify(dispositionUpdate)
  }]
});
With spocto_id as the partition key:

All updates for allocation A123 always go to the same partition (e.g., Partition 2)
They're processed in order
No risk of newer data being overwritten by older data

Follow-up questions you might get:
Q: What if you have millions of unique spocto_ids? Won't that create too many partitions?
A: "No, partition count is fixed when you create the topic—say, 10 partitions. Millions of different spocto_ids will be distributed across those 10 partitions via hashing. Multiple IDs hash to the same partition, but all messages for a single ID always go to the same partition, which is what we need for ordering."
Q: How do you handle hot partitions if one spocto_id gets updated much more frequently?
A: "In our case, allocations aren't updated frequently enough for this to be a problem—typically a few updates per allocation over its lifecycle. But if it were an issue, we'd monitor partition lag and potentially increase partition count, or use a composite key like bank_id:spocto_id to spread load more evenly."
Q: What about your current setup? Have you seen out-of-order issues?
A: "Honestly, we haven't observed it in production yet, likely because our update frequency per allocation is low and there's enough time between updates that ordering issues don't manifest. But architecturally, it's a gap. If we see increased update rates—say, if we add real-time payment status webhooks—this could become a real problem. Adding partition keys is low-hanging fruit for reliability."

2. Saga Pattern vs. Eventual Consistency
Core Concepts:
What is the Saga pattern?
Saga is a design pattern for managing distributed transactions across microservices. Instead of a single ACID transaction, you have a sequence of local transactions with compensating transactions to undo changes if something fails.
Two types:

Orchestration: A central coordinator tells each service what to do and handles rollbacks
Choreography: Services react to events and trigger the next step; no central coordinator

Compensating transactions:
If Step 3 fails, you execute compensation for Step 2 and Step 1 to rollback the system to a consistent state.
Eventual consistency:
A different model where you accept temporary inconsistency, knowing the system will converge to a consistent state eventually (usually through retries or background reconciliation).
Your Call+ Scenario:
What you actually have (be precise):
You don't have a true Saga pattern. What you have is eventual consistency with retry-based reconciliation:

Agent submits disposition
Update Call+ DB (local transaction)
Attempt to publish to Kafka with exponential backoff retries
If successful, set sync_flag = 1
If failed after retries, leave sync_flag = 0
Background job periodically scans for sync_flag = 0 and retries publishing

Why this isn't a Saga:

You're not rolling back the Call+ DB update if Kafka publish fails
You don't have compensating transactions
You're relying on eventual convergence via retry, not orchestrated compensation

The honest explanation:
"I mentioned Saga earlier, but to be precise, we're actually implementing eventual consistency with retry-based reconciliation. Here's what happens:
When an agent submits a disposition, we update the Call+ database in a local transaction. We then attempt to publish the update to Kafka for CDS consumption. If that publish fails despite exponential backoff retries, we don't roll back the database update—we set a sync_flag = 0 on that record.
A background job runs every few minutes, scanning for records with sync_flag = 0, and retries publishing them. This means there's a window—usually seconds to minutes—where Call+ and CDS are out of sync. But eventually, they converge.
We accept this temporary inconsistency because:

The Call+ database is the source of truth for calling operations
Blocking agents until CDS confirms receipt would hurt user experience
The 360-degree view (which reads from CDS) being slightly stale is acceptable for the use case

If we needed stronger consistency, we'd implement a true Saga with compensating transactions—for example, if Kafka publish fails, we'd either write a tombstone record to indicate 'pending sync' or publish a 'revert' event once CDS is back online."
Follow-up questions you might get:
Q: What if the background job fails to publish repeatedly? Do records stay out of sync forever?
A: "We have monitoring that alerts if sync_flag = 0 records exceed a threshold or age beyond 15 minutes. At that point, we investigate—usually it's a Kafka outage or network issue. In extreme cases, we can manually trigger sync or perform a reconciliation job that compares Call+ and CDS states and pushes missing updates."
Q: How do you handle idempotency? If you retry publishing the same update multiple times, could CDS process it multiple times?
A: "Good question. Each disposition update event includes the spocto_id and a timestamp field. CDS uses these to deduplicate—before processing an event, it checks if it already has a more recent update for that spocto_id. If yes, it discards the event. This makes the processing idempotent."
Q: What happens if the database update succeeds but the application crashes before attempting the Kafka publish?
A: "That's a gap in our current implementation. The record would have sync_flag = 0 by default (assuming we set it optimistically), so the background job would pick it up. But you're right—there's a race condition window. A more robust approach would be to use the outbox pattern: write both the DB update and an 'outbox' entry in a single local transaction, then have a separate process read the outbox and publish to Kafka. That guarantees at-least-once delivery."

3. Load Testing & Performance Validation
Core Concepts:
Load testing validates that a system can handle expected production loads. Key metrics:

Throughput: Requests/transactions per second
Latency: Response time (usually measured at percentiles: p50, p95, p99)
Concurrency: Number of simultaneous users/requests
Resource utilization: CPU, memory, disk I/O, network
Error rate: Percentage of failed requests under load
Saturation point: Where adding more load causes degradation

Types of load tests:

Load test: Expected peak production load
Stress test: Beyond peak until system breaks (find limits)
Soak test: Sustained load over hours/days (find memory leaks, resource exhaustion)
Spike test: Sudden traffic spikes (verify auto-scaling)

Your Call+ Scenario:
Since you weren't directly involved, here's how to frame it honestly while showing understanding:
The honest answer:
"The load testing was primarily handled by our DevOps and QA teams in collaboration with the PSU's IT team, as they had specific infrastructure they wanted validated. I was involved in defining the test scenarios and reviewing the results, but didn't execute the tests myself.
From what I remember reviewing in the reports:
Test scenarios included:

Allocation ingestion: We validated we could ingest a full month's worth of data (50 lakh records) within an acceptable timeframe—I believe the target was under 4 hours for a full batch.
Concurrent agent load: We simulated varying numbers of agents making calls simultaneously. The on-prem deployment was sized for up to 500 concurrent agents, and we validated that API response times stayed under 500ms at the 95th percentile at that concurrency.
Disposition write throughput: Since dispositions trigger database writes and Kafka publishes, we tested sustained disposition submission rates to ensure we wouldn't bottleneck.
Report generation under load: We generated multiple reports concurrently while the system was under calling load to ensure report generation didn't degrade the core calling experience.
Stress testing: We pushed beyond expected load to find the saturation point. For the on-prem deployment, we found degradation starting around 10 lakh active allocations in the database with 500 concurrent agents.

Key findings that influenced our design:

We needed to separate report generation into its own service because under combined load, it competed for resources and degraded API response times
Redis was critical—without it for session/cache, we couldn't maintain low latency at high concurrency
Database connection pooling needed careful tuning; too few connections caused queueing, too many caused contention

If you need more specific numbers, I'd need to dig into the detailed test reports we generated, but those were the high-level outcomes that validated our architectural decisions."
Follow-up questions you might get:
Q: What tool did you use for load testing?
A: "The QA team primarily used JMeter for simulating agent load and API calls, with some custom Python scripts for simulating the data ingestion pipeline. For the Kafka load testing, we used Kafka's own performance testing tools to validate throughput and lag under various producer/consumer configurations."
Q: How did you establish performance baselines?
A: "We started with requirements from the PSU contracts—they specified maximum response times and minimum concurrent user capacity. We also looked at our existing cloud deployment metrics to understand typical usage patterns, then scaled those up based on the PSU's projected agent count and allocation volumes."
Q: Did you do any performance optimization based on the test results?
A: "Yes, a few key optimizations came out of testing:

We added database indexes on fields commonly used in queries (like assigned_to, batch_id, disposition_status)
We tuned Redis cache TTLs based on actual data access patterns observed during tests
We right-sized our Kubernetes resource requests and limits for each service based on observed usage
We implemented connection pooling for the database with specific sizing based on measured concurrency"


4. On-Premise Infrastructure Details
Core Concepts:
On-premise deployment means running software on the customer's infrastructure rather than cloud. Key challenges:

Limited resources: Can't elastically scale like cloud
Infrastructure differences: Different storage, networking, security tools
No managed services: Can't use AWS S3, RDS, managed Kafka
Limited access: Usually can't SSH directly, must work through customer IT
Compliance: Must meet customer's security, audit, data residency requirements

Your Call+ Scenario:
What you should be able to articulate:
"The on-premise deployments required us to adapt our architecture to work with their infrastructure constraints:
Storage:
We replaced AWS S3 with MinIO, which is an S3-compatible object storage that runs on-premise. The beauty of MinIO is that it implements the S3 API, so our application code didn't need to change—we just pointed to a different endpoint in our configuration. The PSU's IT team managed the MinIO deployment and capacity planning.
Messaging Infrastructure:
We deployed Kafka on-premise on their servers. This required working closely with their ops team on:

Cluster sizing (broker count, replication factor)
Retention policies (how long to keep messages based on their storage constraints)
Network configuration (ensuring services could reach Kafka brokers)
Monitoring setup (since they didn't allow external monitoring tools)

Container Orchestration:
Kubernetes was deployed on their infrastructure. We provided:

Deployment manifests (YAML files defining our services, resource requirements, scaling policies)
Capacity planning guidance (based on our load testing)
Runbooks for common operational tasks

The challenge here was fixed resources. In cloud, if we need more capacity, we just scale horizontally. On-prem, they had a fixed number of servers, so we had to be more thoughtful about resource allocation and limits per service to prevent any single service from monopolizing resources.
Monitoring & Observability:
This was one of the bigger pain points. They didn't allow external monitoring tools like New Relic or cloud-hosted Grafana for security reasons. So we:

Set up self-hosted Grafana + Prometheus: Deployed on their infrastructure to collect metrics from our services
Structured logging: All services output JSON-structured logs to local files
Log aggregation: Built a pipeline (ELK stack or similar) to aggregate logs from all containers for searchability
Alerting: Configured Prometheus alerting rules that integrate with their internal ticketing system

The biggest operational difference: In cloud, we could view dashboards and logs from anywhere. On-prem, especially during initial deployment, we often had to be physically present at their location to troubleshoot issues since we didn't have remote access initially.
Security & Compliance:
PSUs had specific requirements:

Data residency: All data must stay within their premises
Network isolation: Services could only communicate within their internal network
Access control: We couldn't have direct SSH access; all operations went through their IT team
Audit trails: Every action needed to be logged and retained for regulatory compliance

Deployment Process:
Unlike our cloud deployment where we'd push updates directly:

We'd build container images and package them
Transfer images to their network (often via secure file transfer)
Their IT team would deploy updates during approved maintenance windows
We'd provide runbooks and be available (sometimes on-site) during deployment
Validation testing post-deployment to ensure everything worked

Multi-Tenancy Management:
Initially, each PSU had a separate code branch because they requested custom features. This became unsustainable—imagine having to cherry-pick bug fixes across 5+ branches.
We refactored to a config-driven approach:

Single codebase for all deployments
Customer-specific behavior controlled via configuration files
Feature flags for optional functionality
Client-specific branding/labels through config

This meant:

One bug fix applies to everyone automatically
New features developed once, enabled per client via config
Much easier to maintain and test

The configuration file looks something like:
yamlclient_id: "psu_bank_a"
deployment_type: "on_premise"
storage:
  type: "minio"
  endpoint: "https://minio.psubank.internal"
kafka:
  brokers: ["kafka1.internal:9092", "kafka2.internal:9092"]
features:
  advanced_reporting: true
  sms_integration: false
capacity:
  max_allocations: 1000000
  concurrent_agents: 500
```

### **Performance Constraints:**

The on-prem deployments were sized for their expected load—typically up to 10 lakh allocations. We stress-tested to that number and found:
- Below 8 lakh: Performance was excellent
- 8-10 lakh: Noticeable slowdown in some queries, but acceptable
- Above 10 lakh: Significant degradation

This was acceptable because:
1. Their contracts were for that volume
2. Upgrading their hardware to support more would be their decision
3. We documented the limits clearly

If they wanted to handle more, we'd work with them on hardware upgrades or database optimizations."

### **Follow-up questions you might get:**

**Q: What was the biggest operational difference between cloud and on-prem?**

A: "Speed of iteration. In cloud, if something breaks, we can roll back in minutes. We have full observability and can deploy hotfixes immediately. On-prem, every change requires coordination with their IT team, scheduled maintenance windows, and often physical presence. A fix that would take 15 minutes in cloud could take days on-prem due to process overhead. This forced us to be much more careful about testing before deployment."

**Q: How did you handle database management on-prem?**

A: "We typically ran MySQL in containers managed by Kubernetes, with persistent volumes on their storage infrastructure. Their IT team handled backups, but we provided scripts and procedures. We also had to be more thoughtful about schema migrations—we used tools like Flyway to version and automate migrations, but tested them extensively in staging before production since rollbacks were harder."

**Q: Did you use any infrastructure-as-code tools?**

A: "We used Helm charts for Kubernetes deployments, which made it easier to package and version our applications. Configuration was managed through Helm values files, making it relatively simple to deploy to different environments. For their infrastructure provisioning (VMs, networking), they typically used their own tools—sometimes Terraform, sometimes their internal provisioning systems."

---

## 5. Microservices Communication & Data Consistency

### **Core Concepts:**

In microservices, services are independently deployable and own their data. Communication patterns:

1. **Synchronous (REST/gRPC):** Service A calls Service B and waits for response
   - Pros: Simple, immediate response, strong consistency
   - Cons: Tight coupling, service dependencies, latency, cascade failures

2. **Asynchronous (Messaging/Events):** Service A publishes event, Service B subscribes
   - Pros: Loose coupling, resilience, scalability
   - Cons: Eventual consistency, complexity, debugging harder

**Data consistency models:**

- **Strong consistency:** All services see the same data immediately (ACID transactions)
- **Eventual consistency:** Services might see different data temporarily, but converge

### **Your Call+ Scenario:**

**The architectural pattern:**

"Our microservices communicate using a hybrid approach:

### **Synchronous (REST APIs):**

Used for request-response patterns where immediate feedback is needed:

- Frontend calls Call+ API service for allocation listings, profile views, disposition submission
- Call+ API calls VC Dialer API to initiate calls
- Call+ API calls CDS API for 360-degree view (pulling data from CDS)

Example flow:
```
User clicks "Call" 
→ Frontend POST /api/allocations/{id}/call
→ Call+ API service receives request
→ Call+ API calls VC Dialer: POST /dialer/initiate_call
→ VC Dialer returns call_id
→ Call+ API returns response to frontend
```

We use REST because:
- The frontend needs immediate feedback (can't wait for async processing)
- The telephony integration is synchronous by nature
- The 360 view is a real-time pull of data

### **Asynchronous (Kafka Events):**

Used for state changes that other services need to know about:

- Call+ publishes disposition updates to Kafka for CDS consumption
- CDS publishes allocation ingestion events to Kafka for Call+ (and other platforms) consumption
- Report generation requests are published to Kafka for the report service

Example flow:
```
Agent submits disposition
→ Call+ updates its database
→ Call+ publishes event: { spocto_id, disposition, timestamp, ... }
→ Kafka stores event
→ CDS consumer reads event (eventually)
→ CDS updates its database
We use Kafka because:

CDS doesn't need to know immediately (eventual consistency is acceptable)
If CDS is down, Call+ isn't blocked (resilience)
Multiple consumers can react to the same event (extensibility)
We have durability—if a consumer is down, it catches up when it restarts

Data Consistency Trade-offs:
We accept eventual consistency between Call+ and CDS. Here's why and how we handle it:
Why it's acceptable:

CDS is not critical for calling operations—agents don't need CDS to do their job
The 360-degree view (which reads from CDS) being slightly stale is acceptable
Strong consistency would require synchronous calls to CDS, which would:

Block agents if CDS is slow or down
Increase latency
Create tight coupling



How we ensure eventual convergence:

Kafka provides durability—events aren't lost
CDS consumers use at-least-once delivery with idempotency
Background jobs scan for sync_flag = 0 records and retry
We monitor lag (time between event publish and CDS processing)
Alerts fire if lag exceeds thresholds (e.g., 5 minutes)

Edge case handling:
Scenario: Agent submits disposition, immediately checks 360 view

Call+ DB is updated
Event published to Kafka
Agent clicks 360 view
Call+ calls CDS API
CDS hasn't processed the event yet (lag of 2-3 seconds)
360 view shows stale data (doesn't have latest disposition)

We handle this with:

No special handling in production—we accept eventual consistency
Users understand there might be a small delay
In the future, we could add a "last updated" timestamp to the view to set expectations

Why not use distributed transactions (2PC)?
Two-phase commit could give us strong consistency, but:

It's complex to implement correctly
It's slow (latency of coordination)
It reduces availability (if coordinator fails, everyone blocks)
Our use case doesn't require it—eventual consistency is sufficient

Service dependencies and failure handling:
We design for failures:

Circuit breakers: If VC Dialer is consistently failing, we open the circuit and show a user-friendly error instead of repeatedly hammering it
Timeouts: Every external call has a timeout (e.g., 5 seconds for VC Dialer)
Retries with backoff: Transient failures get retried with exponential backoff
Fallbacks: If CDS is down when fetching 360 view, we show just the Call+ data with a message "Full history temporarily unavailable"

Service ownership:
Each service owns its data:

Call+ owns calling data (allocations assigned to agents, dispositions, call logs)
CDS owns the single source of truth for allocation metadata
Report service owns report generation state and output files

This prevents tight coupling—Call+ doesn't directly query CDS's database; it uses CDS's API.
Deployment independence:
Because services communicate asynchronously via Kafka, we can deploy them independently:

Deploy new Call+ version → doesn't affect CDS
Deploy new report service → doesn't affect calling operations
CDS goes down for maintenance → Call+ continues operating, events queue up in Kafka, CDS catches up when it's back

This independence is critical for achieving high availability."
Follow-up questions you might get:
Q: What happens if Kafka itself goes down?
A: "If Kafka is completely down, we have a problem—Call+ can't publish disposition updates, and they won't reach CDS. However:

Call+ continues operating—agents can still make calls and submit dispositions because we update the Call+ DB first
Those updates get sync_flag = 0 since we can't publish
When Kafka comes back online, our background job retries publishing the queued updates
Kafka is deployed as a cluster (multiple brokers) for high availability, so complete Kafka failure is rare

In production, we've never had a total Kafka outage. We've had individual broker failures, but Kafka's replication ensures no data loss and minimal disruption."
Q: How do you handle schema evolution—if Call+ changes the event format, how does CDS handle it?
A: "We use backward-compatible schema evolution:

New fields are added as optional, never breaking existing fields
We version our events (e.g., event_version: 2) so consumers can handle multiple versions
Major breaking changes are coordinated: deploy consumers first (they understand both old and new format), then deploy producers (sending new format)
Ideally, we'd use a schema registry (like Confluent Schema Registry) to enforce compatibility checks, but we haven't implemented that yet—it's on our roadmap."

Q: What's your strategy for testing microservices interactions?
A: "We have multiple testing layers:

Unit tests: Each service tests its logic in isolation with mocked dependencies
Contract tests: We use contract testing (ideally something like Pact) to ensure Call+'s expectations of CDS's API match what CDS actually provides
Integration tests: We spin up services with a test Kafka instance and verify end-to-end flows
End-to-end tests: In staging, we run full user journey tests against the actual system

The challenge is microservices make testing more complex. You can't just test everything in one monolithic test suite—you need to think about service boundaries and contracts."

Final Preparation Tips
Practice these narratives out loud:

The complete Call+ story (10-15 minutes)
The architecture revamp story (5-7 minutes)
The PSU deployment challenges (5 minutes)

Have these numbers memorized:

40-50 lakh allocations per month
10 banks
99.99% uptime
₹200 Cr in deals
4 parallel reports
3 minutes for full report
10 lakh record capacity for on-prem
25 disposition types

Be ready to draw diagrams:

Overall system architecture
Data flow (SFTP → Pipeline → CDS → Kafka → Call+)
Hierarchical assignment (DH → Supervisor → TL → Agent)
Kafka partitioning concepts

When you don't know something:

"I don't recall the exact details, but the approach was..."
"That was handled by [team], but I can speak to the outcomes..."
"I'd need to check my notes on the specifics, but the high-level approach was..."

Connect everything back to business value:

Don't just say "we added Kafka"—say "we added Kafka to enable asynchronous processing, which improved resilience and allowed CDS to be down without blocking agent operations"
Don't just say "we containerized"—say "containerization enabled independent scaling and zero-downtime deployments, which was critical for meeting PSU SLAs"

You've got this. The key is confidence, clarity, and honesty. Good luck with the Tide interview!
