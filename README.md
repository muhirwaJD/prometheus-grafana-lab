# Prometheus & Grafana Monitoring Lab

## 📁 Repository Structure

```
.
├── README.md
├── services/
│   ├── docker-compose.yml
│   └── prometheus.yml
├── screenshots/
│   ├── docker-ps.png
│   ├── prometheus-targets.png
│   ├── grafana-datasource.png
│   ├── grafana-panels.png
│   └── grafana-variable.png
└── answers/
    └── assessment-questions.md
```

---

## 🛠️ Setup Instructions

### 1. Clone the Repo

```bash
git clone https://github.com/muhirwaJD/prometheus-grafana-lab.git
cd prometheus-grafana-lab
```

### 2. Navigate to the Compose Directory

```bash
cd services
```

### 3. Launch the Stack

```bash
sudo docker-compose up -d
```

### 4. Access the Services

* **Prometheus**: [http://localhost:9090](http://localhost:9090)
* **Node Exporter**: [http://localhost:9100](http://localhost:9100)
* **Grafana**: [http://localhost:3000](http://localhost:3000)

  * Login: `admin / admin`

---

## 📊 Data Source Configuration

In Grafana:

* Go to **Configuration → Data Sources**
* Choose **Prometheus**
* Set URL to: `http://prometheus:9090`
* Click **Save & Test**

---

## 🧠 Dashboards

* Uptime (Stat panel)
* CPU & Memory (Time Series)
* Disk Usage (Gauge with thresholds)
* Dashboard Variable: `$job`

---

## ✅ Screenshots Provided
```
screenshots/assessment-questions.md
```

* Docker containers running
* Prometheus targets (UP)
* Grafana panels created
* Grafana data source configuration
* Variable dropdown in Grafana

---

## 📘 Assessment Answers

All detailed, original answers are located in:

```
answers/assessment-questions.md
```

Each section includes personal observations and configuration insights based on hands-on experience.

---

## 🔒 Notes

* All ports are exposed only for lab testing. In production, secure with HTTPS, auth, and firewalls.
* Secrets and credentials should be managed with Docker Secrets or .env files.

---
