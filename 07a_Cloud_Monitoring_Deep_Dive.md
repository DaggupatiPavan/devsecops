# 07a. Cloud & Monitoring Deep Dive: IAM Boundaries, Grok & Exporters

## 1. AWS IAM Permissions Boundaries
**Problem**: You want to give a Junior Dev the ability to create IAM Users (so they can manage their team).
**Risk**: The Junior Dev creates a user with `AdministratorAccess` and escalates their own privileges.
**Solution**: Permissions Boundary.

### The Concept
A Boundary is a "Maximum Possible Permissions" shield.
*   **Rule**: "You can create a user, BUT you must attach the 'JuniorDevBoundary' to them."
*   **Boundary**: "This user can NEVER do `iam:DeleteUser` or `s3:DeleteBucket`."

### The Policy (JSON)
Attach this to the Junior Dev:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "CreateUserWithBoundary",
            "Effect": "Allow",
            "Action": "iam:CreateUser",
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "iam:PermissionsBoundary": "arn:aws:iam::123456789012:policy/JuniorDevBoundary"
                }
            }
        }
    ]
}
```
Now, if they try to create a user *without* the boundary, AWS denies it.

---

## 2. ELK Grok Patterns (Regex on Steroids)
**Problem**: You have a custom log format: `[2023-10-01] | ERROR | User: admin | Msg: Failed`
Logstash doesn't understand it.
**Solution**: Grok.

### The Pattern
```text
\[%{TIMESTAMP_ISO8601:timestamp}\] \| %{LOGLEVEL:level} \| User: %{USERNAME:user} \| Msg: %{GREEDYDATA:message}
```

### Testing (Kibana Dev Tools)
You don't need to restart Logstash to test. Use the Kibana Debugger.
```json
POST _ingest/pipeline/_simulate
{
  "pipeline": {
    "processors": [
      {
        "grok": {
          "field": "message",
          "patterns": ["\\[%{TIMESTAMP_ISO8601:timestamp}\\] \\| %{LOGLEVEL:level} ..."]
        }
      }
    ]
  },
  "docs": [
    { "_source": { "message": "[2023-10-01] | ERROR | User: admin | Msg: Failed" } }
  ]
}
```

---

## 3. Prometheus Custom Exporters (Python)
**Problem**: You want to monitor "Number of Magic Links Sent". This isn't a CPU metric.
**Solution**: Write a tiny Python web server that Prometheus scrapes.

### The Code (`exporter.py`)
```python
from prometheus_client import start_http_server, Counter
import time
import random

# 1. Define the Metric
MAGIC_LINKS = Counter('magic_links_sent_total', 'Total number of magic links sent')

def process_request():
    # Your app logic here...
    # 2. Increment the Metric
    MAGIC_LINKS.inc()

if __name__ == '__main__':
    # 3. Start Server on Port 8000
    start_http_server(8000)
    print("Exporter running on port 8000...")
    
    while True:
        process_request()
        time.sleep(random.random())
```

### Prometheus Config (`prometheus.yml`)
```yaml
scrape_configs:
  - job_name: 'my-custom-app'
    static_configs:
      - targets: ['localhost:8000']
```
Now Prometheus will scrape `http://localhost:8000/metrics` and get your custom data.
