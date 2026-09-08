# Incident report — production host compromise, 09/2025

>**AI assistance note**:  > - AI assistance was used in the _editing_ of this report.  All of the content, data and history are mine; AI was used as a resource for edits.
---

## 0 ·  Provenance

Disclosures:

- This is a retrospective review of an actual incident from a production server.  This is recreated from verifiable facts when possible and memory when data is absent.  The host was wiped during remediation so no logs exist.
- The timeline is reconstructed from data from 3rd parties.  The dates are verified directly but parts may be withheld for privacy reasons.
- Everything is anonymized. While the production environment is no longer active, privacy still applies.
- The method of entry to server is suspected but never verified.


---

## 1 · Environment and posture *before* the incident

The box:

- A VPS running Ubuntu 22.04
  - 4 vCPU, 8 GB RAM, 100 GB storage.
  - running a LAMP stack to host WordPress
  - maintained by me without support.

Maintenance and Patching:

I did maintenance on an ad hoc basis.  Instead of setting a regular cadence for updates (monthly, etc) I did them whenever I happened to log in to the SSH or the WordPress admin side.  I chose to keep some updates slightly behind the current version to maintain compatibility - specifically, PHP because WordPress rarely supported the most current version.

> I had tried to update it to 24.04 but it failed.  I decided that 22.04 was an LTS version and acceptable if it supported the applications I needed.

Monitoring:

I had no active monitoring.  The only thing I had that was close to monitoring was an automation that made a daily social media post which provided a "heartbeat" of sorts which, in hindsight, turned out to be the only signal I had.

> Note: this server was not actively maintained at the time of the incident.  The business had gone dormant and the server was left unattended.

Architectural choices:

- All payment handling was done off-site.  I deliberately made this choice to minimize the risk of customer financial information being compromised in the event of a breach.

---

## 2 · Detection — the absence of a signal *was* the signal

Without any monitoring system, I first noticed that anything was wrong when social media engagement changed.  Daily posts to x.com had a regular cadence of likes/reposts/comments and I noticed that I hadn't gotten any notifications from x.com

I had included Instagram in my autoposting but it failed regularly so I didn't pay much attention to it as an Instagram failure was as much noise as a signal that something was wrong.

I checked the social media account to see if it was a normal "miss" or if posts were even being made.  I noticed that nothing had been posted in 2 days, so I assumed that something in the automation had failed.

I checked the website and got a server timeout and pages failing to load.

I logged into SSH and found the server was slow to respond.

I restarted the server but that didn't solve the issues. I pinged the domain and got a response from the server with the correct IP address.  So I had confirmed that the domain registration and DNS records were working.

---

## 3 · Scope assessment — the decision point

I had confirmed that the DNS and registration were working so I assumed the problem was with the server itself.

I began digging into the WordPress folders for the site and found multiple files had filename changes.  A significant number of files had been modified across multiple folders including the root of the WordPress install.

As the server also held separate WordPress installs for other domains, I checked the other WordPress installs and found similar filename changes.

At the time, I suspected that the attackers had managed to traverse up multiple levels and into other WordPress folders.

> However, looking back now, this is not the only possibility.  The various WordPress installs were all running the same version of WordPress and likely had very similar plugin sets so they all likely had similar vulnerabilities.

---

## 4 · Containment

I made the decision to take the server offline and shut it down until deeper work could be done.

When the server was brought back on line, I modified the UFW to block all traffic in and out except the port that was used for SSH - including 80/443. This gave me a server that was still compromised but reduced the risk of the attackers regaining access or a dormant payload being distributed.

As this was a server for an inactive company and hadn't been maintained for some period of time, I was not concerned about availability.

---

## 5 · Eradication — and why I rejected the restore

I made the decision to completely wipe the server and rebuild from scratch. Before I did that, I made a backup of the MySQL databases and downloaded it.

The reason I decided to wipe the server and rebuild was the list of unknowns:

- unknown attack vector
- unknown start of the attack
- unknown extent of the attack
  - Compromised *application* or compromised *server*?
- backups that could not be trusted because of the unknowns.

---

## 6 · Selective data salvage

I rebuilt the content from the MySQL database downloaded earlier.  However, instead of restoring the entire database, I only restored the content tables.

Restoring other tables would have opened up old vulnerabilities *e.g.* the user table could have unknown users with escalated privileges.

---

## 7 · Recovery

I rebuilt the server from a fresh install of Ubuntu 22.04 and rebuilt the LAMP stack from known good sources.

I hardened the server with passwordless SSH key-only logins and remote root logins disabled.  I moved SSH to a nonstandard port as well.

I limited UFW access to HTTP traffic and SSH. All other ports were blocked.

I set up fresh WordPress installs and turned them on.  I deliberately reduced the WordPress plugin sets and old unsupported plugins were left uninstalled.  *(This was the suspected attack vector.)*

I issued fresh SSL certificates for each domain.

I chose this path to give me a server that was unlikely to be immediately reinfected or compromised.

I completed this process over approximately 2 weeks alongside full time employment.

---

## 8 · Corrective actions

I brought all of the WordPress installs current and installed the minimum of plugins that would do what was needed.  I made a judgement call on any plugins that had not been updated within 6 months and tried to replace them with a different plugin that was actively maintained.

---

## 9 · Lessons learned

**An unmaintained server is vulnerable.**

- The site had run without interruption for 10 years prior to this incident.  While the server was patched and updated at an unscheduled cadence, it was enough to keep the server running.
- Once the server went unmaintained for a period of time, the vulnerabilities were magnified and grew across a broader surface.

**Backups are only backups if they can be trusted.**

- I used the Duplicator plugin for backups which were then held off-site in a Google Drive assuming that *a* backup was good enough.
- However, a plugin in a compromised WordPress server can't be trusted.  If the source and creator of the backups is compromised, everything is compromised.
- An attacker with access across the `www` tree could have had access to the backups as well.

**The attack vector was (and still is) unknown.**

- At the time I assumed that the attacker had traversed the `www` tree since multiple installs were all attacked.
- However, a simpler option is that all the WordPress installs had the same vulnerability because they were all similarly built.
- The attacker may not have done more than exploit the same vulnerability across everything without access beyond the WordPress installs.

**Risk management reduces what's in the blast radius.**

- Early in the server's history I made a choice to leave all payment handling off-site and in the hands of the payment processors.
- I made this choice intentionally to reduce the collateral damage in the event of a breach.

**Destroying evidence makes reconstruction harder.**

- Wiping the server before downloading the logs made it impossible to identify the vector or vulnerability.  Today I'd copy the logs before wiping.

Recent changes:

- I installed a Wazuh agent (with AI assistance) on the server for monitoring and logging events.

---

## 10 · Derived timeline

| Marker | Date | What it actually marks |
|---|---|---|
| Last automated post | **09/23/2025** | When the site stopped functioning well enough to post.  **Not when access began.** |
| Automated posting re-enabled | **10/31/2025** | When I re-enabled posting after the rebuild was stable. **An upper bound on recovery completion, not the moment service returned.** |

The *outage* lasted ≤ 38 days.  A longer time than I would normally want, but the dates are markers that don't tell the whole story.

1. I only know that on 9/23 the server stopped working *enough* to do the daily social media post.
2. 9/23 may not have been the start of the attack.  The actual date of compromise is unknown but must have been on or before 9/23.  It might have been the day before or it might have been 6 months before.  I just can't know.  This reinforced my choice to rebuild the server from scratch.
3. 10/31 was when social media posting resumed. The server had resumed service prior to that but I held social media posting until I was satisfied that it was stable.
4. This window of time comfortably fits my recollection of the shutdown period as well as the rebuild time.

The daily social media posting acted as a heartbeat of sorts but it was not monitoring.  The stopped heartbeat led to identifying more symptoms but it was no substitute for monitoring. **And this heartbeat is the only objective data I have for this incident**.

---

## 11 · Limitations of this report

This report is based on my memory of events except where 3rd party data can establish facts - e.g. social media posting gap establishes the minimal bounds of the incident.

The timeline was verified against an external source.  This establishes 2 dates and nothing more.  The server may have been compromised before the first date, and the server was restored prior to the second date.

There are no host artifacts.  The wiping of the server and subsequent restoration eliminated all logs.

Without any logs or artifacts from the compromised state, it is impossible to establish what the attack vector was.

This was a single operator environment.  Without a team, there are no peers to provide context, details, facts, or support any claims.

I cannot share screenshots or other proof due to privacy concerns.