# Automation-with-Python

The listed project in this module covers:

---

### **1. EC2 Health & House-Keeping**  
| Task | What the script does |
|------|----------------------|
| **Status Checks** | Prints state & reachability of **all EC2s** in a region. |
| **Scheduled Checks** | Runs above script via **cron / Lambda** and logs results. |
| **Tagging** | Adds `Environment: prod|dev|test` tags to every instance automatically. |

---

### **2. Backup & Restore Lifecycle**  
| Step | Automation |
|------|------------|
| **Backup** | **Create snapshots** only for **production** EC2 volumes nightly. |
| **Cleanup** | Keep **latest 2 snapshots** per volume; delete older ones. |
| **Restore** | Spin up **new volume from snapshot**, attach to **existing EC2**. |

---

### **3. Website Monitoring & Self-Healing**  
| Component | Implementation |
|-----------|----------------|
| **Health-check** | Python script hits site every minute via `requests`. |
| **Alerting** | Sends **Gmail SMTP alert** on non-200 response or connection error. |
| **Recovery** | Uses **Paramiko** to SSH into server and **restart nginx container**; falls back to **Linode API reboot** if needed. |
| **Scheduling** | Runs as **cron job** for 24×7 watch.

---

### **4. Error Handling & Logging**  
- Added **try/except** blocks around all AWS calls.  
- **Logged** successes & failures to local file for audit trails.
