# 🔍 ELK-Setup

> Splunk-free SOC logging with the ELK stack — Docker Compose, syslog ingestion, Kibana dashboards

![Docker Compose](https://img.shields.io/badge/Docker_Compose-2496ED?style=flat&logo=docker&logoColor=white)
![Elasticsearch](https://img.shields.io/badge/Elasticsearch_8.x-005571?style=flat&logo=elasticsearch&logoColor=white)
![Kibana](https://img.shields.io/badge/Kibana-005571?style=flat&logo=kibana&logoColor=white)
![Logstash](https://img.shields.io/badge/Logstash-005571?style=flat&logo=logstash&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu_22.04-E95420?style=flat&logo=ubuntu&logoColor=white)

> **Why ELK?** Elastic's open-source stack covers everything a Splunk deployment does — ingestion, parsing, search, and dashboards — without the licensing cost. Building this from scratch is a real SOC skill that recruiters notice.
>
> ⏱️ **Estimated time:** ~7 hours (setup-heavy; dashboards are fast once the stack is live)

---

## 📁 Portfolio Output

```
ELK-Setup/
├── docker-compose.yml          # Elasticsearch + Logstash + Kibana
├── logstash/
│   └── pipeline/
│       └── syslog.conf         # UDP 514 input → ES output
├── screenshots/
│   ├── kibana-discover.png
│   └── auth-failures-dashboard.png
└── README.md
```

---

## ✅ Prerequisites

- Docker Engine 24+ and Docker Compose v2 installed
- Ubuntu 22.04 host (VM or bare metal) — syslog source
- At least 6 GB RAM available (Elasticsearch is memory-hungry)
- Ports `5601`, `9200`, `5514` open on host firewall
- ⚠️ Run `sysctl -w vm.max_map_count=262144` — required by Elasticsearch

---

## 📊 Overview

| | |
|---|---|
| **Total time** | ~7 hours |
| **Skill level** | Intermediate (Docker familiarity helpful) |
| **Stack version** | ELK 8.13 (all images pinned to same tag) |
| **SOC relevance** | High — SIEM ingestion, log analysis, KQL queries |

---

## 🪜 Step-by-Step Guide

### Step 1 — Set the kernel parameter (host OS)

Elasticsearch requires a high virtual memory limit. Set it on your Ubuntu host **before** launching any containers — without this, Elasticsearch exits immediately with an error.

```bash
# Temporary (resets on reboot)
sudo sysctl -w vm.max_map_count=262144

# Permanent — add to sysctl.conf
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

---

### Step 2 — Create the project structure

Scaffold the directory layout. The Logstash pipeline directory is mounted as a volume so you can edit config files without rebuilding the container.

```bash
mkdir -p week8-elk-setup/logstash/pipeline
cd week8-elk-setup
```

---

### Step 3 — Write `docker-compose.yml`

This Compose file brings up all three services pinned to the same version tag. Security is disabled for simplicity in a lab environment — **never do this in production**.

```yaml
version: '3.8'

services:

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.13.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false   # lab only
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
    ports:
      - "9200:9200"
    volumes:
      - esdata:/usr/share/elasticsearch/data
    networks:
      - elk

  logstash:
    image: docker.elastic.co/logstash/logstash:8.13.0
    container_name: logstash
    ports:
      - "5514:5514/udp"   # syslog ingestion
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
    depends_on:
      - elasticsearch
    networks:
      - elk

  kibana:
    image: docker.elastic.co/kibana/kibana:8.13.0
    container_name: kibana
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch
    networks:
      - elk

volumes:
  esdata:

networks:
  elk:
```

> **Memory note:** `ES_JAVA_OPTS=-Xms1g -Xmx1g` allocates 1 GB to Elasticsearch. On machines with less than 8 GB RAM, lower this to `512m`. Kibana and Logstash each need ~512 MB on top.

---

### Step 4 — Configure the Logstash syslog pipeline

Logstash uses a pipeline config to define how logs enter (**input**), how they are transformed (**filter**), and where they go (**output**). Create the file below to listen for UDP syslog on port 5514 and forward parsed events to Elasticsearch.

```ruby
# logstash/pipeline/syslog.conf

input {
  udp {
    port => 5514
    type => "syslog"
  }
}

filter {
  # Tag successful SSH logins
  if "Accepted password" in [message] {
    mutate { add_tag => ["auth_success"] }
  }

  # Tag and parse failed SSH logins
  if "Failed password" in [message] {
    grok {
      match => {
        "message" => [
          "^<%{INT:syslog_pri}>%{MONTH:month} +%{MONTHDAY:day} %{TIME:time} %{HOSTNAME:syslog_host} %{DATA:program}\[%{NUMBER:pid}\]: Failed password for (invalid user )?%{USERNAME:auth_username} from %{IP:auth_source_ip} port %{NUMBER:auth_port} ssh2$",
          "^<%{INT:syslog_pri}>%{INT:version} %{TIMESTAMP_ISO8601:syslog_timestamp} %{HOSTNAME:syslog_host} %{WORD:program} %{NUMBER:pid} - - Failed password for (invalid user )?%{USERNAME:auth_username} from %{IP:auth_source_ip} port %{NUMBER:auth_port} ssh2$",
          "^<%{INT:syslog_pri}>%{MONTH:month} +%{MONTHDAY:day} %{TIME:time} %{HOSTNAME:syslog_host} %{DATA:program}\[%{NUMBER:pid}\]: Failed password for (invalid user )?%{USERNAME:auth_username} from %{IP:auth_source_ip} port %{NUMBER:auth_port}$"
        ]
      }
    }

    mutate {
      add_tag => ["auth_failure"]
    }
  }

  # GeoIP only for failed logins
  if "auth_failure" in [tags] {
    geoip {
      source => "auth_source_ip"
      target => "geoip"
      database => "/usr/share/GeoIP/GeoLite2-City.mmdb"
    }
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "syslog-%{+YYYY.MM.dd}"
    retry_initial_interval => 5
    retry_max_interval => 30
    timeout => 60
  }
  stdout { codec => rubydebug }
}



```

> **How it works:** The `syslog` input plugin handles RFC 3164 and RFC 5424 automatically. The `grok` filter parses fields into named keys. The daily index pattern (`syslog-YYYY.MM.dd`) is the industry standard for rolling log retention.

---

### Step 5 — Launch the stack

```bash
# Start all three services in the background
docker compose up -d

# Watch Elasticsearch health (wait for "green" or "yellow")
docker compose logs -f elasticsearch

# Confirm the API is responding (~60 sec after start)
curl -s http://localhost:9200/_cluster/health | python3 -m json.tool
```

> **Expected output:** Elasticsearch will report `"status": "yellow"` on a single-node cluster (no replica shards). That is normal and expected — the stack is healthy for lab use.

---

### Step 6 — Forward Ubuntu logs to Logstash

Tell rsyslog on the Ubuntu host to forward all log entries to Logstash via UDP. Replace `DOCKER_HOST_IP` with the IP of the machine running your containers (use `127.0.0.1` if running locally).

```bash
# Add forwarding rule to rsyslog
echo '*.* @DOCKER_HOST_IP:5514' | sudo tee /etc/rsyslog.d/50-logstash.conf

# Restart rsyslog to apply
sudo systemctl restart rsyslog

# Generate a test auth failure
logger -p auth.warning "Failed password for invalid user testuser from 192.168.1.1 port 22 ssh2"
```

> **How forwarding works:** The `@` prefix in rsyslog means UDP. Use `@@` for TCP. The `*.*` wildcard forwards all facilities and severities — you can narrow this to `auth.*` once you've confirmed end-to-end flow.

---

### Step 7 — Create an index pattern in Kibana

Open Kibana at **http://localhost:5601** and follow these steps to make your syslog data searchable in Discover.

1. Navigate to **Stack Management → Index Patterns**
2. Click **Create index pattern**
3. Enter pattern: `syslog-*`
4. Set **Time field** to `@timestamp`
5. Click **Save index pattern**
6. Go to **Discover** — your syslog events appear in the timeline

> **No data showing?** Check that rsyslog is forwarding: run `docker compose logs logstash` — you should see incoming events printed via the `rubydebug` stdout codec. Also verify the time range in Kibana is set to "Last 15 minutes."

---

### Step 8 — Build the auth failures dashboard

Create a visualization to track SSH brute-force and failed sudo attempts — the kind of panel that lives in every real SOC dashboard.

1. Go to **Dashboards → Create dashboard**
2. Click **Add visualization → Lens**
3. Set index pattern to `syslog-*`
4. Add a KQL filter: `tags: "auth_failure"`
5. Chart type: **Bar vertical stacked**, X-axis = `@timestamp` (auto-interval), Y-axis = **Count**
6. Add a second panel: Data table with `logsource.hostname` terms to see which hosts generate failures
7. Save the dashboard as **Auth Failures Overview**

> **Portfolio tip:** Export the dashboard JSON (**Stack Management → Saved Objects → Export**) and commit it to your repo. This lets interviewers import and run your exact dashboard — far more impressive than screenshots alone.

---

## 🔌 Port Reference

| Service | Port | Protocol | Purpose |
|---|---|---|---|
| Elasticsearch | `9200` | TCP | REST API — health checks, index queries |
| Elasticsearch | `9300` | TCP | Cluster transport (internal, not exposed) |
| Logstash | `5514` | UDP | Syslog ingestion from rsyslog |
| Kibana | `5601` | TCP | Web UI — Discover, Dashboards, Lens |

---

## 🛠️ Common Troubleshooting

| Symptom | Fix |
|---|---|
| ES exits instantly | `vm.max_map_count` not set — re-run the sysctl command from Step 1 |
| Logstash won't start | Pipeline config syntax error — run `docker compose logs logstash` and look for `CONFIG ERROR` |
| No events in Kibana | Check rsyslog is running (`systemctl status rsyslog`) and that UDP 5514 isn't blocked by UFW |
| Index pattern not found | Logstash hasn't written any events yet — send a test log with `logger` and refresh |
| Out of memory | Reduce `ES_JAVA_OPTS` heap to `-Xms512m -Xmx512m` |

---

## 🚀 Next Steps

- [ ] Enable Elasticsearch security (TLS + user auth) for a production-grade deployment
- [ ] Add Filebeat on endpoints to ship logs without touching rsyslog
- [ ] Write a Kibana alert rule to email on >10 auth failures in 5 minutes
- [ ] Explore GeoIP enrichment in Logstash to map attacker origins
- [ ] Set index lifecycle management (ILM) to auto-delete logs older than 30 days

---

*Cybersecurity project · Stack: Elastic 8.13 · Docker Compose 2.x · Ubuntu 22.04*
