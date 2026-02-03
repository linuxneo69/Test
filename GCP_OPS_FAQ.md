You can bridge your local environment with **Google Cloud Observability (GCO)** using **Google Cloud Managed Service for Prometheus (GMP)**.

If you have a Prometheus datasource URL and user access to Grafana, you are halfway there. However, to fully "connect" them, you will need to set up a data bridge.

---

## The Workflow Overview

To see your in-house metrics and dashboards in GCP, you follow a two-part process:

1. **Data Migration:** Send your local Prometheus metrics to GCP via **Remote Write**.
2. **Dashboard Migration:** Import your local Grafana dashboard JSON files into **Cloud Monitoring**.

---

## Part 1: Exporting Metrics to GCP (Data Link)

Since GCP cannot "pull" metrics from your internal URL (due to firewalls/security), your local Prometheus must "push" them to GCP.

### 1. Setup GCP Authentication

* In the GCP Console, create a **Service Account**.
* Assign it the role: `roles/monitoring.metricWriter`.
* Generate and download a **JSON Key** for this service account.

### 2. Configure Prometheus "Remote Write"

You (or your admin) will need to add a small configuration block to your local `prometheus.yml` file. This tells Prometheus to copy all data to Google's cloud storage.

```yaml
remote_write:
- url: https://monitoring.googleapis.com/v1/projects/YOUR_PROJECT_ID/location/global/prometheus/api/v1/write
  authorization:
    credentials_file: /path/to/your/service-account-key.json

```

> **Note:** If you cannot edit the Prometheus config, you can use the **GCP Ops Agent** on a local machine to scrape that "Prometheus datasource URL" you have and forward it to GCP.

---

## Part 2: Import Grafana Dashboards to GCO

Google Cloud Monitoring has a built-in importer that converts Grafana JSON into native GCP widgets.

### 1. Export from Grafana

1. Open your dashboard in Grafana.
2. Click the **Share** icon > **Export** tab.
3. Click **Save to file** to get the JSON.

### 2. Import to GCP

1. In GCP Console, go to **Monitoring** > **Dashboards**.
2. Click **Import Dashboard** at the top.
3. Upload your Grafana JSON file.
4. GCO will automatically attempt to map your Grafana panels to GCP widgets.

---

## Part 3: Testing & Creating Widgets

Once your data is flowing, you can create new widgets directly in GCP:

* **Use PromQL:** GCP Monitoring now supports **PromQL** natively. You don’t need to learn Google’s proprietary language (MQL).
* **Metrics Explorer:** Go to **Monitoring** > **Metrics Explorer**, select **PromQL** as the language, and type `up` or any of your local metric names. They will appear with the prefix `prometheus.googleapis.com/`.

---

## Requirements Checklist

| Component | Access Needed |
| --- | --- |
| **GCP Project** | `Editor` or `Monitoring Admin` permissions. |
| **Grafana** | `Viewer` access is enough to export JSON, but `Editor` is better for managed exports. |
| **Prometheus** | Access to the `prometheus.yml` file OR a server to host an Ops Agent. |

---

### Important Limitation

* **Alerts:** Grafana Alerts do not import automatically. You will need to recreate your alert logic using **Cloud Monitoring Alerting Policies**.
* **Plugins:** Custom Grafana plugins (like specialized 3D maps or unique community panels) might not render in GCP and will show as basic charts.
