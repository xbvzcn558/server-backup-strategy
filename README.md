# server backup: A Practical Guide to Strategy, Tools, and Offsite Storage Costs for Dedicated and Cloud Servers

If you're searching "server backup," you're probably in one of two situations: you've just realized your dedicated or cloud server has no real backup, or you have something called a backup but no idea whether it would actually survive a bad night. Both are common. What's less common is a setup that's been tested all the way through to a restore.

This guide covers the parts that matter: what a server backup has to protect against, the 3-2-1 strategy, which backup type fits your workload, the tools people actually run on Linux and Windows servers, and — the part most guides skip — what the offsite copy costs at real, published rates, including the backup services from hosting provider Sharktech.

## What a Server Backup Actually Has to Survive

Before picking tools, it helps to know what you're defending against. In practice, servers lose data in a handful of predictable ways:

- **Hardware failure.** Drives die. On a dedicated server, that's your problem; on a cloud instance, it's the provider's problem — until their redundancy doesn't cover the exact failure that happened.
- **Ransomware and cyberattacks.** These don't just encrypt files; modern strains target backup repositories and admin accounts first, precisely so you can't recover cheaply.
- **Human error.** A mistyped `rm -rf`, a dropped production table, a config overwrite at 2 a.m. This is the most common cause of data loss, and no amount of RAID fixes it.
- **Provider or datacenter events.** Outages happen even at good providers. If your only copy lives on the same infrastructure, you're exposed to the same blast radius.

The common thread: a copy on the same machine, or even on a second machine in the same rack, protects you from almost none of these. Which leads to the strategy that's been standard advice for decades for a reason.

## The 3-2-1 Rule: Still the Best Starting Point

The 3-2-1 rule, as documented by backup vendors like Veeam and repeated across essentially every credible guide on the subject, means:

- **3** copies of your data (the working copy plus two backups)
- **2** different types of media (e.g., local disk plus offsite storage)
- **1** copy stored offsite

> A backup copied to a different disk on the same server is not offsite. A backup on another server in the same rack is barely offsite. The offsite copy is the one that saves you when the building, the provider, or the ransomware operator takes everything else down.

Most server admins get the first two numbers done easily. The third number — where to put the offsite copy, and what it costs every month — is where plans stall. That's the part worth pricing out carefully, and we'll get to real numbers shortly.

## Full, Incremental, or Differential: Picking a Backup Type

Every backup job is some mix of three basic types, and the tradeoff is always the same: backup speed and storage cost on one side, restore speed on the other.

| Backup type | How it works | Backup speed | Storage use | Restore time |
| --- | --- | --- | --- | --- |
| **Full** | Copies everything, every run | Slowest | Highest | Fastest — one dataset, done |
| **Incremental** | Copies only changes since the last backup of any type | Fastest, smallest | Lowest | Slowest — needs the full plus the entire chain |
| **Differential** | Copies all changes since the last full | Middle, grows over time | Middle | Middle — full plus one differential |

The classic mistake is optimizing only for backup speed. Incrementals are cheap to run, but if your chain has 30 links and one is corrupted, the restore stops there. As AWS's own backup documentation puts it, the differential strategy exists specifically to trade a bit of storage for faster, more robust restores.

A reasonable default for a small or medium server: a weekly full backup plus daily incrementals — or, if you use a deduplicating tool like BorgBackup or Restic, the full-vs-incremental distinction mostly disappears, since those tools only store changed blocks anyway.

## File-Level, Disk Image, or Agent-Based: Match the Method to the Server

The next decision is scope. There are three practical approaches, and they differ in what you get back after a disaster:

1. **File-level backup** (rsync, Borg, Restic, etc.). You back up directories, databases dumps, and configs. Flexible and cheap, but after a server failure you're rebuilding the OS, packages, and configuration yourself before your data is useful again.
2. **Disk image / snapshot backup.** You capture the whole volume. Restore is close to boot-and-go. On the hosting side, this is increasingly built in — Sharktech's cloud platform, for example, lets customers download their server disk images at any time for offsite backup or disaster recovery, and its Private Cloud product includes one-click volume snapshots.
3. **Agent-based backup with a management layer.** Software installed on the server handles scheduling, encryption, deduplication, and restore, and you manage it from a console. This is where products like Acronis Cyber Protect sit, and it's the category most "just get it done" operators eventually land on.

None of these is universally right. A single-purpose web server with a documented config might be perfectly served by file-level backup. A messy server with years of accumulated tweaks is much safer with an image or agent-based approach, because nobody remembers how it was configured.

## The Tools People Actually Run on Linux Servers

If you search backup tool discussions among Linux admins (Reddit's r/sysadmin and r/linuxadmin threads are full of them), the same names keep coming up:

- **rsync** — the universal baseline; not a backup system by itself, but the transport layer under half of everything else
- **BorgBackup / Restic** — deduplicating, encrypted repositories; the modern favorites for set-and-forget
- **Duplicity / Duplicati** — encrypted, bandwidth-efficient, popular for pushing to remote targets
- **Bacula / Amanda** — the old-school enterprise frameworks for fleets of servers

The relevant detail for this guide: most of the modern tools (Restic, Duplicity, Duplicati, Borg via wrappers) can write directly to S3-compatible object storage as their offsite target. That means your offsite copy in a 3-2-1 setup can be plain, cheap storage — you don't need an expensive "backup product" on the destination side.

## Where to Put the Offsite Copy — and What It Costs

This is where budgets get made or broken. Published rates from major providers, for context:

- **AWS S3 Standard**: $0.023 per GB for the first 50 TB — roughly **$23/TB/month** before any egress charges
- **Backblaze B2**: **$6.95/TB/month** with free egress up to 3x storage
- **Enterprise backup software** (the Veeam-class tier): industry analyses put per-TB costs anywhere from **$300 to $2,500 per year** once licensing is counted

Sharktech — a hosting provider that's been around for 20 years, running DDoS-protected infrastructure across five data centers in Los Angeles, Las Vegas, Denver, Chicago, and Amsterdam — currently sells two things that fit directly into this slot, at rates worth comparing against the above.

**Option 1: Acronis Cloud Backup (managed).** For $4.00/month you get 200 GB of managed cloud backup storage with Acronis Cyber Protect — encryption, deduplication, ransomware protection and automated threat detection included, supporting Windows, Linux, and macOS on physical, virtual, and cloud servers. Restore can be per-file or whole-system. Beyond 200 GB, additional capacity is billed per GB.

**Option 2: S3 Object Storage (DIY target).** A flat-rate S3-compatible bucket service. The product page advertises **$4.90/TB** storage with 1 TB of bandwidth included; the current order portal lists the entry configuration from **$6.00/month** for 1 TB. Bandwidth scales from 1 TB up to 1 PB, and since it's S3 API-compatible, Restic, Duplicity, Duplicati, and the usual DevOps tooling (Jenkins, GitLab, Terraform) all speak to it natively.

The choice between them comes down to who does the work. Acronis is the "the software handles it" path — agent, scheduler, encryption, anti-ransomware, one console. S3 is the "I already have Restic/Borg, I just need somewhere cheap and reliable to point it" path.

## Sharktech's Backup Plans: The Full Lineup

The table below covers both backup-related offerings as currently listed in Sharktech's order portal and on their published Acronis service pricing:

| Plan | Core configuration | Price | Billing cycle | Order |
| --- | --- | --- | --- | --- |
| **Acronis Cloud Backup** | 200 GB managed cloud backup (Acronis Cyber Protect: encryption, dedup, anti-ransomware); extra GB at $0.02 | $4.00/month | Monthly | [ Order Acronis Cloud Backup](https://portal.sharktech.net/aff.php?aff=1611&gid=109) |
| **Acronis Cloud Backup** | 200 GB; extra GB at $0.04 | $8.00 per 3 months | Quarterly | [ Order Acronis Cloud Backup](https://portal.sharktech.net/aff.php?aff=1611&gid=109) |
| **Acronis Cloud Backup** | 200 GB; extra GB at $0.06 | $12.00 per 6 months | Semi-annual | [ Order Acronis Cloud Backup](https://portal.sharktech.net/aff.php?aff=1611&gid=109) |
| **Acronis Cloud Backup** | 200 GB; extra GB at $0.12 | $24.00/year | Annual | [ Order Acronis Cloud Backup](https://portal.sharktech.net/aff.php?aff=1611&gid=109) |
| **Object Storage (S3)** | 1 TB storage + 1 TB bandwidth (scalable to 1 PB); S3 API-compatible; all 5 datacenter locations | from $6.00/month (product page advertises flat $4.90/TB) | Monthly | [ Order S3 Object Storage](https://portal.sharktech.net/aff=1611&gid=105) |

A few notes that affect the math:

- **The monthly figure is the one cross-verified on the current order portal.** The quarterly, semi-annual, and annual rates come from Sharktech's published Acronis service pricing. On base storage alone, annual billing ($24/year) works out to half the monthly rate ($48/year) for the same 200 GB — but the published per-GB rate for capacity beyond 200 GB rises with longer cycles, so if you'll overflow the base allotment regularly, monthly billing's $0.02/GB is the friendlier overflow rate.
- **Files Sync & Share is an optional add-on** on the Acronis plans (0 GB up to 100 TB per the order portal), published from $0.03/GB/month if you want file syncing alongside backup. Skip it if you only need backup.
- Acronis Cloud Backup is available across all five Sharktech datacenter locations.
- Sharktech's support is 24/7 with phone access, and their cloud services carry a 99.999% uptime guarantee — relevant because a backup target that's down when your job fires is a silent failure.

If you already know your data size, you can check the current configurations and exact order-form pricing directly: [👉 View Acronis Cloud Backup plans](https://portal.sharktech.net/aff.php?aff=1611&gid=109) or [👉 Check S3 Object Storage pricing](https://portal.sharktech.net/aff.php?aff=1611&gid=105).

## What a Real Setup Costs: Two Worked Examples

Abstract pricing is fine; concrete numbers are better. Two common server profiles:

**A 200–500 GB application server.** Base Acronis plan at $4.00/month covers 200 GB. At the published $0.02/GB rate for monthly billing, a 500 GB server pays roughly $4.00 + (300 × $0.02) = **$10/month** for managed, encrypted, anti-ransomware backup with whole-system restore capability. Compare that against rebuilding a production app server from memory.

**A 2 TB media or archive server.** With S3 object storage at the advertised $4.90/TB flat rate, 2 TB runs about **$9.80/month** as the offsite leg of a Restic or Duplicity setup — the tooling is free, the storage is the cost. The exact figure for your configuration shows on the order form, since bandwidth selection (1 TB–1 PB) affects the final price.

Both examples assume you keep a second copy somewhere local — a second disk, a NAS — which is usually the cheap half of 3-2-1. The offsite leg is the part you rent, and at these rates it's the cheapest insurance line item most server operators have.

## A Setup Checklist That Avoids the Classic Mistakes

1. **Define what you actually lose.** Decide your recovery point objective — how many hours of data you can afford to lose — before choosing a schedule. Most small servers want daily; transaction-heavy apps want more frequent.
2. **Pick the scope honestly.** If rebuilding the OS and config from scratch would take a weekend, use image or agent-based backup. If it would take twenty minutes because everything is in Ansible, file-level is fine.
3. **Automate on a schedule.** Manual backups get skipped exactly when everyone is busiest. Every tool named above supports cron or its own scheduler; Acronis handles scheduling in the agent.
4. **Encrypt the offsite copy.** Non-negotiable if the data touches customer information. Restic, Borg, Duplicity, and Acronis all do this natively.
5. **Keep version history, not just the latest copy.** Ransomware encrypting today's file means you need last week's. Retention policies exist for this reason.
6. **Test a full restore on a schedule.** Quarterly is a common cadence. A backup that has never been restored is a superstition, not a strategy.
7. **Monitor job logs.** Silent failures — full disks, expired credentials, a storage endpoint that moved — are the number-one way backups die quietly for months.

## Straight Answers to Common Questions

**Is RAID a backup?** No. RAID protects against a disk failing; it does nothing against deletion, ransomware, or a bad `rm`. Every backup guide says this because every year someone learns it the hard way.

**How often should a server back up?** Match it to how much data loss you can absorb. Daily is the sensible floor for most servers; databases handling transactions often warrant hourly or continuous replication on top of a daily backup.

**Do I need to back up the entire OS?** Not strictly — but weigh the rebuild time honestly. Configs, package lists, and database dumps cover a disciplined setup; images or agents cover everything else, including the things you forgot were configured.

**How long should I keep backups?** Long enough to survive a delayed ransomware discovery. If your only copies are from the last 48 hours and the attack started five days ago, the backups are encrypted too. Weeks-to-months of version history is the practical answer.

## The Bottom Line

A server backup that works comes down to three decisions: what to capture, how often, and where the offsite copy lives. The first two are engineering choices you make once and automate. The third is a running cost — and at hyperscaler rates ($23/TB at AWS before egress), it's often the line item that keeps people from actually finishing their 3-2-1 setup.

Sharktech's two offerings bracket the market nicely on that last point: a managed, agent-based option with anti-ransomware features starting at **$4.00/month for 200 GB**, and a flat-rate S3-compatible storage target for DIY tooling like Restic and Borg, advertised at **$4.90/TB** with the entry package listed from $6.00/month in the portal. Either one completes the offsite leg of a 3-2-1 setup for less than the cost of a decent lunch.

If the managed route fits your situation — you'd rather not hand-roll encryption, scheduling, and restore testing — the Acronis plans are the fastest path from "no backup" to "done": [👉 Get started with Acronis Cloud Backup from $4/month](https://portal.sharktech.net/aff.php?aff=1611&gid=109). If you already run Restic or Duplicity and just need a cheap, redundant offsite target across five datacenters: [👉 Set up S3 Object Storage for your backups](https://portal.sharktech.net/aff.php?aff=1611&gid=105).

Whichever you choose, do the one thing most people skip: run a test restore this month. That's the moment a backup stops being a plan and starts being a fact.
