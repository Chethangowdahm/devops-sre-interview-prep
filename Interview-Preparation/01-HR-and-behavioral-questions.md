# HR & Behavioral Questions - Complete Confidence Guide
> Chethan H M | Senior DevOps/SRE | 6.2 Years
> Read this the night before your interview. You are ready!

---

## THE STAR METHOD
**S**ituation - **T**ask - **A**ction - **R**esult
Use this structure for every behavioral question. Be specific. Use numbers.

---

## TELL ME ABOUT YOURSELF (Most Important!)

Practice this answer until it feels natural (90 seconds):

"I'm Chethan, a Senior DevOps and SRE Engineer with 6.2 years of experience building and maintaining scalable cloud infrastructure.

I started at Ascer Solutions doing AWS infrastructure, CI/CD, and Linux administration. I then joined Altruist Technologies where I built CI/CD pipelines with Jenkins, containerized applications using Docker, and managed AWS infrastructure — reducing manual setup time by 60% with Ansible automation.

For the past 1.5 years, I've been at Vendasta Technologies — a B2B SaaS platform serving 60,000+ businesses globally. As Senior SRE, I manage our entire GCP and GKE infrastructure. Key achievements include implementing GitOps with ArgoCD (reduced deployment errors by 70%), setting up comprehensive observability with Datadog and Prometheus, and owning our on-call and incident response.

I'm particularly strong in Kubernetes, GitOps, Terraform, and building observability pipelines. I'm looking for a Senior SRE/DevOps role where I can drive reliability at scale."

---

## BEHAVIORAL QUESTIONS & STORIES

### 'Tell me about a production incident you handled'

**Situation**: At Vendasta, API gateway was returning 500 errors for 40% of requests at 2 AM. Customer impact.
**Task**: I was on-call. Identify cause, mitigate, restore service.
**Action**: Checked Datadog dashboard → correlated with deployment 30 mins prior → rolled back in ArgoCD (git revert) → service restored in 15 minutes → wrote post-mortem.
**Result**: MTTR of 15 minutes. Post-mortem led to DB connection pool monitoring for all services.

---

### 'Tell me about improving a process'

**Situation**: At Vendasta, deployments took 45+ minutes, manual, error-prone.
**Task**: Reduce friction, eliminate errors.
**Action**: Implemented GitOps with ArgoCD. All config in Git, automated syncing.
**Result**: Deployment time from 45 minutes to under 5 minutes. Zero manual errors. Full audit trail.

---

### 'Most challenging technical problem'

**Situation**: Memory leak causing cascading failures — service OOMing, HPA scaling it further, consuming all node memory, evicting other services.
**Task**: Stabilize immediately, then find root cause.
**Action**: Set memory limits, paused HPA, restored stability. Profiled app — found DB connections not closing on timeout. Fixed connection lifecycle code.
**Result**: Service stable. Added memory + connection pool metrics. Zero recurrence.

---

### 'How do you handle disagreements?'

"I focus on data and outcomes, not opinions.

Example: Disagreed on Helm vs Kustomize. Instead of arguing, I proposed we both do a small proof-of-concept and evaluate against our team's specific needs.

We chose Helm based on the team's familiarity and ecosystem. I'm always open to being wrong — what matters is we make the right technical decision together."

---

### 'Tell me about working under pressure'

"During a major product launch at Vendasta (expected 10x traffic), I pre-scaled our cluster, load tested infrastructure, and prepared rollback runbooks.

During launch, one microservice became a bottleneck. I identified it quickly via Datadog, scaled it manually, and coordinated with the dev team to optimize a slow query.

Result: Successful launch, zero customer impact."

---

### 'Why are you leaving your current job?'

"Vendasta has been an excellent experience. I've grown significantly here. I'm looking for [a larger scale challenge / more ownership / specific technology you're excited about at the new company].

I'm particularly excited about this role because [specific reason related to company/tech stack/scale]."

**DO NOT**: Badmouth employer. Mention salary as only reason.

---

### 'Where do you see yourself in 5 years?'

"I'd like to be a Staff or Principal Engineer, leading platform reliability for millions of users. I want to deepen expertise in SRE principles, chaos engineering, and eventually mentor engineers.

This role excites me because it's a step toward that — the scale and complexity here will challenge me to grow significantly."

---

### 'What is your biggest weakness?'

"Early in my career, I was too perfectionistic with documentation — spending too long on runbooks when we needed action.

I've improved by time-boxing: 30 minutes for initial runbook, get it reviewed, iterate. Good enough now beats perfect later."

---

## QUESTIONS TO ASK THE INTERVIEWER

Always have 3-4 ready. Shows genuine interest.

1. "What does the on-call rotation look like, and how do you track and improve MTTR?"
2. "What's the biggest reliability or scalability challenge the team is facing right now?"
3. "How does the SRE team collaborate with development teams — embedded or centralized?"
4. "What does success look like in the first 90 days for this role?"
5. "What's the tech stack and infrastructure scale I'd be working with?"
6. "What's the career growth path from this role?"

---

## PRE-INTERVIEW CHECKLIST

**Night before**:
- [ ] Research company (product, tech stack, funding/growth, recent news)
- [ ] Map your experience to their job description
- [ ] Prepare 3 STAR stories
- [ ] Practice 'Tell me about yourself' aloud
- [ ] Test video/audio setup
- [ ] Sleep 7-8 hours

**Morning of**:
- [ ] Review key technical concepts for their stack
- [ ] Have water nearby
- [ ] Log in/arrive 5 minutes early
- [ ] Take 3 deep breaths before starting

**During interview**:
- [ ] Pause before answering (2-3 seconds is professional, not weak)
- [ ] Ask clarifying questions for ambiguous questions
- [ ] Be specific — numbers and real examples
- [ ] Think aloud for system design
- [ ] Okay to say: 'I haven't done X specifically, but here's how I'd approach it'

---

## SALARY NEGOTIATION

- Don't give a number first if you can avoid it
- 'I'm looking for something competitive with the market for 6+ years SRE. What budget do you have?'
- If pushed, give a range (research Glassdoor, LinkedIn Salary first)
- Always negotiate — first offer rarely best. Ask once at minimum.

---

## CONFIDENCE MANTRAS

**Before walking in/logging in, say to yourself**:

1. I have 6.2 years of REAL production experience at scale (60,000+ businesses).
2. I have solved real problems: cascading failures, deployment automation, infrastructure at scale.
3. I know Kubernetes deeply — I've run production clusters, managed incidents, optimized performance.
4. Nervousness means I care. It will pass after the first 2 minutes.
5. I prepared this entire study guide. I am ready.
6. It's a two-way conversation. I'm also evaluating them.

**Remember**: Senior engineers regularly say 'I'd need to look that up' or 'I'm not sure of the exact syntax but the approach would be...' — that's fine. What matters is your problem-solving approach and experience.

**You've got this, Chethan.**
