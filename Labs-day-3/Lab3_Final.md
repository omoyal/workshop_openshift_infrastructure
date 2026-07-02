## 🎁 The Deliverables: Your Day-One Survival Kit

The exercise you just finished wrote three files for you. Fill in the blanks, commit them to your team's Git — this is what you'll actually open next week.

---

<br><br>

### 📘 File 1: `onboarding-Team.md` — when a new team arrives

```markdown
# Team Onboarding — <company>
OpenShift 4.20, <date>_

## Inputs
Team: ____  |  Lead (admin): ____  |  Members (edit): ____
Budget: CPU ____ / Mem ____ / Pods ____

## Steps
⬜ Users in IdP + group <team>-devs
⬜ Project + display name
⬜ RBAC: edit→group, admin→lead
⬜ Quota + LimitRange + NetworkPolicy pack
⬜ Smoke test AS THE USER: login → deploy → route → browser
⬜ Show the team: Topology, quota gauges, Observe → Dashboards
```

---
<br><br>


### ⚡ File 2: `cheatsheet.md` — the 20 commands you'll actually type

```markdown
# OpenShift Day-to-Day — Cheat Sheet


## Is the cluster healthy? (run this FIRST, always)
oc get co | grep -v 'True.*False.*False'    # empty output = healthy
oc get nodes
oc get clusterversion



## Why is this pod broken? (in order)
oc get pods                                  # STATUS + RESTARTS
oc describe pod <pod>                        # read Events at the bottom
oc logs <pod> [--previous]                   # --previous = the crashed one
oc get events --sort-by=.lastTimestamp | tail -20



## The classics — decoded
CrashLoopBackOff   → oc logs --previous      (app crashes on start)
ImagePullBackOff   → oc describe pod          (image name/registry/secret)
Pending            → oc describe pod          (quota? taint? no resources?)
Permission denied  → SCC. Check:
  oc get pod <pod> -o jsonpath='{.metadata.annotations.openshift\.io/scc}'



## Why can't user X do Y? (answers 90% of tickets)
oc auth can-i <verb> <resource> -n <ns> --as=<user>
oc get rolebindings -n <ns>
oc describe quota -n <ns>



## Node needs maintenance
oc adm cordon <node>
oc adm drain <node> --ignore-daemonsets --delete-emptydir-data
oc adm uncordon <node>



## Break glass
oc debug node/<node>   →   chroot /host      # root shell, audited
oc adm must-gather                            # before opening a case
```

---

<br><br>

### 🚑 File 3: `first-15-minutes.md` — when something is on fire

```markdown
# Incident: First 15 Minutes
_Print me. Tape me to the wall._

<br>

1. PLATFORM OR APP?
   oc get co | grep -v 'True.*False.*False'
   → co Degraded?  PLATFORM issue — read its message, that's your lead
   → all clean?    APP issue — go to 2

<br>

2. WHERE DOES IT HURT?
   oc get pods -n <ns> -o wide     # note the NODE column
   → one pod?      describe + logs (see cheatsheet)
   → many pods, same node?  NODE issue — oc describe node <node>
   → many pods, all nodes?  quota / config / image — check events

<br>

3. WHAT CHANGED?
   oc get events -A --sort-by=.lastTimestamp | tail -30
   Ask the humans: deploy? upgrade? config change?

<br>

4. STOP THE BLEEDING (in order of preference)
   oc rollout undo deployment/<app> -n <ns>    # bad deploy → rollback
   oc scale / cordon                           # contain it
   ...only THEN root-cause in peace.

<br>

5. IF YOU CALL RED HAT
   oc adm must-gather                          # attach the tarball
   Have ready: cluster version, what changed, when it started
```

---

✅ **Final checkpoint of the workshop:** all three files filled in and committed to your Git. Onboarding, daily driving, and firefighting — covered.
