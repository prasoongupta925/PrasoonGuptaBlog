---
title: "When Your Infrastructure Scales to 10+ Servers, Logs Become Your Nightmare"
seoTitle: "Managing Log Overload in Large Infrastructure"
seoDescription: "Create a cost-effective logging system with Promtail, Loki, and Grafana for managing logs in WordPress infrastructure with over 10 servers"
datePublished: Mon Mar 24 2025 18:30:00 GMT+0000 (Coordinated Universal Time)
cuid: cmkgpvgbp000102jo02ie5vv8
slug: when-your-infrastructure-scales-to-10-servers-logs-become-your-nightmare
tags: aws, cloud-computing, monitoring, devops, loki, grafana, observability, promtail

---

## Building a self-managed centralized logging system using Promtail, Loki, and Grafana to monitor distributed WordPress infrastructure handling 12M+ daily requests—without expensive managed observability services

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1768212059311/54748bf5-b0a4-4d6f-90e6-53f79c78e6db.jpeg align="center")

After scaling robu.in from one server to a distributed WordPress infrastructure handling 12 million daily requests, I faced a new problem: when something breaks at 3 AM, which of the 10+ servers is causing it? Hunting through SSH sessions and individual log files was no longer viable. This is the complete story of how I built a production-grade centralized logging system using the PLG stack (Promtail + Loki + Grafana) that monitors every request, error, and slow query across the entire infrastructure—saving hours of debugging time and catching issues before customers notice.

> ***Series Note: This is Episode 2 of an 8-part series documenting my journey building self-managed WordPress infrastructure at scale.*** [***Read Episode 1: NFS Foundation***](https://medium.com/@prasoongupta925/building-a-self-managed-file-sharing-system-using-nfs-across-cloud-providers-77da35202581)

## 🚨 The Distributed Logging Problem

In Episode 1, I showed you how NFS enabled horizontal scaling. The infrastructure now looked like this:

**5+ WordPress servers** (auto-scaling across DigitalOcean and AWS)  
**2 MySQL database servers** (master-slave replication)  
**2 PostgreSQL servers** (for Odoo ERP)  
**2 Odoo application servers**  
**1 staging server** (st-server for testing)

Total: **12+ production servers** generating logs 24/7.

## The Old Way (SSH Hell)

When a customer reported a checkout error, my debugging workflow was:

1. SSH into load balancer → check which backend served the request
    
2. SSH into that WordPress server → tail nginx access logs
    
3. SSH into another WordPress server → check if error repeated
    
4. SSH into MySQL server → check slow query log
    
5. SSH into another server... you get the idea
    

**Average time to identify root cause: 20-45 minutes.**

For an e-commerce platform doing ₹257 crore in revenue, every minute of debugging = potential lost sales.

## The "Managed" Solutions Were Too Expensive

I researched the standard options:

**AWS CloudWatch Logs:**

* $0.50 per GB ingested
    
* $0.03 per GB stored per month
    
* For 50GB daily logs = $750/month + query costs
    

**Datadog:**

* $15 per host per month
    
* 12 servers × $15 = $180/month minimum
    
* Log retention costs extra
    

**Elastic Stack (ELK):**

* Heavy resource consumption
    
* Complex to manage
    
* Elasticsearch cluster = expensive
    

For a bootstrapped infrastructure, I needed something self-managed, lightweight, and cost-effective.

---

## 💡 The PLG Stack Solution

I chose the **PLG stack** (Promtail + Loki + Grafana) for centralized logging:

**Promtail** → Lightweight log shipper on each server  
**Loki** → Log aggregation and indexing (like Prometheus, but for logs)  
**Grafana** → Visualization and alerting dashboard

## Why PLG over ELK?

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1768214953351/a05ede64-3934-40fc-9f66-6f297fe3d627.png align="center")

For a distributed WordPress infrastructure, PLG was the perfect fit.

---

## 🏗️ Architecture Overview

**The Flow:**

```plaintext
WordPress Servers (wp2, wp3) → Promtail → Loki → Grafana
MySQL Servers → Promtail → Loki → Grafana
PostgreSQL/Odoo Servers → Promtail → Loki → Grafana
Staging Server → Promtail → Loki → Grafana
```

**One centralized monitoring server** running Loki + Grafana receives logs from all 12+ servers in real-time.

No SSH needed. No manual log tailing. Just open Grafana dashboard and see everything.

---

## ⚙️ Implementation: Step-by-Step

## Phase 1: Installing Loki (Central Server)

On the monitoring server ([log.](http://log.robu.in)xyx.in):

**1\. Download and install Loki:**

```bash
cd /usr/local/bin
sudo curl -O -L "https://github.com/grafana/loki/releases/download/v2.9.3/loki-linux-amd64.zip"
sudo unzip "loki-linux-amd64.zip"
sudo chmod a+x "loki-linux-amd64"
```

**2\. Create Loki configuration:**

```bash
sudo mkdir -p /etc/loki
sudo nano /etc/loki/loki-local-config.yaml
```

```yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  path_prefix: /tmp/loki
  storage:
    filesystem:
      chunks_directory: /tmp/loki/chunks
      rules_directory: /tmp/loki/rules
  replication_factor: 1
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

limits_config:
  retention_period: 720h  # 30 days retention
```

**3\. Create systemd service:**

```bash
sudo nano /etc/systemd/system/loki.service
```

```bash
[Unit]
Description=Loki service
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/loki-linux-amd64 -config.file=/etc/loki/loki-local-config.yaml
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

**4\. Start Loki:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable loki
sudo systemctl start loki
sudo systemctl status loki
```

**5\. Configure firewall:**

```bash
# Allow Loki port from all servers
sudo ufw allow from 10.139.48.0/24 to any port 3100
sudo ufw allow from 172.31.0.0/16 to any port 3100
```

Loki is now running and listening on port 3100.

---

## Phase 2: Installing Promtail (On Each Server)

Promtail needs to be installed on **every server** that generates logs.

**On WordPress servers (wp2, wp3, st-server):**

**1\. Download Promtail:**

```plaintext
cd /usr/local/bin
sudo curl -O -L "https://github.com/grafana/loki/releases/download/v2.9.3/promtail-linux-amd64.zip"
sudo unzip "promtail-linux-amd64.zip"
sudo chmod a+x "promtail-linux-amd64"
```

**2\. Create Promtail configuration:**

```bash
sudo mkdir -p /etc/promtail
sudo nano /etc/promtail/promtail-config.yaml
```

**Configuration for WordPress server (wp2):**

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://log.robu.in:3100/loki/api/v1/push

scrape_configs:
  # Nginx access logs
  - job_name: nginx-access-log
    static_configs:
      - targets:
          - localhost
        labels:
          job: nginx-access-log-wp2
          host: wp2
          __path__: /var/log/nginx/access.log

  # Nginx error logs
  - job_name: nginx-error-log
    static_configs:
      - targets:
          - localhost
        labels:
          job: nginx-error-log-wp2
          host: wp2
          __path__: /var/log/nginx/error.log

  # PHP slow request log
  - job_name: php-slow-log
    static_configs:
      - targets:
          - localhost
        labels:
          job: php-slow-request-log-wp2
          host: wp2
          __path__: /var/log/php8.1-fpm-slow.log

  # PHP error log
  - job_name: php-error-log
    static_configs:
      - targets:
          - localhost
        labels:
          job: php-error-log-wp2
          host: wp2
          __path__: /var/log/php8.1-fpm.log

  # WordPress debug log
  - job_name: wordpress-debug-log
    static_configs:
      - targets:
          - localhost
        labels:
          job: wordpress-debug-log-wp2
          host: wp2
          __path__: /var/www/html/wp-content/debug.log
```

**3\. Create systemd service:**

```plaintext
sudo nano /etc/systemd/system/promtail.service
```

```plaintext
[Unit]
Description=Promtail service
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/promtail-linux-amd64 -config.file=/etc/promtail/promtail-config.yaml
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

**4\. Start Promtail:**

```plaintext
sudo systemctl daemon-reload
sudo systemctl enable promtail
sudo systemctl start promtail
sudo systemctl status promtail
```

**5\. Verify logs are being sent:**

```bash
# Check Promtail is reading logs
curl http://localhost:9080/metrics

# Check Loki is receiving logs (on monitoring server)
curl http://localhost:3100/loki/api/v1/label
```

---

## Phase 3: MySQL Slow Query Logging

For MySQL database servers, capturing slow queries was critical for performance optimization.

**On MySQL server:**

**1\. Enable slow query log in MySQL:**

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Add:

```plaintext
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time = 2
log_queries_not_using_indexes = 1
```

```bash
sudo systemctl restart mysql
```

**2\. Configure Promtail for MySQL logs:**

```plaintext
scrape_configs:
  # MySQL slow query log
  - job_name: mysql-slow-query-log
    static_configs:
      - targets:
          - localhost
        labels:
          job: mysql-slow-query-log-production
          host: mysql-master
          __path__: /var/log/mysql/mysql-slow.log

  # MySQL error log
  - job_name: mysql-error-log
    static_configs:
      - targets:
          - localhost
        labels:
          job: mysql-error-log-production
          host: mysql-master
          __path__: /var/log/mysql/error.log
```

Start Promtail and MySQL logs start flowing to Loki.

---

## Phase 4: PostgreSQL & Odoo Logging

For the Odoo ERP system running on PostgreSQL:

**PostgreSQL slow query configuration:**

```bash
sudo nano /etc/postgresql/14/main/postgresql.conf
```

```plaintext
log_min_duration_statement = 2000  # Log queries taking > 2 seconds
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d '
logging_collector = on
log_directory = 'pg_log'
log_filename = 'postgresql-%Y-%m-%d.log'
```

```bash
sudo systemctl restart postgresql
```

**Promtail config for PostgreSQL:**

```yaml
scrape_configs:
  - job_name: postgres-slow-query-log
    static_configs:
      - targets:
          - localhost
        labels:
          job: postgres-server-live-odoo-slow-query-log
          host: odoo-db
          __path__: /var/log/postgresql/pg_log/*.log
```

**Odoo application logs:**

```yaml
scrape_configs:
  - job_name: odoo-app-log
    static_configs:
      - targets:
          - localhost
        labels:
          job: odoo-application-log
          host: odoo1
          __path__: /var/log/odoo/odoo-server.log
```

---

## Phase 5: Grafana Dashboard Creation

On the monitoring server with Grafana installed:

**1\. Add Loki as data source:**

* Navigate to **Configuration → Data Sources**
    
* Click **Add data source**
    
* Select **Loki**
    
* URL: [`http://localhost:3100`](http://localhost:3100)
    
* Click **Save & Test**
    

**2\. Create production dashboards:**

I created 6 production dashboards to monitor the entire infrastructure:

## Dashboard 1: MySQL Slow Query Logs

**Query to find slow queries:**

```plaintext
{job="mysql-slow-query-log-production"} 
|= "Query_time"
| regexp "Query_time: (?P<duration>\\d+\\.\\d+)"
| duration > 2
```

**Visualization:** Table showing slow queries with execution time

## Dashboard 2: Nginx Error Logs

**Query for 5xx errors:**

```plaintext
{job="nginx-error-log-wp2"}
|~ "error|crit|alert|emerg"
```

**Visualization:** Time series graph showing error rate over time

## Dashboard 3: PHP Slow Request Log

**Query for requests taking &gt;5 seconds:**

```plaintext
{job="php-slow-request-log-wp2"}
| regexp "pool www.*request: \"(?P<request>.*)\".*\\d+ sec"
```

**Visualization:** Table with request URI and execution time

## Dashboard 4: WordPress Debug Log

**Query for fatal errors:**

```plaintext
{job="wordpress-debug-log-wp2"}
|~ "Fatal error|WARNING|CRITICAL"
```

**Panel:** Recent errors with context

## Dashboard 5: PostgreSQL Slow Queries (Odoo)

This was critical for tracking specific Odoo ERP slow queries:

**Query to count sale\_order\_line queries:**

```plaintext
count_over_time({job="postgres-server-live-odoo-slow-query-log"} 
|= "FROM \"sale_order_line\"" [24h])
```

**Query for queries &gt;2 seconds:**

```plaintext
sum(count_over_time({job="postgres-server-live-odoo-slow-query-log"} 
|~ "duration: [2-9][0-9]{3,}\\.[0-9]+ ms" [1h]))
```

These LogQL queries helped identify that `sale_order_line` table queries were causing 40% of database load—leading to optimization work that improved order processing speed by 3x.

## Dashboard 6: Combined Infrastructure Health

**Multi-server error aggregation:**

```plaintext
sum by (host) (rate({job=~".*error.*"}[5m]))
```

**Visualization:** Bar chart showing error rate per server

This dashboard immediately showed which server in the auto-scaling group was having issues.

---

## 📊 Real Production Scenarios

## Scenario 1: The 3 AM Alert

**Problem:** Customer couldn't complete checkout at 2:47 AM.

**Old way:** Would discover the issue next morning. Lost sale.

**With PLG:** Alert fired immediately. Grafana dashboard showed:

```plaintext
[wp3] nginx-error-log: "connect() failed (111: Connection refused) while connecting to upstream"
```

**Root cause:** Redis session server restarted (covered in Episode 4).  
**Resolution time:** 5 minutes.  
**Customer impact:** Minimal—issue fixed before morning traffic spike.

## Scenario 2: Slow Product Search

**Symptom:** Users reporting slow product search results.

**Grafana query:**

```plaintext
{job="mysql-slow-query-log-production"} 
|= "wp_posts" 
| regexp "Query_time: (?P<time>\\d+)"
```

**Discovery:** Product search queries taking 8-12 seconds due to missing database index.

**Action:** Added index on `wp_`[`posts.post`](http://posts.post)`_title`:

```sql
ALTER TABLE wp_posts ADD INDEX idx_post_title (post_title(50));
```

**Result:** Search time reduced from 8s → 0.3s. Discovered in 10 minutes instead of days.

## Scenario 3: Odoo Invoice Generation Slowdown

**Symptom:** Finance team reporting slow invoice generation in Odoo ERP.

**Grafana PostgreSQL dashboard showed:**

```plaintext
{job="postgres-server-live-odoo-slow-query-log"} 
|= "SELECT \"sale_order_line\".id"
```

Query count: **15,000+ queries in 24 hours**.

**Root cause:** N+1 query problem in Odoo custom module.

**Resolution:** Optimized module to use batch queries. Invoice generation time: 45s → 8s.

**Without centralized logging:** This would have taken weeks to identify.

---

## 💰 Cost Analysis

## PLG Stack (Self-Managed)

**Infrastructure:**

* 1 monitoring server (t3.medium): $30/month
    
* Promtail on existing servers: $0 (minimal CPU/RAM)
    
* Storage for 30-day log retention: $15/month (EBS)
    

**Total: $45/month**

## vs. Managed Solutions

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1768216839119/9bc1a485-0cb5-4431-a43e-bf09acb20edd.png align="center")

**Savings: $550-750/month = $6,600-9,000/year**

Over 3 years: **$20,000-27,000 saved** while maintaining complete control and unlimited retention.

---

## 📈 Production Results

After implementing PLG stack across all infrastructure:

## Operational Metrics

✅ **Mean Time to Detection (MTTD):** 2 minutes (down from 20-45 minutes)  
✅ **Mean Time to Resolution (MTTR):** 15 minutes (down from 1-3 hours)  
✅ **Incidents caught proactively:** 70% (before customer reports)  
✅ **Debug time saved:** 10-15 hours/week  
✅ **Log retention:** 30 days (vs. 7 days with CloudWatch free tier)

## Business Impact

* **Reduced downtime:** From 99.5% to 99.9% uptime
    
* **Faster incident response:** Issues resolved before peak traffic hours
    
* **Proactive optimization:** Identified slow queries before they caused outages
    
* **Cost savings:** $6,600+/year vs managed logging
    

## Technical Wins

* **12+ servers monitored** from single Grafana dashboard
    
* **Real-time visibility** into nginx, PHP, MySQL, PostgreSQL, Odoo logs
    
* **Custom alerting** via Grafana (Slack, email, PagerDuty)
    
* **Zero vendor lock-in:** All open-source, portable to any cloud
    

---

## 🎯 Key Takeaways

## 1\. Centralized Logging is Non-Negotiable at Scale

Once you scale beyond 2-3 servers, SSH-based debugging becomes impossible. Centralized logging isn't a luxury—it's a requirement for maintaining production systems.

## 2\. PLG Stack is Production-Ready

The combination of Promtail + Loki + Grafana is lightweight, powerful, and cost-effective. It handles 50GB+ daily logs without breaking a sweat on a $30/month server.

## 3\. LogQL is Powerful

Loki's LogQL query language (similar to Prometheus PromQL) makes log analysis fast and flexible. Complex queries like regex extraction and aggregation are simple to write.

## 4\. Log Everything That Matters

We log:

* Nginx access/error logs
    
* PHP slow requests/errors
    
* WordPress debug logs
    
* MySQL slow queries
    
* PostgreSQL slow queries
    
* Odoo application logs
    

**The more context you have, the faster you debug.**

## 5\. Retention = Long-Term Analysis

With 30-day retention, we can analyze trends:

* Which days have highest error rates?
    
* Are slow queries increasing over time?
    
* What's the correlation between traffic spikes and errors?
    

Managed services charge premium for retention. Self-managed = unlimited storage at disk costs.

## 6\. Alerting Saves Time

Grafana's alerting routes critical errors to Slack in real-time. No more manual dashboard checking—issues come to you.

---

## 🔮 What's Next: Episode 3

Centralized logging solved observability. But robu.in's traffic was growing 22% year-over-year. Manual server management was no longer sustainable.

## Episode 3 Preview: Horizontal Scaling & Auto-Scaling

**Coming next month:** How I migrated from manual DigitalOcean droplets to AWS Auto Scaling Groups.

You'll learn:

* EC2 Launch Templates with user data automation
    
* Auto Scaling Groups configuration for WordPress
    
* Application Load Balancer setup and health checks
    
* Target Group auto-registration
    
* Scaling policies based on CPU and request count
    
* Zero-downtime deployments during traffic spikes
    

The infrastructure now automatically adds servers during traffic spikes and removes them during low traffic—handling Black Friday 300% traffic increases without manual intervention.

**Subscribe to catch Episode 3!**

---

## 📚 Resources

* **Loki Documentation:** [**https://grafana.com/docs/loki/latest/**](https://grafana.com/docs/loki/latest/)
    
* **LogQL Query Examples:** [**https://grafana.com/docs/loki/latest/query/query\_examples/**](https://grafana.com/docs/loki/latest/query/query_examples/)
    
* **Promtail Configuration:** [**https://grafana.com/docs/loki/latest/send-data/promtail/configuration/**](https://grafana.com/docs/loki/latest/send-data/promtail/configuration/)
    

---

## Questions?

Have you implemented centralized logging for distributed systems? What challenges did you face?

**Drop a comment below** or connect with me on [**LinkedIn**](https://www.linkedin.com/in/prasoongupta925)!

---

## About Me

I'm a CLoud DevOps Engineer and Infrastructure Architect specializing in:

* Multi-cloud solutions (AWS, DigitalOcean, Azure)
    
* Self-managed infrastructure at scale
    
* Cost-optimized architectures
    
* Production observability
    

🌐 Technical blog: [**cloudwithprasoon.tech**](https://cloudwithprasoon.tech/)

---

#Grafana #Loki #Promtail #Observability #Logging #DevOps #WordPress #Infrastructure #Monitoring #AWS