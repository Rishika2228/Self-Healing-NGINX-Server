Yep — here is the **entire README in one single code block** so you can copy it all at once and paste directly into `README.md`.

```markdown
# 🔄 Self-Healing NGINX Server

> A DevOps project that detects an NGINX failure, triggers an alert, and automatically recovers the server without manual intervention.

---

## 📌 About the Project

What if a web server could notice that something went wrong and fix itself?

That was the main idea behind this project.

For this project, I built a **self-healing web server environment on AWS EC2** where NGINX is continuously monitored. If NGINX stops responding, the monitoring system detects the failure and automatically starts a recovery process.

Instead of manually logging into the server and running:

```bash
sudo systemctl restart nginx
```

the system handles the recovery automatically.

The complete flow is:

```text
NGINX Failure
     ↓
Blackbox Exporter detects failure
     ↓
Prometheus detects the failed probe
     ↓
Prometheus fires NginxDown alert
     ↓
Alertmanager receives the alert
     ↓
Alertmanager sends webhook
     ↓
Flask Webhook receives the alert
     ↓
Flask triggers Ansible
     ↓
Ansible restarts NGINX
     ↓
Ansible verifies HTTP 200
     ↓
NGINX is healthy again
```

The project was built mainly as a **hands-on DevOps learning project** to understand how monitoring, alerting, automation, and recovery can be connected together.

---

# 🎯 Project Goal

The main goal was to build a simple but practical **self-healing infrastructure workflow**.

I wanted the system to be able to:

- Monitor the web server continuously
- Detect when NGINX becomes unavailable
- Generate an alert automatically
- Send the alert to a custom webhook
- Trigger an automated recovery action
- Restart NGINX
- Verify that NGINX is working again
- Avoid manual intervention during the recovery process

In short:

> **Detect the problem → trigger automation → fix the problem → verify the recovery.**

---

# 🏗️ Architecture

The complete project runs on a single AWS EC2 instance.

```text
                         AWS EC2
                    Ubuntu 24.04 LTS
                           │
                           ▼
                      ┌─────────┐
                      │  NGINX  │
                      │  :80    │
                      └────┬────┘
                           │
                    HTTP Health Check
                           │
                           ▼
                ┌─────────────────────┐
                │  Blackbox Exporter  │
                │       :9115         │
                └──────────┬──────────┘
                           │
                           ▼
                    ┌─────────────┐
                    │  Prometheus │
                    │    :9090    │
                    └──────┬──────┘
                           │
                     NginxDown Alert
                           │
                           ▼
                    ┌─────────────┐
                    │ Alertmanager│
                    │    :9093    │
                    └──────┬──────┘
                           │
                         Webhook
                           │
                           ▼
                    ┌─────────────┐
                    │    Flask    │
                    │    :5000    │
                    └──────┬──────┘
                           │
                     Triggers Ansible
                           │
                           ▼
                    ┌─────────────┐
                    │   Ansible   │
                    └──────┬──────┘
                           │
                    Restart NGINX
                           │
                           ▼
                    HTTP 200 Check
                           │
                           ▼
                       ✅ Healthy
```

---

# 🛠️ Technologies Used

| Technology | Purpose |
|---|---|
| AWS EC2 | Hosts the complete project |
| Ubuntu 24.04 LTS | Operating system |
| NGINX | Web server |
| Prometheus | Monitoring and alert evaluation |
| Blackbox Exporter | Checks whether NGINX is responding |
| Alertmanager | Handles and forwards Prometheus alerts |
| Flask | Receives the Alertmanager webhook |
| Ansible | Automates NGINX recovery |
| Node Exporter | Collects system-level metrics |
| systemd | Runs services automatically |

---

# ☁️ AWS Infrastructure

The project was deployed on an AWS EC2 instance.

### EC2 Configuration

```text
Operating System : Ubuntu Server 24.04 LTS
Architecture     : x86_64
Instance Type    : t3.micro
Storage          : gp3
Web Server       : NGINX
```

The project intentionally uses a small EC2 instance because the main purpose is to demonstrate the DevOps workflow rather than run a production application.

## EC2 Instance

<!-- ================================================= -->
<!-- 📸 SCREENSHOT SPACE                               -->
<!-- Put your AWS EC2 screenshot here                  -->
<!-- File: screenshots/01-ec2-instance.png             -->
<!-- ================================================= -->

![AWS EC2 Instance](screenshots/01-ec2-instance.png)

*AWS EC2 instance running the self-healing server.*

---

# 🔐 Security Group Configuration

I kept the security group relatively simple and only exposed the ports that were required.

| Port | Protocol | Purpose | Public Access |
|---|---|---|---|
| 22 | TCP | SSH | My IP only |
| 80 | TCP | NGINX HTTP | Yes |
| 9090 | TCP | Prometheus | No |
| 9093 | TCP | Alertmanager | No |
| 9100 | TCP | Node Exporter | No |
| 9115 | TCP | Blackbox Exporter | No |
| 5000 | TCP | Flask Webhook | No |

The monitoring and automation components communicate internally on the EC2 instance.

The Flask webhook is specifically bound to:

```text
127.0.0.1:5000
```

so it is not directly exposed to the internet.

---

# 📁 Project Structure

The main project files are stored under:

```text
/opt/self-healing/
```

The structure is:

```text
self-healing/
│
├── webhook.py
├── inventory
└── restart_nginx.yml
```

### `webhook.py`

The Flask application that receives alerts from Alertmanager.

### `inventory`

The Ansible inventory that defines the target server.

### `restart_nginx.yml`

The Ansible playbook responsible for restarting and verifying NGINX.

---

# 🌐 Step 1 — NGINX

NGINX is the web server that the entire self-healing workflow monitors.

First, I verified that NGINX was running correctly:

```bash
sudo systemctl status nginx --no-pager
```

A healthy service shows:

```text
Active: active (running)
```

I also checked the actual HTTP response:

```bash
curl -I http://localhost
```

The expected response is:

```text
HTTP/1.1 200 OK
```

This distinction is important.

It isn't enough for the NGINX process to simply exist. The monitoring system should also know whether the web server is actually responding to HTTP requests.

## NGINX Health Check

<!-- ================================================= -->
<!-- 📸 SCREENSHOT SPACE                               -->
<!-- Put your NGINX healthy screenshot here            -->
<!-- File: screenshots/02-nginx-healthy.png            -->
<!-- ================================================= -->

![NGINX Healthy](screenshots/02-nginx-healthy.png)

*NGINX running successfully and returning HTTP 200.*

---

# 📡 Step 2 — Blackbox Exporter

For the project, I used **Blackbox Exporter** to monitor the actual HTTP availability of NGINX.

Instead of only checking whether a process is running, Blackbox Exporter performs an HTTP probe against:

```text
http://localhost
```

The important metric is:

```text
probe_success
```

If the HTTP request succeeds:

```text
probe_success = 1
```

If the request fails:

```text
probe_success = 0
```

This makes Blackbox Exporter useful for checking whether a service is actually reachable.

---

# 📊 Step 3 — Prometheus

Prometheus continuously collects the metrics generated by Blackbox Exporter.

The Blackbox scrape configuration looks like:

```yaml
- job_name: 'blackbox'

  metrics_path: /probe

  params:
    module: [http_2xx]

  static_configs:
    - targets:
      - http://localhost

  relabel_configs:

    - source_labels: [__address__]
      target_label: __param_target

    - source_labels: [__param_target]
      target_label: instance

    - target_label: __address__
      replacement: localhost:9115
```

Prometheus uses this information to continuously check the health of NGINX.

---

# 🚨 Step 4 — NGINX Alert

I created a Prometheus alert specifically for the NGINX health check.

```yaml
groups:

  - name: nginx-alerts

    rules:

      - alert: NginxDown

        expr: probe_success{instance="http://localhost"} == 0

        for: 30s

        labels:
          severity: critical

        annotations:
          summary: "NGINX is down"

          description: "NGINX is not responding to HTTP health checks."
```

The important part is:

```yaml
for: 30s
```

This means Prometheus doesn't immediately trigger the recovery process because of a very short interruption.

The health check needs to remain failed for 30 seconds before the alert becomes active.

The alert is called:

```text
NginxDown
```

---

# 🔔 Step 5 — Alertmanager

Once Prometheus fires the `NginxDown` alert, it sends the alert to Alertmanager.

Alertmanager is responsible for handling the alert and forwarding it to the Flask webhook.

The receiver configuration is:

```yaml
global:

route:
  receiver: 'self-healing-webhook'

receivers:

  - name: 'self-healing-webhook'

    webhook_configs:

      - url: 'http://127.0.0.1:5000/alert'

        send_resolved: true
```

So the communication becomes:

```text
Prometheus
    ↓
Alertmanager
    ↓
Flask Webhook
```

---

# 🐍 Step 6 — Flask Webhook

The Flask application acts as the bridge between Alertmanager and Ansible.

When Alertmanager sends the alert, Flask receives it at:

```text
POST /alert
```

The application checks the alert name.

If the alert is:

```text
NginxDown
```

Flask starts the Ansible recovery playbook.

The important part of the Python logic is:

```python
if alertname == "NginxDown":

    result = subprocess.run(
        [
            "ansible-playbook",
            "-i",
            "/opt/self-healing/inventory",
            "/opt/self-healing/restart_nginx.yml"
        ],
        capture_output=True,
        text=True
    )
```

The Flask application also has a simple health endpoint:

```text
GET /health
```

which can be checked using:

```bash
curl http://127.0.0.1:5000/health
```

Expected response:

```text
Webhook is healthy
```

---

# ⚙️ Step 7 — Running Flask with systemd

Initially, the Flask application was tested manually.

After confirming that the webhook worked, I configured it as a `systemd` service.

Service file:

```text
/etc/systemd/system/self-healing-webhook.service
```

Configuration:

```ini
[Unit]
Description=Self-Healing NGINX Flask Webhook
After=network.target

[Service]
User=root
WorkingDirectory=/opt/self-healing
ExecStart=/usr/bin/python3 /opt/self-healing/webhook.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

After creating the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable self-healing-webhook
sudo systemctl start self-healing-webhook
```

The service can then be checked using:

```bash
sudo systemctl status self-healing-webhook --no-pager
```

The important result is:

```text
Active: active (running)
```

## Flask Webhook Service

<!-- ================================================= -->
<!-- 📸 SCREENSHOT SPACE                               -->
<!-- Put your Flask systemd screenshot here            -->
<!-- File: screenshots/03-webhook-service.png          -->
<!-- ================================================= -->

![Flask Webhook Service](screenshots/03-webhook-service.png)

*Flask webhook running continuously as a systemd service.*

---

# 🤖 Step 8 — Ansible Recovery

Ansible is responsible for actually fixing the problem.

The inventory contains:

```ini
[webserver]
localhost ansible_connection=local
```

Since everything is running on the same EC2 instance, Ansible connects locally.

The recovery playbook is:

```yaml
---
- name: Self Heal NGINX

  hosts: webserver

  become: true

  tasks:

    - name: Restart NGINX

      ansible.builtin.service:
        name: nginx
        state: restarted

    - name: Verify NGINX

      ansible.builtin.uri:
        url: http://localhost
        status_code: 200

    - name: Display recovery message

      ansible.builtin.debug:
        msg: "NGINX successfully recovered."
```

There are two important actions here.

### 1. Restart NGINX

```yaml
state: restarted
```

This brings the service back up.

### 2. Verify NGINX

```yaml
status_code: 200
```

This makes sure the recovery actually worked.

So the automation doesn't just say:

> "I restarted NGINX."

It checks whether NGINX is actually responding.

---

# 📈 Step 9 — Node Exporter

Along with Blackbox Exporter, I also installed Node Exporter.

Node Exporter collects system-level metrics such as:

- CPU usage
- Memory usage
- Disk usage
- Filesystem information
- Network statistics
- System load

It runs on:

```text
localhost:9100
```

Prometheus collects these metrics so the project can monitor both:

```text
Application availability
        +
System health
```

This also leaves room for adding CPU, memory, and disk-based alerts later.

---

# 🧪 Testing the Self-Healing System

Now comes the most important part of the project.

Instead of simply showing that every component works individually, I tested the complete chain together.

## Step 1 — Confirm NGINX is healthy

First:

```bash
sudo systemctl status nginx --no-pager
```

Expected:

```text
Active: active (running)
```

Then:

```bash
curl -I http://localhost
```

Expected:

```text
HTTP/1.1 200 OK
```

---

## Step 2 — Simulate an NGINX failure

To simulate a real failure, I manually stopped NGINX:

```bash
sudo systemctl stop nginx
```

At this point, the web server was intentionally unavailable.

I did **not** restart it manually.

This is important because the whole purpose of the project is to test whether the automation can recover the service by itself.

---

## Step 3 — Blackbox Exporter detects the failure

Once NGINX stops responding, the Blackbox HTTP probe starts failing.

The metric changes from:

```text
probe_success = 1
```

to:

```text
probe_success = 0
```

---

## Step 4 — Prometheus triggers the alert

Prometheus sees the failed probe.

Because the alert has:

```yaml
for: 30s
```

the failure needs to continue for 30 seconds before `NginxDown` becomes active.

---

## Step 5 — Alertmanager receives the alert

Prometheus sends the active alert to Alertmanager.

Alertmanager then forwards it to:

```text
http://127.0.0.1:5000/alert
```

---

## Step 6 — Flask receives the webhook

Flask receives the alert and checks:

```text
alertname = NginxDown
```

It then launches:

```text
restart_nginx.yml
```

---

## Step 7 — Ansible restarts NGINX

Ansible executes:

```text
Restart NGINX
```

and then checks:

```text
http://localhost
```

for:

```text
HTTP 200
```

---

## Step 8 — NGINX recovers

NGINX starts running again.

The important part is:

> **I didn't manually restart NGINX after stopping it.**

The recovery was performed by the monitoring and automation pipeline.

---

# 🚀 Final Demo

This is the final end-to-end test of the project.

The complete process looks like this:

```text
                    FAILURE
                       │
                       ▼
              ┌─────────────────┐
              │  NGINX stopped  │
              └────────┬────────┘
                       │
                       ▼
              Blackbox Exporter
                       │
                 probe_success=0
                       │
                       ▼
                 Prometheus
                       │
                 NginxDown alert
                       │
                       ▼
                 Alertmanager
                       │
                    Webhook
                       │
                       ▼
                Flask /alert
                       │
                  Run Ansible
                       │
                       ▼
                Restart NGINX
                       │
                  HTTP 200
                       │
                       ▼
                    HEALTHY
```

---

## 📸 Demo Screenshot 1 — AWS EC2 Instance

The EC2 instance hosting the project is running successfully.

<!-- ===================================================== -->
<!-- 📸 DROP SCREENSHOT HERE                                -->
<!-- File: screenshots/01-ec2-instance.png                  -->
<!-- ===================================================== -->

![AWS EC2 Instance](<img width="959" height="194" alt="Screenshot 2026-09-21 175025" src="https://github.com/user-attachments/assets/1a153fbb-417d-4d50-a3df-b79158caeded" />
)

**What this screenshot shows:**  
The AWS EC2 instance is running and available. This is the machine where the complete self-healing infrastructure has been deployed.

---

## 📸 Demo Screenshot 2 — NGINX Healthy

Before starting the failure test, NGINX was confirmed to be running.

```bash
sudo systemctl status nginx --no-pager
```

and:

```bash
curl -I http://localhost
```

returned:

```text
HTTP/1.1 200 OK
```

<!-- ===================================================== -->
<!-- 📸 DROP SCREENSHOT HERE                                -->
<!-- File: screenshots/02-nginx-healthy.png                 -->
<!-- ===================================================== -->

![NGINX Healthy](<img width="959" height="422" alt="Screenshot 2026-09-21 174933" src="https://github.com/user-attachments/assets/93478e06-1b87-400d-9397-ebe0ddd2ae64" />
)

**What this screenshot shows:**  
NGINX is running normally and the local HTTP health check is returning `200 OK`.

---

## 📸 Demo Screenshot 3 — NGINX Failure and Detection

To simulate the failure:

```bash
sudo systemctl stop nginx
```

The monitoring system then detects that the HTTP endpoint is unavailable.

<!-- ===================================================== -->
<!-- 📸 DROP SCREENSHOT HERE                                -->
<!-- File: screenshots/03-nginx-failure.png                 -->
<!-- ===================================================== -->

![NGINX Failure Detection](<img width="959" height="446" alt="Screenshot 2026-09-19 213328" src="https://github.com/user-attachments/assets/f960f628-4645-4d90-a656-284ac5c5e6ce" />
)

**What this screenshot shows:**  
The failure was intentionally triggered and the monitoring/recovery pipeline started processing the alert.

---

## 📸 Demo Screenshot 4 — Flask Webhook Running

The Flask webhook is running as a systemd service.

```bash
sudo systemctl status self-healing-webhook --no-pager
```

The service reports:

```text
Active: active (running)
```

<!-- ===================================================== -->
<!-- 📸 DROP SCREENSHOT HERE                                -->
<!-- File: screenshots/04-webhook-service.png               -->
<!-- ===================================================== -->

![Flask Webhook](<img width="953" height="341" alt="Screenshot 2026-09-21 170024" src="https://github.com/user-attachments/assets/775432ef-c13b-4fc4-98ff-9db94a355ba8" />
)

**What this screenshot shows:**  
The Flask webhook is running continuously in the background and is ready to receive Alertmanager notifications.

---

## 📸 Demo Screenshot 5 — Automatic Recovery

The recovery logs show that Ansible was triggered automatically.

The important events include:

```text
Ansible restarted NGINX
```

followed by:

```text
HTTP status_code=[200]
```

and:

```text
POST /alert HTTP/1.1" 200
```

<!-- ===================================================== -->
<!-- 📸 DROP SCREENSHOT HERE                                -->
<!-- File: screenshots/05-automatic-recovery.png             -->
<!-- ===================================================== -->

![Automatic Recovery](<img width="959" height="422" alt="Screenshot 2026-09-21 174933" src="https://github.com/user-attachments/assets/5949d338-4d1d-468f-b059-c37454b084eb" />
)

**What this screenshot shows:**  
The alert reached Flask, Ansible restarted NGINX, and the recovery playbook successfully verified that NGINX was returning HTTP 200.

---

# 🖼️ Screenshot Folder Structure

Keep the screenshots inside a folder named:

```text
screenshots/
```

Recommended structure:

```text
self-healing-server/
│
├── screenshots/
│   ├── 01-ec2-instance.png
│   ├── 02-nginx-healthy.png
│   ├── 03-nginx-failure.png
│   ├── 04-webhook-service.png
│   └── 05-automatic-recovery.png
│
├── webhook.py
├── inventory
├── restart_nginx.yml
└── README.md
```

You can use different filenames if you want. Just make sure the filename in the README matches the actual image filename.

---

# ✅ Final Test Result

The final end-to-end test successfully demonstrated the complete recovery workflow.

| Test | Result |
|---|---|
| EC2 instance running | ✅ |
| NGINX initially healthy | ✅ |
| HTTP health check | ✅ |
| NGINX failure simulated | ✅ |
| Blackbox detects failure | ✅ |
| Prometheus detects failure | ✅ |
| `NginxDown` alert triggered | ✅ |
| Alertmanager forwards alert | ✅ |
| Flask receives webhook | ✅ |
| Ansible starts automatically | ✅ |
| NGINX restarted | ✅ |
| HTTP 200 verification | ✅ |
| Manual NGINX restart required | ❌ |

The most important result is that after intentionally stopping NGINX, the system was able to recover it automatically.

---

# 🔍 Logs During Recovery

One of the useful parts of the project was being able to see the recovery process directly through the system logs.

For example:

```bash
sudo journalctl -u self-healing-webhook --since "5 minutes ago" --no-pager
```

The logs showed Ansible being invoked:

```text
ansible-ansible.legacy.systemd
Invoked with name=nginx state=restarted
```

followed by the health verification:

```text
ansible-ansible.legacy.uri
Invoked with status_code=[200]
url=[http://localhost]
```

The webhook request was also successfully processed:

```text
"POST /alert HTTP/1.1" 200
```

This provided useful evidence that the recovery wasn't just a manual restart — the complete automation chain was actually being executed.

---

# 💡 Why I Built This

I built this project because I wanted to understand what happens beyond simply running a server.

In many basic projects, we start NGINX, check that the webpage works, and stop there.

But in a real DevOps environment, things can fail.

A service can crash.

A web server can become unavailable.

An application can stop responding.

The interesting part is what happens **after the failure**.

This project helped me understand the idea of connecting:

```text
Monitoring
     +
Alerting
     +
Automation
     +
Recovery
```

into one workflow.

---

# 📚 What I Learned

## 1. Prometheus Monitoring

I learned how Prometheus collects metrics and evaluates alert rules.

## 2. Blackbox Monitoring

I learned the difference between monitoring a process and monitoring whether a service is actually reachable.

Blackbox Exporter allowed me to perform a real HTTP health check against NGINX.

## 3. Alertmanager

I learned how Prometheus alerts can be routed to another system using Alertmanager.

## 4. Webhooks

Building the Flask webhook helped me understand how monitoring systems can communicate with custom applications through HTTP requests.

## 5. Ansible Automation

I learned how Ansible can be used to automate recovery tasks instead of manually running Linux commands.

## 6. systemd

I learned how to turn the Flask webhook into a persistent Linux service using systemd.

## 7. AWS EC2

I gained practical experience deploying and configuring the entire environment on an AWS EC2 instance.

## 8. Self-Healing Infrastructure

The biggest takeaway was understanding that monitoring becomes much more useful when it can trigger an automated response.

Instead of:

```text
Something broke
     ↓
Human notices
     ↓
Human logs in
     ↓
Human fixes it
```

the goal becomes:

```text
Something broke
     ↓
System notices
     ↓
System triggers automation
     ↓
System fixes it
     ↓
System verifies recovery
```

---

# 🔐 Security Notes

This project is primarily a learning/demo project and is not intended to be used as-is in production.

Some basic security decisions were still followed:

- SSH is restricted to my IP.
- Flask listens only on localhost.
- Prometheus is not publicly exposed.
- Alertmanager is not publicly exposed.
- Node Exporter is not publicly exposed.
- Blackbox Exporter is not publicly exposed.
- Only the required web traffic is publicly accessible.

For a production environment, the setup would need additional security controls, authentication, secrets management, and a production-grade Flask deployment.

---

# 🚧 Current Limitations

There are a few limitations in this version.

### Single Server

Everything currently runs on one EC2 instance.

If the entire EC2 instance goes down, the monitoring system also goes down.

### Flask Development Server

The Flask application currently uses Flask's built-in development server.

For production, it should be replaced with something like Gunicorn behind a proper service configuration.

### Basic Recovery Logic

The current recovery action is focused on restarting NGINX.

More advanced systems could have different recovery actions depending on the alert.

---

# 🔮 Future Improvements

There are several things I would like to add in a future version:

- 📊 Grafana dashboards
- 📧 Email notifications
- 💬 Slack notifications
- 🐳 Docker-based deployment
- 🏗️ Terraform for infrastructure provisioning
- 🔐 Better secrets management
- 🌐 Multiple web servers
- ❤️ More advanced health checks
- 💻 CPU/RAM/disk alerts
- 🔁 Retry and failure-handling logic
- ☁️ More scalable AWS architecture
- 🚀 CI/CD pipeline
- 📈 Long-term monitoring and visualization

A future version could also move the monitoring components to separate infrastructure so that monitoring remains available even if the application server itself completely fails.

---

# 🎯 Final Takeaway

The main idea behind this project is pretty simple:

> **Don't just monitor failures. Automate the recovery.**

This project demonstrates how several DevOps tools can work together to create a basic self-healing infrastructure workflow.

NGINX fails.

Prometheus notices.

Alertmanager sends the alert.

Flask receives it.

Ansible fixes the problem.

Then the system checks whether the service is healthy again.

```text
       DETECT
          ↓
        ALERT
          ↓
       AUTOMATE
          ↓
       RECOVER
          ↓
       VERIFY
          ↓
        HEALTHY
```

This project gave me practical experience with monitoring, alerting, Linux services, webhooks, Ansible automation, AWS EC2, and the basic idea behind self-healing infrastructure.

---

# ⭐ Project Status

**Status:** ✅ Completed and Tested

**Environment:** AWS EC2 + Ubuntu 24.04 LTS

**Main Goal:** Automated NGINX failure detection and recovery

**Result:** Successfully tested end-to-end self-healing workflow.
```
