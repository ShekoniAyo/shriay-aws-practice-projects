# Three-Tier Web App on AWS — Hands-On Practice

**Scenario:** AtlasWorks needs an internal inventory dashboard — a web app behind a load balancer, running on Auto Scaling EC2 instances, backed by a Multi-AZ RDS database. Nothing serverless this time — this is the classic architecture the SAA exam tests most heavily: public/private subnets, security groups as the real access boundary, and a database that survives an AZ failure.

**Architecture:**
```
Internet → ALB (public subnets) → EC2 in ASG (private app subnets) → RDS Multi-AZ (private db subnets)
                                          ↑
                                   NAT Gateway (public subnet, for outbound updates only)
```

No SSH keys, no public IPs on app or database instances — you'll use **Session Manager** to reach the app instances, which is both the modern practice and one less thing that can leak.

---

# Part 1 — VPC and subnets

## 1. Create the VPC

1. **VPC → Create VPC**
2. Name: `atlasworks-vpc`
3. IPv4 CIDR: `10.0.0.0/16`
4. Create the VPC

## 2. Create six subnets across two Availability Zones

| Subnet name | CIDR | AZ | Tier |
|---|---|---|---|
| `public-a` | `10.0.0.0/24` | AZ-a | Public |
| `public-b` | `10.0.1.0/24` | AZ-b | Public |
| `app-a` | `10.0.10.0/24` | AZ-a | Private (app) |
| `app-b` | `10.0.11.0/24` | AZ-b | Private (app) |
| `db-a` | `10.0.20.0/24` | AZ-a | Private (db) |
| `db-b` | `10.0.21.0/24` | AZ-b | Private (db) |

Create all six under **VPC → Subnets → Create subnet**, all inside `atlasworks-vpc`.

**Why two AZs:** this is the whole point of the exercise — if one AZ goes down, the ALB, ASG, and RDS all have a healthy resource in the other AZ to fall back on. A single-AZ version of this architecture isn't really testable for the failure scenarios that matter later in this guide.

## 3. Create and attach an Internet Gateway

1. **VPC → Internet Gateways → Create**, name `atlasworks-igw`
2. Select it → **Actions → Attach to VPC** → `atlasworks-vpc`

## 4. Create a NAT Gateway

1. **VPC → NAT Gateways → Create NAT Gateway**
2. Subnet: `public-a`
3. Allocate a new Elastic IP
4. Create it — this is what lets your **private** app instances download OS updates/packages without being reachable from the internet themselves. One NAT Gateway (in one public subnet) is enough for practice; production would typically have one per AZ for resilience.

## 5. Create route tables

**Public route table:**
1. Create route table `public-rt`, associate with `public-a` and `public-b`
2. Add route: `0.0.0.0/0` → target `atlasworks-igw`

**App route table:**
1. Create route table `app-rt`, associate with `app-a` and `app-b`
2. Add route: `0.0.0.0/0` → target the NAT Gateway from Step 4

**DB route table:**
1. Create route table `db-rt`, associate with `db-a` and `db-b`
2. Leave it with only the default local route — **no route to the internet at all**. The database tier has no business reaching out anywhere.

---

# Part 2 — Security groups (your real access boundary)

Security groups are stateful and reference *each other* here, which is the pattern the exam cares about — not raw IP ranges.

1. **`alb-sg`** — Inbound: `HTTP 80` from `0.0.0.0/0`. Outbound: all.
2. **`app-sg`** — Inbound: `HTTP 80` from **`alb-sg`** (select the security group itself as the source, not an IP range). Outbound: all.
3. **`db-sg`** — Inbound: `MySQL/Aurora 3306` from **`app-sg`**. Outbound: all.

Notice the chain: internet → `alb-sg` → `app-sg` → `db-sg`. Nothing can reach the app tier except the ALB, and nothing can reach the database except the app tier — enforced by security group references, not subnet placement alone.

---

# Part 3 — RDS database

> **Free Tier note:** Multi-AZ RDS isn't Free Tier eligible — it runs a full synchronous standby instance, which doubles the cost. This section uses a **Single-AZ** `db.t3.micro` instead, which is Free Tier eligible for 12 months. The DB subnet group still spans both AZs (`db-a` and `db-b`) even though only one is used right now — that's not wasted effort, it's what lets you flip **Multi-AZ: Yes** later with zero redesign, since RDS just provisions the standby into the second subnet that's already there. If you're doing this exercise in a real job later with a full account, come back and re-run Part 8.3 with Multi-AZ enabled to see the actual failover behavior.

## 1. Create a DB subnet group

1. **RDS → Subnet groups → Create**
2. Name: `atlasworks-db-subnet-group`
3. VPC: `atlasworks-vpc`
4. Add subnets `db-a` and `db-b` — both, even though only one AZ will actually host the instance for now

## 2. Create the database

1. **RDS → Create database**
2. Engine: **MySQL**
3. Template: **Free tier** — this template automatically enforces Free Tier–eligible settings for you (single-AZ, `db.t3.micro`, no extra storage autoscaling), which is a good safety net while your trial is active
4. DB instance identifier: `atlasworks-db`
5. Master username: `admin`, set a password and note it down
6. Instance size: confirm it shows `db.t3.micro`
7. **Multi-AZ deployment:** leave unchecked / not offered (the Free Tier template hides this option entirely)
8. VPC: `atlasworks-vpc`; DB subnet group: `atlasworks-db-subnet-group`
9. **Public access: No**
10. VPC security group: `db-sg`
11. Initial database name: `atlasworks`
12. Create the database — single-AZ provisioning is noticeably faster than Multi-AZ, usually just a few minutes

## 3. Note the endpoint

Once available, copy the **endpoint** (e.g. `atlasworks-db.abcdef123456.us-east-1.rds.amazonaws.com`) — you'll put this in the app's environment shortly.

---

# Part 4 — Launch template for the app tier

## 1. Create an IAM role for the instances

1. **IAM → Roles → Create role** → trusted entity: EC2
2. Attach policy: `AmazonSSMManagedInstanceCore` (this is what lets you connect via Session Manager instead of SSH/key pairs)
3. Name it `atlasworks-app-role`

## 2. Create the launch template

1. **EC2 → Launch Templates → Create launch template**
2. Name: `atlasworks-app-template`
3. AMI: **Amazon Linux 2023**
4. Instance type: `t3.micro`
5. Key pair: **None** (you're using Session Manager, not SSH)
6. Network settings: security group `app-sg`
7. IAM instance profile: `atlasworks-app-role`
8. **Advanced details → User data**, paste this in:

```bash
#!/bin/bash
dnf install -y python3-pip
pip3 install flask pymysql

mkdir -p /opt/atlasworks
cat > /opt/atlasworks/app.py << 'EOF'
import os
import socket
from flask import Flask, jsonify
import pymysql

app = Flask(__name__)

DB_HOST = os.environ.get('DB_HOST', 'not-configured')
DB_USER = os.environ.get('DB_USER', 'admin')
DB_PASSWORD = os.environ.get('DB_PASSWORD', '')
DB_NAME = os.environ.get('DB_NAME', 'atlasworks')

def get_connection():
    return pymysql.connect(
        host=DB_HOST, user=DB_USER, password=DB_PASSWORD,
        database=DB_NAME, connect_timeout=5
    )

@app.route('/health')
def health():
    return jsonify({'status': 'healthy', 'instance': socket.gethostname()})

@app.route('/')
def index():
    try:
        conn = get_connection()
        with conn.cursor() as cur:
            cur.execute("SELECT item_name, quantity FROM inventory")
            rows = cur.fetchall()
        conn.close()
        items = ''.join(f"<tr><td>{r[0]}</td><td>{r[1]}</td></tr>" for r in rows)
        return f"""
        <h1>AtlasWorks Inventory</h1>
        <p>Served by instance: {socket.gethostname()}</p>
        <table border="1" cellpadding="8">
          <tr><th>Item</th><th>Quantity</th></tr>
          {items}
        </table>
        """
    except Exception as e:
        return f"<h1>AtlasWorks Inventory</h1><p>Served by: {socket.gethostname()}</p><p>DB error: {e}</p>", 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=80)
EOF

cat > /etc/systemd/system/atlasworks.service << 'EOF'
[Unit]
Description=AtlasWorks Inventory App
After=network.target

[Service]
Environment=DB_HOST=__DB_HOST__
Environment=DB_USER=admin
Environment=DB_PASSWORD=__DB_PASSWORD__
Environment=DB_NAME=atlasworks
ExecStart=/usr/bin/python3 /opt/atlasworks/app.py
Restart=always
User=root

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable atlasworks
systemctl start atlasworks
```

**Before saving the template**, replace `__DB_HOST__` with your RDS endpoint from Part 3, and `__DB_PASSWORD__` with the master password you set. (For real production use you'd pull these from Secrets Manager/Parameter Store instead of embedding them in user data — worth doing as a follow-up hardening exercise once this works.)

9. Save the launch template

---

# Part 5 — Load the inventory table

Since the database has no public access, you'll reach it through an app-tier instance using Session Manager port forwarding.

## 1. Launch one temporary EC2 instance manually to seed the database

1. **EC2 → Launch instance**, same AMI/type/subnet (`app-a`)/security group (`app-sg`)/IAM role (`atlasworks-app-role`) as the launch template, no key pair
2. Once running, select it → **Connect → Session Manager → Connect** — this opens a browser-based shell, no SSH needed

## 2. From that Session Manager shell, install a MySQL client and seed data

```bash
sudo dnf install -y mariadb105
mysql -h <your-rds-endpoint> -u admin -p atlasworks
```

Enter the master password when prompted, then run:

```sql
CREATE TABLE inventory (
  item_name VARCHAR(100),
  quantity INT
);

INSERT INTO inventory (item_name, quantity) VALUES
  ('Steel Beams', 240),
  ('Copper Wiring (m)', 1500),
  ('Safety Helmets', 88),
  ('Forklift Pallets', 32);

EXIT;
```

3. Terminate this temporary instance once done — it was only needed to seed the table.

---

# Part 6 — Target group and Application Load Balancer

## 1. Create the target group

1. **EC2 → Target Groups → Create target group**
2. Type: **Instances**
3. Name: `atlasworks-tg`
4. Protocol: HTTP, port 80
5. VPC: `atlasworks-vpc`
6. Health check path: `/health`
7. Create it (leave it empty for now — the ASG will register instances automatically)

## 2. Create the ALB

1. **EC2 → Load Balancers → Create → Application Load Balancer**
2. Name: `atlasworks-alb`
3. Scheme: **Internet-facing**
4. VPC: `atlasworks-vpc`; subnets: `public-a` and `public-b`
5. Security group: `alb-sg`
6. Listener: HTTP 80 → forward to `atlasworks-tg`
7. Create the load balancer

---

# Part 7 — Auto Scaling Group

## 1. Create the ASG

1. **EC2 → Auto Scaling Groups → Create**
2. Name: `atlasworks-asg`
3. Launch template: `atlasworks-app-template`
4. VPC: `atlasworks-vpc`; subnets: `app-a` and `app-b` — **not the public subnets**, this is what makes it a genuinely private app tier
5. Attach to existing load balancer target group: `atlasworks-tg`
6. Health checks: enable **ELB health checks** (not just EC2 status checks — this way the ASG replaces an instance if the app itself is unhealthy, not just if the VM is down)
7. Desired capacity: `2`, Minimum: `2`, Maximum: `4`
8. Scaling policy: **Target tracking**, metric: Average CPU utilization, target value: `50`
9. Create the ASG

## 2. Wait for instances to launch and register

Give it a few minutes, then check **Target Groups → atlasworks-tg → Targets** — both instances should show `healthy`.

---

# Part 8 — Test it

## 1. Access the app

1. **EC2 → Load Balancers → atlasworks-alb** — copy the **DNS name**
2. Open it in a browser — you should see the AtlasWorks Inventory table with the four seeded items, and a note showing which instance served the request
3. Refresh several times — notice the "served by instance" hostname alternating between your two instances, confirming the ALB is actually load balancing across both

## 2. Test Auto Scaling

1. Connect to one of the running app instances via **Session Manager**
2. Generate CPU load to trigger scale-out:
```bash
sudo dnf install -y stress
stress --cpu 2 --timeout 300
```
3. Watch **Auto Scaling Groups → atlasworks-asg → Activity** — within a few minutes you should see a new instance launching as average CPU crosses the 50% target
4. After the 5-minute `stress` command ends, CPU drops and — after the cooldown period — the ASG should scale back down toward desired capacity

## 3. Test RDS recovery behavior (Single-AZ version)

Without Multi-AZ, a reboot is a real outage — there's no standby to fail over to, so this test shows you the *difference* Multi-AZ would make, which is arguably more instructive than watching a seamless failover.

1. **RDS → atlasworks-db → Actions → Reboot** (no failover option will be offered, since there's no standby)
2. While it's rebooting, refresh the app in your browser repeatedly — you should see the `DB error` on the page for the **full duration** of the reboot (typically 30–60 seconds), not just a couple of seconds
3. Once RDS shows `Available` again, the app should recover on its own on the next request, since the app reconnects fresh each time rather than holding a stale connection

**What Multi-AZ would change here:** with Multi-AZ enabled, that same reboot (with "reboot with failover" checked) would fail over to the already-running, already-synced standby in the second AZ — typically a 1-2 minute outage instead of this single-instance reboot, but more importantly, Multi-AZ protects you from **unplanned** failures (the underlying host dying, storage issues, an AZ outage) where there's no "reboot" at all, just a sudden disconnect. A planned reboot on Single-AZ and an unplanned host failure on Single-AZ look identical to your app: total unavailability until RDS recovers on the same host. That's the core exam distinction — Multi-AZ isn't about performance, it's about eliminating that single point of failure.

## 4. Confirm the private tiers are actually private

1. Try to find a public IP on any `app-a`/`app-b` instance or on the RDS instance in the console — there shouldn't be one
2. Try connecting to the RDS endpoint from your own laptop (outside the VPC) with a MySQL client — it should time out, since `db-sg` only allows traffic from `app-sg`, and there's no route from the internet into that subnet at all

---

## What you've built

A genuinely production-shaped architecture: public/private subnet separation enforced by route tables, security groups chained to each other rather than open IP ranges, an ALB distributing traffic across AZs, an Auto Scaling Group that reacts to real load and replaces unhealthy instances, and zero SSH keys anywhere — Session Manager for all instance access. The database runs Single-AZ to stay within Free Tier, but the subnet group already spans two AZs, so enabling Multi-AZ later is a single checkbox, not a rebuild.

**Suggested next extensions:**
- Move `DB_PASSWORD` out of user data and into **Secrets Manager**, with the app fetching it at startup via IAM role permissions instead
- Add a **Bastion-free** private CI/CD path: have the launch template pull app code from **CodeDeploy**/S3 instead of embedding it directly in user data
- Add an **NACL** on the db subnets as a second layer of defense on top of the security group, and test that both layers actually matter
- If you get access to a non-trial account later, re-run Part 8.3 with Multi-AZ actually enabled and compare the outage duration directly against what you saw here
