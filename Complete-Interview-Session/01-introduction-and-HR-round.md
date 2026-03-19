# Complete Interview Session — File 1: Introduction & HR Round

> **How to use this file:** Read each question out loud, then answer using the provided script. Practice until it feels natural. This simulates exactly how a real interview begins.

---

## PHASE 1: BEFORE THE INTERVIEW STARTS

### Mindset Checklist
- [ ] You have 6.2 years of real production experience — that is your superpower
- [ ] You have worked on systems serving 60,000+ businesses at Vendasta
- [ ] You have hands-on experience with GKE, ArgoCD, Datadog, Terraform, AWS, GCP
- [ ] You are not a fresher — you are a Senior engineer. Speak with authority.
- [ ] Interviewers want to HIRE you. They are on your side.

### What to Prepare Before Interview
- 3 strong STAR stories ready (incident you resolved, migration you led, system you improved)
- Numbers ready: cluster sizes, uptime %, latency reductions, cost savings
- Questions to ask the interviewer (shows genuine interest)
- Know the company's tech stack from job description

---

## PHASE 2: INTRODUCTION ROUND

---

### Q1. "Tell me about yourself."

**[INTERVIEWER INTENT]:** They want to understand your journey, assess communication skills, and see if you fit the role.

**[YOUR ANSWER — Script to memorize and personalize]:**

"Sure! My name is Chethan H M, and I'm a Senior DevOps and Site Reliability Engineer with 6.2 years of experience building and maintaining production infrastructure at scale.

I started my career at Ascer Solutions where I built my foundation — setting up CI/CD pipelines with Jenkins, deploying workloads on AWS, and writing Bash automation scripts for routine operations.

I then moved to Altruist Technologies as a DevOps Engineer, where I managed AWS infrastructure — EC2, VPC, RDS, S3, IAM — and containerized legacy applications using Docker. I introduced Prometheus and Grafana for monitoring and improved deployment frequency significantly using Jenkins pipelines.

Currently, I'm a Senior SRE at Vendasta Technologies, one of the largest B2B SaaS platforms serving over 60,000 businesses worldwide. My responsibilities include managing GKE clusters, driving GitOps adoption using ArgoCD, building observability pipelines with Datadog, defining SLOs and error budgets, and leading infrastructure reliability initiatives using Terraform on GCP.

I've handled everything from zero-downtime migrations, incident management, Kubernetes autoscaling, cost optimization, to onboarding teams onto internal developer platforms.

I'm now looking for a senior role where I can continue solving large-scale reliability and infrastructure challenges, and contribute to a strong engineering culture."

**[KEY METRICS TO ADD]:**
- '60,000+ businesses served'
- '99.9% SLO maintained'
- 'Reduced deployment time by 60% using GitOps'
- 'Managed 15+ microservices on GKE'

---

### Q2. "Walk me through your current role and day-to-day responsibilities."

**[YOUR ANSWER]:**

"In my current role at Vendasta as a Senior SRE, my day typically involves three layers of work:

**Reliability Engineering:** I maintain SLIs and SLOs for our critical services. I own the error budget policy — when error budget is at risk, I work with dev teams to freeze features and focus on reliability. I use Datadog dashboards to monitor RED metrics — Rate, Errors, Duration — for each microservice.

**Infrastructure and Platform:** I manage GKE clusters on GCP, handle Terraform modules for infrastructure provisioning, and own ArgoCD-based GitOps deployments. Any new microservice onboarding goes through me for infrastructure review.

**Incident Management:** I'm part of the on-call rotation. I handle P1/P2 incidents, write post-mortems, and drive action items to prevent recurrence. I've built runbooks in Confluence for all critical failure scenarios.

On the side, I also mentor junior engineers on Kubernetes troubleshooting, write internal documentation, and contribute to platform engineering initiatives like standardizing Helm charts and improving CI/CD pipelines."

---

### Q3. "Why are you looking for a change?"

**[YOUR ANSWER — Keep it positive, forward-looking]:**

"I've had a fantastic journey at Vendasta. I've grown significantly — from deploying infrastructure to owning full reliability engineering for large-scale systems. I feel I've made a strong impact there.

I'm now looking for the next challenge — a role where I can take on more ownership, work on larger-scale systems or more complex infrastructure challenges, and potentially grow into a principal or staff SRE role. I'm also excited about [mention something specific about the company you're interviewing with — their scale, their tech stack, their mission].

I'm not leaving because of dissatisfaction — I'm leaving because I'm ready to grow into the next level."

---

### Q4. "What is your notice period?"

**[YOUR ANSWER]:**
"My current notice period is [X days/weeks]. However, I can negotiate an early release with my manager if there's a strong mutual fit. I'm motivated to join the right team and will do what it takes to make the transition smooth."

---

### Q5. "What are your salary expectations?"

**[YOUR ANSWER — Strategy: anchor high, stay flexible]:**

"Based on my 6.2 years of experience, my current compensation, and the market benchmarks for Senior SRE roles in Bangalore, I'm targeting a CTC in the range of [X to Y]. However, I'm open to discussion based on the complete package — learning opportunity, work culture, scope of impact, and growth trajectory are equally important to me."

**[TIP]:** Always give a range, not a fixed number. Put your target in the lower-middle of the range.

---

## PHASE 3: HR & BEHAVIORAL ROUND

---

### Q6. "Tell me about a major incident you handled. How did you manage it?"

**[STAR METHOD]**

**Situation:** "At Vendasta, we had a P1 incident where 3 GKE pods in our payment processing service were crash-looping in production. This impacted ~15% of our customers trying to access billing features."

**Task:** "As the on-call SRE, I was responsible for triaging the issue, communicating with stakeholders, and restoring service within our SLO target of 99.9%."

**Action:**
"First, I checked Datadog dashboards — error rate had spiked to 40% in the billing service. I ran `kubectl describe pod` and saw OOMKilled events — the pods were running out of memory. The root cause was a recent deployment that had introduced a memory leak in a new feature flag evaluation library.

I immediately rolled back the deployment using ArgoCD — one click rollback to the previous stable image. Pods recovered in under 3 minutes.

Simultaneously, I sent an incident update to the Slack #incidents channel every 10 minutes so stakeholders were informed.

After stabilization, I ran a post-mortem with the dev team. We identified that the new library hadn't been load-tested. We added memory profiling to the CI pipeline and set up a canary deployment strategy for that service."

**Result:** "Service was fully restored in 8 minutes. The post-mortem led to 3 action items — memory profiling in CI, canary rollouts, and load testing gates. We had zero recurrence of that issue."

---

### Q7. "Describe a time you improved a process or reduced manual work."

**[STAR METHOD]**

**Situation:** "At Altruist Technologies, the deployment process was entirely manual — developers would SSH into EC2 instances, pull Docker images, and restart containers. This took 45 minutes per deployment and was highly error-prone."

**Task:** "I was asked to modernize the CI/CD pipeline."

**Action:** "I designed a Jenkins declarative pipeline that automated everything — build, test, push to ECR, deploy to ECS or EC2 via Ansible playbooks. I also added Slack notifications for build status and deployment outcomes. I introduced environment-specific pipelines — dev, staging, production — with manual approval gates for production."

**Result:** "Deployment time dropped from 45 minutes to under 7 minutes. Human error in deployments went to zero. Developer confidence increased, and we went from deploying once a week to deploying multiple times a day."

---

### Q8. "Tell me about a time you disagreed with a team member or manager. How did you handle it?"

**[YOUR ANSWER]:**

"At Vendasta, there was a discussion about whether to use a managed database service or self-managed PostgreSQL on GKE for a new microservice. The developer team preferred self-managed for cost reasons, but I had concerns about operational overhead and reliability.

Rather than dismissing the idea, I prepared a comparison document — covering maintenance burden, backup complexity, failover time, and true total cost of ownership including engineering hours for managing it. I presented this in a team sync.

We collectively agreed that the managed Cloud SQL option was the right call, especially given our SLO requirements. The developer later thanked me because they didn't have to manage database upgrades or handle replication issues.

My approach was: data over opinion, and always make decisions in the best interest of the system and the team."

---

### Q9. "What is your biggest strength as an SRE/DevOps engineer?"

**[YOUR ANSWER]:**

"My biggest strength is the combination of deep technical breadth and a reliability-first mindset. I don't just automate things — I think about failure modes, observability, and what happens at 2 AM when something breaks.

I've built systems from scratch, and I've also had to recover systems under pressure. That breadth — knowing both the happy path and the failure path — is what makes me effective as an SRE.

I'm also strong at documentation and knowledge sharing, which I believe is underrated in our field. When I build something, I make sure the next engineer can understand it, operate it, and improve it without me being in the room."

---

### Q10. "What is an area you want to improve or a weakness?"

**[YOUR ANSWER — Be honest but show self-awareness and growth]:**

"One area I've been actively working on is public speaking and presenting technical topics to non-technical stakeholders. Historically I've been more comfortable in engineering discussions, but I realized that as a senior engineer, communicating the 'why' behind infrastructure decisions to product managers and leadership is just as important as the technical work itself.

I've been working on this by volunteering to present post-mortems and architecture reviews in broader team meetings. It's made a noticeable difference — I'm now more comfortable translating technical complexity into business impact."

---

### Q11. "Where do you see yourself in 3-5 years?"

**[YOUR ANSWER]:**

"In 3-5 years, I see myself in a Principal SRE or Staff DevOps Engineer role — someone who sets the reliability and infrastructure strategy at the organizational level. I want to be the person who defines how teams think about SLOs, designs multi-region architectures, and creates internal platforms that make hundreds of engineers more productive.

I'm also interested in eventually contributing to open-source infrastructure tooling and mentoring the next generation of SRE engineers."

---

### Q12. "Do you have any questions for us?"

**[ALWAYS ASK SMART QUESTIONS — Shows interest and senior thinking]:**

1. "What does the on-call rotation look like, and how does your team approach incident management and post-mortems?"
2. "What are the biggest reliability or infrastructure challenges the team is facing right now?"
3. "How do you measure the success of an SRE in this organization? What does great look like in this role?"
4. "What is the current state of your observability stack, and where do you want to take it?"
5. "What would you say are the biggest technical challenges I'd be working on in the first 6 months?"

---

## PHASE 4: CONFIDENCE BUILDERS

### Before Walking In (or Logging On)
- Take 3 deep breaths
- Remember: every tool on your resume, you have used in production
- Your experience at Vendasta serving 60,000+ businesses is genuinely impressive — own it
- Interviewers respect engineers who say 'I don't know but here's how I'd approach it' more than those who bluff

### Power Phrases to Use in Interviews
- 'In production at Vendasta, what we found was...'
- 'The tradeoff I considered was...'
- 'From an SRE perspective, the failure mode here would be...'
- 'I approached this by first understanding the blast radius...'
- 'The metric I would use to measure success here is...'

---

*Next: Move to File 02 — Core DevOps & SRE Concepts Round*
