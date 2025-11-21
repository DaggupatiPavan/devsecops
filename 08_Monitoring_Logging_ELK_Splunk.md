# 08. Monitoring & Logging: ELK, Splunk & Prometheus

Security is not a state; it's a process of continuous monitoring.

## 1. ELK Stack (Elasticsearch, Logstash, Kibana)

### A. Architecture
1.  **Beats (Filebeat)**: Lightweight agent on the server. Ships logs to Logstash.
2.  **Logstash**: The ETL pipeline. Parses, filters, and enriches logs (e.g., adds GeoIP).
3.  **Elasticsearch**: The Search Engine.
4.  **Kibana**: The UI.

### B. Logstash Configuration (`logstash.conf`)
The magic happens here. We use **Grok** filters to parse unstructured text into JSON.

```ruby
input {
  beats {
    port => 5044
  }
}

filter {
  if [event][module] == "nginx" {
    grok {
      match => { "message" => "%{IPORHOST:clientip} ... %{NUMBER:response}" }
    }
    geoip {
      source => "clientip"
    }
  }
}

output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "nginx-logs-%{+YYYY.MM.dd}"
  }
}
```

### C. Kibana for Security (KQL)
Kibana Query Language is powerful.
*   **Find SSH Failures**: `event.dataset: "system.auth" AND message: "Failed password"`
*   **Find Root Login**: `user.name: "root" AND event.outcome: "success"`
*   **Find SQL Injection Attempts**: `url.query: "SELECT" OR url.query: "UNION"`

---

## 2. Splunk: The Enterprise SIEM

### A. SPL (Search Processing Language)
Splunk is famous for its pipe-based language (like Linux).

#### Query 1: Brute Force Detection
Find IPs that failed to login 10 times in 5 minutes.
```splunk
index=linux_secure "Failed password"
| stats count by src_ip
| where count > 10
| sort - count
```

#### Query 2: Rare Process Execution
Find processes that have run less than 5 times in the last month (Anomaly Detection).
```splunk
index=os_logs sourcetype=ps
| stats count by process_name
| where count < 5
```

#### Query 3: Data Exfiltration
Find internal hosts sending > 1GB of data to external IPs.
```splunk
index=firewall direction=outbound
| stats sum(bytes) as total_bytes by src_ip, dest_ip
| where total_bytes > 1073741824
```

---

## 3. Prometheus & Grafana: Metrics Security

### A. What to Monitor?
Logs tell you *what* happened. Metrics tell you *how much*.
*   **Spike in 403 Forbidden**: Someone is scanning your site.
*   **Spike in CPU**: Crypto mining?
*   **Drop in Traffic**: DoS attack?

### B. AlertManager Configuration (`alertmanager.yml`)
Don't stare at dashboards. Make the dashboard call you.

```yaml
route:
  receiver: 'slack-notifications'
  group_by: ['alertname']

receivers:
- name: 'slack-notifications'
  slack_configs:
  - api_url: 'https://hooks.slack.com/...'
    channel: '#security-alerts'
    text: "Alert: {{ .CommonAnnotations.summary }}"
```

### C. Prometheus Rules (`alerts.yml`)
```yaml
groups:
- name: security
  rules:
  - alert: HighErrorRate
    expr: rate(http_requests_total{status=~"5.."}[5m]) > 10
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "High 5xx error rate detected"
```

---

## 4. Interview Preparation

### Q1: What is the difference between Logging and Monitoring?
*   **Answer**:
    *   **Monitoring**: "Is the system healthy?" (Metrics: CPU, RAM, Latency). It answers "Is it working?".
    *   **Logging**: "What is the system doing?" (Events: User login, Error stack trace). It answers "Why is it broken?".

### Q2: How do you handle "Log Flooding" during an attack?
*   **Answer**:
    *   "If an attacker generates millions of logs to crash my SIEM (DoS):
    *   1. **Sampling**: Configure Logstash to drop 90% of DEBUG logs.
    *   2. **Buffering**: Use a message queue (Kafka/Redis) before Logstash to absorb the spike.
    *   3. **Circuit Breaker**: Temporarily stop logging from the specific IP."

### Q3: Explain "Correlation" in SIEM.
*   **Answer**:
    *   "Correlation is connecting two seemingly unrelated events.
    *   Event A: User logs in from VPN (Normal).
    *   Event B: Same User logs in from China 5 minutes later (Impossible Travel).
    *   SIEM correlates A + B and triggers an alert."

---

## 5. Hands-On Lab
1.  **ELK**: Spin up ELK stack using Docker Compose.
2.  **Filebeat**: Install Filebeat on your host machine and ship `/var/log/auth.log` to Logstash.
3.  **Kibana**: Create a Dashboard showing "Top 10 Failed SSH IPs".
4.  **Splunk**: Install Splunk Free. Index some dummy logs. Write an SPL query to find the word "error".
