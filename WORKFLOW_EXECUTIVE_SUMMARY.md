# AppArmor Profile Management Workflow
## Executive Summary

**Document Version:** 1.0  
**Last Updated:** 2025-11-21  
**Target Audience:** Management, Project Stakeholders  
**Target Environment:** Raspberry Pi production fleet with DEV testing machine

## Answer of previous question

**Process list (Objective 1):** During every baseline capture (both initial and post-update), task3-create-baseline.yml invokes create_baseline.sh, which records the full set of Apache processes (PIDs, command lines, users). These baselines are stored under /root/apache-monitor/data/baselines/ and fetched back to data/baselines/ for review.

**Day 0 (pre-update) file/path inventory (Objective 2):** The initial baseline (“Day 0”) captures all paths Apache accessed under the original profile. That JSON output is archived and used when constructing the first AppArmor profile, so you have the exact filesystem footprint at deployment time.

**Day 2 (post-update) file/path inventory (Objective 3):** After any regular or emergency OS update, we rerun the baseline on DEV and feed the “Day 2” snapshot into detect_apache_changes.sh. The resulting delta (apache-delta-*.json/.txt) explicitly lists new, removed, and denied file paths. That’s what drives the profile updates and the approval package.

---

## Overview

This document provides a high-level overview of the automated AppArmor security profile management system for Apache services on client production sites. The system ensures security compliance while minimizing operational risk through automated testing and validation.

### Key Benefits

✅ **Zero Production Downtime** - All changes tested on DEV before production  
✅ **Automated Detection** - System automatically identifies required security changes  
✅ **Human-in-the-Loop** - Critical decisions require human approval  
✅ **Rollback Ready** - Quick recovery if issues arise  
✅ **Audit Trail** - Complete documentation of all changes

---

## System Architecture

```
┌─────────────────┐      ┌──────────────────┐      ┌─────────────────────┐
│  Control Panel  │      │   DEV Machine    │      │  Production Fleet   │
│                 │─────▶│  (Test Site)     │─────▶│  (Client Sites)     │
│                 │      │   Mirrors Prod   │      │   Multiple RPis     │
└─────────────────┘      └──────────────────┘      └─────────────────────┘
```

**No Continuous Monitoring on Production Sites**
- The system does NOT require continuous monitoring or data collection from client production sites
- Production sites operate independently after initial profile deployment
- Workflows are only triggered when OS updates are needed (monthly scheduled or emergency)

---

## Three Operating Phases

### Phase 1: Regular OS Updates (Monthly)

**Trigger:** Scheduled monthly security patch cycle  
**Timeline:** 2-3 days total  
**Risk Level:** Low

```
DEV Testing (Day 1) → Human Approval (Day 1-2) → Production Rollout (Day 2-3)
```

**Process:**
1. Apply OS update to DEV machine
2. System automatically detects security profile changes needed
3. System tests the updated profile on DEV
4. Generate approval document with test results
5. **Human approval required** before production
6. Deploy approved profile to production
7. Apply OS update to production sites (rolling, one at a time)
8. Monitor for 24-72 hours

**Human Decision Point:**
- Review: Are detected changes expected for this OS update?
- Review: Did all DEV tests pass?
- Review: Any security concerns with new file access patterns?
- Decision: Approve or reject deployment to production

---

### Phase 2: Emergency Updates (As Needed)

**Trigger:** Critical security vulnerability (CVE)  
**Timeline:** 4-8 hours  
**Risk Level:** Medium (urgency vs. thoroughness tradeoff)

```
Quick DEV Test (1-2h) → Fast Approval (30min) → Staged Rollout (2-4h) → Intensive Monitoring (2h)
```

**Process:**
1. Assess emergency severity and scope
2. Fast-track testing on DEV (abbreviated tests)
3. Generate emergency approval document
4. **On-call team approval** (abbreviated review)
5. Deploy to one production site (canary)
6. If canary succeeds, roll out to remaining sites
7. Intensive monitoring for first 2 hours

**Human Decision Point:**
- Decision: Can we test on DEV or must go direct to production?
- Review: Did DEV tests pass?
- Decision: Approve emergency deployment?
- Decision: Continue rollout after canary or abort?

---

### Phase 3: Profile Refinement (On Demand)

**Trigger:** New application features or configuration changes  
**Timeline:** 1-2 days  
**Risk Level:** Low

```
Identify Need → DEV Testing → Human Approval → Production Deployment
```

**Process:**
1. Change (new web application, Apache config change) is identified
2. Replicate the change on DEV machine
3. System detects and applies required profile updates
4. Test thoroughly on DEV
5. Generate approval document
6. **Human approval required**
7. Deploy refined profile to production

**Human Decision Point:**
- Review: Are the detected changes appropriate for the new feature?
- Review: Did comprehensive testing pass?
- Decision: Approve deployment to production?

**Note:** This phase is only used when application changes are made - NOT for routine monitoring.

---

## Roles and Responsibilities

| Role | Responsibilities | Time Commitment |
|------|------------------|-----------------|
| **System Administrator** | Initiate workflows, review approval documents, make go/no-go decisions | 1-2 hours per update cycle |
| **Automation System** | Execute tests, detect changes, generate reports | Fully automated |
| **On-Call Engineer** | Emergency approvals, handle rollbacks if needed | On-call availability |

---

## Risk Mitigation

### Before Production Deployment

✅ All changes tested on identical DEV environment  
✅ Automated functional testing validates Apache still works  
✅ Human review of all detected changes  
✅ Approval document provides audit trail

### During Production Deployment

✅ Rolling deployment (one site at a time)  
✅ Automatic backups before any changes  
✅ Immediate verification after each site  
✅ Pause deployment if issues detected

### After Production Deployment

✅ 24-72 hour monitoring period  
✅ Quick rollback available (< 5 minutes)  
✅ Escalation procedures documented  
✅ Post-deployment baseline captured

---

## Workflow Decision Matrix

**Which workflow should we use?**

| Situation | Workflow | Timeline | Approval Level |
|-----------|----------|----------|----------------|
| Monthly OS security patches | Phase 1: Regular Update | 2-3 days | Standard review |
| Critical CVE requiring immediate patch | Phase 2: Emergency Update | 4-8 hours | Fast-track review |
| New web application deployed | Phase 3: Profile Refinement | 1-2 days | Standard review |
| Apache configuration change | Phase 3: Profile Refinement | 1-2 days | Standard review |
| Routine operations | No action | N/A | N/A |

---

## Success Metrics

### Automation Efficiency
- **Manual effort per update:** < 2 hours (vs. 8+ hours manual)
- **Testing coverage:** 100% automated functional tests
- **Detection accuracy:** Automated change detection vs. manual analysis

### Operational Safety
- **Production incidents:** Target 0 incidents from profile updates
- **Rollback rate:** Target < 5% of deployments require rollback
- **Downtime:** Target 0 minutes downtime per update

### Compliance
- **Audit trail:** 100% of changes documented
- **Approval compliance:** 100% of production changes have human approval
- **Security posture:** AppArmor profiles always current with OS version

---

## Escalation Procedures

### Level 1: Automated System Failure
- **Issue:** DEV tests fail, automation errors
- **Response:** System administrator reviews logs, may require profile refinement
- **Timeline:** Resume after troubleshooting (usually < 4 hours)

### Level 2: Production Issues After Deployment
- **Issue:** Production site shows errors after profile deployment
- **Response:** Immediate rollback to previous profile (< 5 minutes)
- **Timeline:** Investigate and retest on DEV before retrying

### Level 3: Complete System Failure
- **Issue:** Multiple production sites affected, rollback unsuccessful
- **Response:** Disable AppArmor enforcement temporarily, escalate to senior engineering
- **Timeline:** Immediate mitigation, root cause analysis within 24 hours

---

## One-Time Setup (Phase 0)

**Timeline:** 4-6 hours (one-time)  
**Required Before First Use:**

1. ✅ Set up control panel with automation tools
2. ✅ Deploy monitoring infrastructure to DEV and production sites
3. ✅ Create initial security baselines
4. ✅ Deploy initial AppArmor profiles
5. ✅ Verify DEV machine mirrors production configuration

**After setup is complete, the system is ready for ongoing operations.**

---

## Monthly Operating Rhythm

```
Week 1: Monitor for OS security updates
        ↓ (if updates available)
Week 2: Run Phase 1 workflow on DEV → Human Approval
        ↓
Week 3: Deploy to production (rolling)
        ↓
Week 4: Post-deployment monitoring & final baseline
```

**Estimated Monthly Time Investment:**
- Planning & initiation: 30 minutes
- Review & approval: 1-2 hours  
- Monitoring & verification: 1 hour
- **Total: ~3-4 hours per month**

---

## Key Differentiators vs. Manual Process

| Aspect | Manual Process | Automated System |
|--------|---------------|------------------|
| Testing before production | Limited or skipped | Comprehensive, every time |
| Change detection | Manual log review | Automated comparison |
| Test coverage | Ad-hoc | Standardized test suite |
| Approval documentation | Informal or missing | Formal approval document |
| Rollback capability | Manual, slow | Automated, < 5 minutes |
| Time per update | 8-16 hours | 2-3 hours |
| Error rate | Higher (human factors) | Lower (consistent process) |
| Audit trail | Incomplete | Complete |

---

## Business Value Summary

### Cost Savings
- **Reduced labor:** 75% reduction in time per update cycle
- **Reduced risk:** Near-zero production incidents from security updates
- **Faster response:** Emergency patches deployed in hours, not days

### Operational Excellence
- **Consistency:** Every update follows the same rigorous process
- **Quality:** Automated testing catches issues before production
- **Scalability:** Same process works for 3 sites or 300 sites

### Compliance & Security
- **Audit readiness:** Complete documentation of all changes
- **Security posture:** Profiles always current with OS versions
- **Risk management:** Human approval for all production changes

---

## Approval Workflow Example

**Typical Regular Update Approval Document Includes:**

```
APPARMOR PROFILE UPDATE APPROVAL REQUEST

Update Type: Regular OS Security Patches
Date: 2025-11-21
Target: Production Fleet (3 Raspberry Pi sites)

SUMMARY OF CHANGES:
- OS updated from version 11.8 to 11.9
- 6 new file paths detected (system libraries)
- 0 denied accesses
- 0 removed paths

DEV TESTING RESULTS:
✅ All HTTP functionality tests passed
✅ Apache service started successfully
✅ No security denials during testing
✅ Profile reloaded without errors

RISK ASSESSMENT: LOW
- All changes are expected for this OS version
- DEV testing completed successfully
- No application behavior changes

RECOMMENDATION: APPROVE for production deployment

APPROVER: _________________  DATE: __________
```

---

## Questions & Answers

**Q: Will this system collect data from client production sites?**  
A: No. The system does NOT continuously monitor or collect data from production sites. Production sites operate independently. The system only accesses production during planned update windows.

**Q: What happens if we skip an update cycle?**  
A: The system is flexible. You can skip monthly updates if needed. The next time you run an update, the system will detect all accumulated changes.

**Q: Can we test the system before deploying to production?**  
A: Yes. The DEV machine exists specifically for this purpose. Every change is tested on DEV before production.

**Q: How long does a rollback take?**  
A: Automated rollback takes < 5 minutes per site. The system maintains backups of all previous profiles.

**Q: Who needs to approve production changes?**  
A: The designated system administrator or on-call engineer. Approval is based on the generated approval document showing DEV test results.

**Q: What if an emergency update can't be tested on DEV?**  
A: In extreme cases, Phase 2 allows direct-to-production deployment with a staged rollout (canary testing on one site first). This is rare and requires explicit approval of the elevated risk.

---

## Next Steps

For detailed technical implementation and command references, see:
- `COMPLETE_WORKFLOW.md` - Detailed technical procedures
- `WORK_SUMMARY.md` - Technical implementation summary
- `FLOWCHARTS.md` - Visual workflow diagrams

---

**Document End**

For questions or clarifications, contact the System Administration team.
