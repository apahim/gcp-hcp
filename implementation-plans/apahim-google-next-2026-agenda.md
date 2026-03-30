# Google Cloud Next 2026 - Personal Agenda

* **Conference**: Google Cloud Next 2026, April 22-24, Mandalay Bay, Las Vegas
* **Attendee**: APahim - SPSE on GCP HCP
* **Focus**: GCP Architecture, AI agents for SRE/Ops, AI-centric SDLC & DevEx
* **Strategy**: Packed schedule of breakouts; team to cover GKE-only, security, and observability tracks

---

## At a glance

- **21 sessions** across 3 days (7 Wed, 8 Thu, 6 Fri) + **28 sessions suggestions** for the team
- **2 keynotes**, **15 breakouts**, **1 workshop**, **1 developer meetup**, **1 discussion group**

## Themes
- Wed: **Agents + platform strategy** day (agent security, production agents, GKE roadmap, ADK)
- Thu: **SRE + operations** day (agentic SDLC, MCP governance, Gemini for SRE, autonomous SRE, networking)
- Fri: **Infrastructure deep dive** day (GKE AI internals, network ops, multi-cluster gateway hands-on, multi-tenant GKE)

### Highlights

- **Securing agents in Google Cloud** (Wed 11:00) — IAM, governance, and runtime defense for agents; critical for our agent security posture
- **Agents transform SRE** (Wed 2:45) — Anna Berenberg (Google SRE) on the future of agentic SRE; validates our diagnostician agent direction
- **ADK roadmap** (Wed 5:00) — Google's agent development framework roadmap; shapes our agent strategy
- **Scaling autonomous SRE with Vertex AI Agent Engine** (Thu 4:00) — most directly relevant session for our diagnostician agent and Cloud Workflows automation
- **Multi-Cluster Gateway Workshop** (Fri 11:00) — hands-on, 90 min; directly applicable to our multi-cluster GKE infrastructure
- **Multi-tenant AI on GKE** (Fri 1:45) — discussion group on our exact challenge: scheduling, priority, fairness across tenant control planes

---

## Wednesday April 22

| Time | Session | Format | Why |
|------|---------|--------|-----|
| 9:00 AM | [Opening keynote: The agentic cloud](https://www.googlecloudevents.com/next-vegas/session/3922022/opening-keynote-the-agentic-cloud) | Keynote | Google's strategic direction for agentic cloud; sets context for all other sessions |
| 11:00 AM | [What's next in IAM: Security, governance, and runtime defense for AI agents](https://www.googlecloudevents.com/next-vegas/session/3913017/securing-agents-in-google-cloud-iam-governance-and-runtime-defense) | Breakout | Critical for securing our agent deployments; IAM patterns for agent identity and runtime defense |
| 12:30 PM | [Build production-ready agents on Google Cloud: A guide for architects and CTOs](https://www.googlecloudevents.com/next-vegas/session/3913053/build-production-ready-agents-on-google-cloud-a-guide-for-architects-and-ctos) | Breakout | Architecture patterns for production agents; informs our diagnostician agent design decisions |
| 1:30 PM | [What's new with GKE: A look toward the next decade](https://www.googlecloudevents.com/next-vegas/session/3912917/what's-new-with-gke-a-look-toward-the-next-decade) | Breakout | GKE is our core platform; roadmap visibility shapes HCP architecture and upgrade planning |
| 2:45 PM | [A peek into the future: How agents transform SRE](https://www.googlecloudevents.com/next-vegas/session/3913050/a-peek-into-the-future-how-agents-transform-sre) | Breakout | Anna Berenberg (Google SRE) on agentic SRE; validates our diagnostician agent direction |
| 4:00 PM | [Navigate the agentic shift in software development with Google](https://www.googlecloudevents.com/next-vegas/session/3912230/navigate-the-agentic-shift-in-software-development-with-google) | Breakout | Shapes our AI-centric SDLC strategy; how Google sees agents changing dev workflows |
| 5:00 PM | [Explore Google's Agent Development Kit capabilities and roadmap](https://www.googlecloudevents.com/next-vegas/session/3913122/explore-google's-agent-development-kit-capabilities-and-roadmap) | Breakout | ADK roadmap shapes our agent framework choices; evaluate for diagnostician agent toolchain |

### Other sessions for team's consideration

- [Build agents at scale: How Shopify powers Sidekick with Claude on Vertex AI](https://www.googlecloudevents.com/next-vegas/session/3913106/build-agents-at-scale-how-shopify-powers-sidekick-with-claude-on-vertex-ai) (11:00 AM)
  - **Priority delegate.** Real-world Claude agent architecture at production scale; directly relevant to our Claude-based tooling
- [Automating excellence: How Gemini and Config Connector help create 10x cloud teams](https://www.googlecloudevents.com/next-vegas/session/3912910/automating-excellence-how-gemini-and-config-connector-help-create-10x-cloud-teams) (11:00 AM)
  - Config Connector patterns for fleet automation; relevant to our infrastructure-as-code approach
- [Google MCP Services: Connect AI agents to cloud infrastructure in minutes](https://www.googlecloudevents.com/next-vegas/session/3912288/google-mcp-services-connect-ai-agents-to-cloud-infrastructure-in-minutes) (11:15 AM)
  - Google's official MCP servers for GCP; evaluate for integration with our agent tooling
- [Deploy and govern the Gemini CLI at scale](https://www.googlecloudevents.com/next-vegas/session/3911930/deploy-and-govern-the-gemini-cli-at-scale) (12:30 PM)
  - Governance patterns for CLI-based AI tools at enterprise scale; informs our DevEx rollout
- [Scaling the fleet: Spotify's vision for the future of Config Connector](https://www.googlecloudevents.com/next-vegas/session/3912950/scaling-the-fleet-spotify's-vision-for-the-future-of-config-connector) (1:30 PM)
  - Spotify's fleet management at scale with Config Connector; parallel to our HCP fleet challenges
- [Engineering the future of Kubernetes for AI at scale](https://www.googlecloudevents.com/next-vegas/session/3912949/engineering-the-future-of-kubernetes-for-ai-at-scale) (4:00 PM)
  - K8s evolution for AI workloads; impacts how we design control planes for AI-heavy tenants
- [Transform cloud operations and management with generative AI](https://www.googlecloudevents.com/next-vegas/session/3913058/transform-cloud-operations-and-management-with-generative-ai) (4:00 PM)
  - GenAI for cloud ops; complementary perspective to our SRE agent strategy
- [What's new for AI on GKE: Training, serving, and agents](https://www.googlecloudevents.com/next-vegas/session/3912907/what's-new-for-ai-on-gke-training-serving-and-agents) (5:15 PM)
  - GKE AI roadmap for training and serving; informs tenant workload support on HCP

---

## Thursday April 23

| Time | Session | Format | Why |
|------|---------|--------|-----|
| 8:00 AM | [The agentic shift in the software development life cycle](https://www.googlecloudevents.com/next-vegas/session/3911926/the-agentic-shift-in-the-software-development-life-cycle) | Breakout | Directly informs our AI-centric SDLC adoption; how agents reshape dev lifecycle end-to-end |
| 9:15 AM | [Tame the AI sprawl: Unify agents and MCP servers](https://www.googlecloudevents.com/next-vegas/session/3913129/tame-the-ai-sprawl-unify-agents-and-mcp-servers) | Breakout | MCP governance patterns; relevant as we scale our MCP server fleet and agent tooling |
| 10:30 AM | [Get real: Agents in the autonomous era](https://www.googlecloudevents.com/next-vegas/session/3920528/get-real-agents-in-the-autonomous-era) | Keynote | Day 2 keynote; Google's vision for autonomous agents sets the strategic framing |
| 11:45 AM | [Supercharge your SRE team with Gemini Cloud Assist](https://www.googlecloudevents.com/next-vegas/session/3913125/supercharge-your-sre-team-with-gemini-cloud-assist) | Breakout | Evaluate Gemini Cloud Assist for HCP operations; potential complement to our diagnostician agent |
| 1:30 PM | [Platform Engineering Meetup](https://www.googlecloudevents.com/next-vegas/session/3913493/platform-engineering-meetup) | Developer Meetup | Networking with platform engineering practitioners; learn how others run hosted platforms at scale |
| 2:45 PM | [Master GKE "Day 2" operations: Pain-free upgrades at scale](https://www.googlecloudevents.com/next-vegas/session/3912904/master-gke-"day-2"-operations-pain-free-upgrades-at-scale) | Breakout | Directly applicable to HCP fleet upgrades; patterns for pain-free GKE lifecycle management |
| 4:00 PM | [Scaling autonomous SRE with Vertex AI Agent Engine](https://www.googlecloudevents.com/next-vegas/session/3912988/scaling-autonomous-sre-with-vertex-ai-agent-engine) | Breakout | Most relevant session for our diagnostician agent; Vertex AI Agent Engine for SRE automation |
| 5:15 PM | [Cross-Cloud Network: An intelligent, governed, performant fabric for global AI](https://www.googlecloudevents.com/next-vegas/session/3913143/cross-cloud-network-an-intelligent-governed-performant-fabric-for-global-ai) | Breakout | Cross-cloud networking roadmap; impacts our HCP network architecture and multi-cloud strategy |

### Other sessions for team's consideration

- [The knowledge source: Reliable product docs for AI agents via MCP](https://www.googlecloudevents.com/next-vegas/session/3912243/the-knowledge-source-reliable-product-docs-for-ai-agents-via-mcp) (9:30 AM)
  - MCP for product docs; patterns for feeding reliable context to our agents
- [Build and scale GKE workloads: A master class in node-level performance](https://www.googlecloudevents.com/next-vegas/session/3912942/build-and-scale-gke-workloads-a-master-class-in-node-level-performance) (9:30 AM)
  - Node-level GKE performance tuning; directly applicable to control plane node optimization
- [Untrusted code, unprecedented speed: High-velocity runtimes for AI agents](https://www.googlecloudevents.com/next-vegas/session/3912922/untrusted-code-unprecedented-speed-high-velocity-runtimes-for-ai-agents) (10:30 AM)
  - Sandboxing and runtime isolation for agent-generated code; security patterns for agent execution
- [Path to production: Improve and scale agent quality with OpenTelemetry](https://www.googlecloudevents.com/next-vegas/session/3913040/path-to-production-improve-and-scale-agent-quality-with-opentelemetry) (1:15 PM)
  - OTel for agent observability; informs how we instrument and monitor our agent fleet
- [How to accelerate AI agents: GKE agent sandbox and pod snapshots](https://www.googlecloudevents.com/next-vegas/session/3912914/how-to-accelerate-ai-agents-gke-agent-sandbox-and-pod-snapshots) (1:15 PM)
  - GKE sandbox and pod snapshot features; potential for faster agent cold starts on HCP
- [Limitless AI orchestration on GKE: Power secure, planetary-scale AI](https://www.googlecloudevents.com/next-vegas/session/3912915/limitless-ai-orchestration-on-gke-power-secure-planetary-scale-ai) (4:00 PM)
  - Large-scale GKE orchestration patterns; relevant to our multi-cluster HCP architecture
- [Unlocking enterprise actions: Bridge your APIs to AI agents with MCP and Apigee](https://www.googlecloudevents.com/next-vegas/session/3913003/unlocking-enterprise-actions-bridge-your-apis-to-ai-agents-with-mcp-and-apigee) (5:15 PM)
  - MCP + API gateway integration; complements the "Tame the AI sprawl" session on MCP governance
- [Achieve state-of-the-art inference: High performance on TPUs and GPUs with llm-d](https://www.googlecloudevents.com/next-vegas/session/3912927/achieve-state-of-the-art-inference-high-performance-on-tpus-and-gpus-with-llm-d) (5:15 PM)
  - High-performance inference on accelerators; relevant if HCP tenants run inference workloads
- [From toil to innovation: Achieving scalable, reliable observability](https://www.googlecloudevents.com/next-vegas/session/3913064/from-toil-to-innovation-achieving-scalable-reliable-observability) (5:15 PM)
  - Observability at scale; patterns for reducing toil in our HCP monitoring stack

---

## Friday April 24

| Time | Session | Format | Why |
|------|---------|--------|-----|
| 8:30 AM | [How GKE builds itself with AI](https://www.googlecloudevents.com/next-vegas/session/3912947/how-gke-builds-itself-with-ai) | Breakout | Insider view on GKE's own AI-driven development; patterns we can adopt for HCP engineering |
| 9:45 AM | [Network monitoring and automated network operations in cross-cloud networks](https://www.googlecloudevents.com/next-vegas/session/3913107/network-monitoring-and-automated-network-operations-in-cross-cloud-networks) | Breakout | Automated network ops directly applicable to HCP networking; monitoring patterns for our fleet |
| 11:00 AM | [Deploying a Multi-Cluster Gateway Across GKE Clusters](https://www.googlecloudevents.com/next-vegas/session/3924727/deploying-a-multi-cluster-gateway-across-gke-clusters) | Workshop (90 min) | Hands-on with multi-cluster gateway; directly applicable to our multi-cluster GKE infrastructure |
| 12:30 PM | [SRE at 5 bi-events: Predictive ops with BigQuery, Vertex AI & Gemini](https://www.googlecloudevents.com/next-vegas/session/3908811/sre-at-5-bi-events-predictive-ops-with-bigquery-vertex-ai-gemini) | Solution Talk | Predictive ops patterns using Vertex AI; informs our approach to proactive incident detection |
| 1:45 PM | [The priority paradox: Navigating multi-tenant AI on GKE](https://www.googlecloudevents.com/next-vegas/session/3976464/the-priority-paradox-navigating-multi-tenant-ai-on-gke) | Discussion Group | Our exact challenge: scheduling, priority, and fairness across tenant control planes on GKE |

### Other sessions for team's consideration

- [Choose the right stack for Go agents](https://www.googlecloudevents.com/next-vegas/session/3911927/choose-the-right-stack-for-go-agents) (8:30 AM)
  - Go agent framework selection; relevant to our Go codebase and agent development strategy
- [Platform engineering for AI: Architect a unified stack on GKE](https://www.googlecloudevents.com/next-vegas/session/3912902/platform-engineering-for-ai-architect-a-unified-stack-on-gke) (8:30 AM)
  - Unified platform stack on GKE for AI; architectural patterns applicable to HCP platform design
- [Operationalize AI: A blueprint for managing enterprise agents at scale](https://www.googlecloudevents.com/next-vegas/session/3913024/operationalize-ai-a-blueprint-for-managing-enterprise-agents-at-scale) (9:45 AM)
  - Enterprise agent lifecycle management; governance and operational patterns at scale
- [How OpenAI builds Kubernetes GPU clusters](https://www.googlecloudevents.com/next-vegas/session/3912935/how-openai-builds-kubernetes-gpu-clusters) (9:45 AM)
  - Competitor intelligence on large-scale K8s cluster management; relevant to HCP design
- [From CLIs to IDEs to PRs: How agents fit into real developer workflows](https://www.googlecloudevents.com/next-vegas/session/3952115/from-clis-to-ides-to-prs-how-agents-fit-into-real-developer-workflows) (11:00 AM)
  - Agent integration in dev workflows; informs our AI-centric SDLC and DevEx strategy
- [Building agents with Go](https://www.googlecloudevents.com/next-vegas/session/3912219/building-agents-with-go) (11:00 AM, Discussion)
  - Hands-on Go agent patterns; directly applicable to our Go-based agent implementations
- [Beyond fine-tuning: The next frontier of training and RL on GKE](https://www.googlecloudevents.com/next-vegas/session/3912900/beyond-fine-tuning-the-next-frontier-of-training-and-rl-on-gke) (11:00 AM)
  - Advanced training on GKE; informs what tenant workloads HCP may need to support
- [Better together: Scaling AI with GKE and Cloud Run](https://www.googlecloudevents.com/next-vegas/session/3912920/better-together-scaling-ai-with-gke-and-cloud-run) (12:30 PM)
  - GKE + Cloud Run hybrid patterns; evaluate for lightweight agent deployment alongside HCP
- [Accelerate CI/CD with coding agents](https://www.googlecloudevents.com/next-vegas/session/3911929/accelerate-cicd-with-coding-agents) (1:45 PM)
  - Coding agents in CI/CD pipelines; directly applicable to our AI-centric SDLC goals
- [Build beyond the CPU: Native AI and agentic autoscaling on GKE](https://www.googlecloudevents.com/next-vegas/session/3912937/build-beyond-the-cpu-native-ai-and-agentic-autoscaling-on-gke) (1:45 PM)
  - Agentic autoscaling on GKE; novel scaling patterns for AI workloads on our platform
- [Boost your resilience posture with native fault injection testing](https://www.googlecloudevents.com/next-vegas/session/3975074/boost-your-resilience-posture-with-native-fault-injection-testing) (1:45 PM)
  - Native fault injection on GKE; directly applicable to HCP reliability and chaos testing
