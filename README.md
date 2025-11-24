# Environmental Observability Stack

A comprehensive observability solution for monitoring energy consumption, carbon emissions, and environmental conditions in data center and home lab environments running on OpenShift/Kubernetes.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              EXTERNAL DATA SOURCES                                   │
├─────────────────────┬─────────────────────┬─────────────────────┬───────────────────┤
│                     │                     │                     │                   │
│   ┌─────────────┐   │   ┌─────────────┐   │   ┌─────────────┐   │   ┌───────────┐   │
│   │   Tibber    │   │   │ Electricity │   │   │    Adax     │   │   │    BMC    │   │
│   │    API      │   │   │    Maps     │   │   │    API      │   │   │  Redfish  │   │
│   │             │   │   │    API      │   │   │             │   │   │    API    │   │
│   │ - Prices    │   │   │             │   │   │ - Room Temp │   │   │           │   │
│   │ - Grid Zone │   │   │ - Carbon    │   │   │ - Heating   │   │   │ - Power   │   │
│   │             │   │   │   Intensity │   │   │   Status    │   │   │ - Temps   │   │
│   │             │   │   │ - Renewable │   │   │             │   │   │ - Fans    │   │
│   │             │   │   │   Mix       │   │   │             │   │   │ - Voltage │   │
│   └──────┬──────┘   │   └──────┬──────┘   │   └──────┬──────┘   │   └─────┬─────┘   │
│          │          │          │          │          │          │         │         │
└──────────┼──────────┴──────────┼──────────┴──────────┼──────────┴─────────┼─────────┘
           │                     │                     │                    │
           │                     │                     │                    │
           ▼                     ▼                     ▼                    │
┌──────────────────────────────────────────────────────────────────────┐   │
│                         DATA COLLECTOR                                │   │
│                      (CronJob - every 15min)                         │   │
│  ┌────────────────────────────────────────────────────────────────┐  │   │
│  │                     collect-data.py                            │  │   │
│  │                                                                │  │   │
│  │  • Fetches electricity prices from Tibber                      │  │   │
│  │  • Gets carbon intensity from ElectricityMaps                  │  │   │
│  │  • Retrieves room temperatures from Adax                       │  │   │
│  │  • Dynamic zone mapping (SE3 → SE-SE3)                         │  │   │
│  │                                                                │  │   │
│  └────────────────────────────────────────────────────────────────┘  │   │
└──────────────────────────────┬───────────────────────────────────────┘   │
                               │                                           │
                               ▼                                           │
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              OPENSHIFT CLUSTER                                       │
│  ┌───────────────────────────────────────────────────────────────────────────────┐  │
│  │                           GRAFANA NAMESPACE                                    │  │
│  │                                                                               │  │
│  │   ┌─────────────┐      ┌─────────────┐      ┌─────────────────────────────┐   │  │
│  │   │  InfluxDB   │◄─────│   Grafana   │─────►│        Dashboards           │   │  │
│  │   │             │      │             │      │                             │   │  │
│  │   │ - energy-   │      │ - Grafana   │      │  ┌─────────────────────┐    │   │  │
│  │   │   data      │      │   Operator  │      │  │  Kepler Dashboard   │    │   │  │
│  │   │   bucket    │      │             │      │  │  - Power metrics    │    │   │  │
│  │   │             │      │             │      │  │  - Energy cost      │    │   │  │
│  │   │ Stores:     │      │             │      │  │  - Carbon emissions │    │   │  │
│  │   │ - Prices    │      │             │      │  └─────────────────────┘    │   │  │
│  │   │ - Carbon    │      │             │      │                             │   │  │
│  │   │ - Temps     │      │             │      │  ┌─────────────────────┐    │   │  │
│  │   │             │      │             │      │  │  Dynamic Energy     │    │   │  │
│  │   └─────────────┘      │             │      │  │  - Real-time cost   │    │   │  │
│  │                        │             │      │  │  - Price forecasts  │    │   │  │
│  │                        │             │      │  └─────────────────────┘    │   │  │
│  │   ┌─────────────┐      │             │      │                             │   │  │
│  │   │ Prometheus  │◄─────│             │      │  ┌─────────────────────┐    │   │  │
│  │   │             │      │             │      │  │  BMC Hardware       │────┼───┼───┘
│  │   │ - Kepler    │      │             │      │  │  - Server temps     │    │   │
│  │   │   metrics   │      │             │      │  │  - Power usage      │    │   │
│  │   │             │      │             │      │  │  - Fan speeds       │    │   │
│  │   └─────────────┘      └─────────────┘      │  │  - Datacenter temp  │    │   │
│  │                                             │  └─────────────────────┘    │   │
│  │                                             │                             │   │
│  │                                             │  ┌─────────────────────┐    │   │
│  │                                             │  │  VM Energy          │    │   │
│  │                                             │  │  - Per-VM power     │    │   │
│  │                                             │  │  - Carbon per VM    │    │   │
│  │                                             │  └─────────────────────┘    │   │
│  │                                             └─────────────────────────────┘   │
│  └───────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
│  ┌───────────────────────────────────────────────────────────────────────────────┐  │
│  │                     OPENSHIFT-POWER-MONITORING NAMESPACE                       │  │
│  │   ┌─────────────────────────────────────────────────────────────────────────┐ │  │
│  │   │                         Kepler DaemonSet                                │ │  │
│  │   │              (via OpenShift Power Monitoring Operator)                  │ │  │
│  │   │                                                                         │ │  │
│  │   │   Runs on every node, measures power consumption using:                 │ │  │
│  │   │   • eBPF probes for process-level metrics                               │ │  │
│  │   │   • RAPL (Running Average Power Limit) for CPU/DRAM                     │ │  │
│  │   │   • Hardware counters and models                                        │ │  │
│  │   │                                                                         │ │  │
│  │   │   Exports metrics: kepler_container_*_total, kepler_node_*_total        │ │  │
│  │   └─────────────────────────────────────────────────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

## Data Flow

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│    Tibber    │     │ Electricity  │     │     Adax     │     │     BMC      │
│    GraphQL   │     │    Maps      │     │   REST API   │     │   Redfish    │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                    │                    │
       │ priceAreaCode      │ carbonIntensity    │ temperature        │ Real-time
       │ current.total      │ renewablePercent   │ heatingEnabled     │ via Infinity
       │ today[].total      │ fossilFreePercent  │                    │ datasource
       │                    │                    │                    │
       ▼                    ▼                    ▼                    │
┌─────────────────────────────────────────────────────────────┐      │
│                    Data Collector CronJob                    │      │
│                                                             │      │
│  1. Fetch Tibber data → Extract priceAreaCode (SE3)         │      │
│  2. Map to ElectricityMaps zone (SE3 → SE-SE3)              │      │
│  3. Fetch carbon intensity for zone                         │      │
│  4. Fetch Adax room temperatures                            │      │
│  5. Write all data to InfluxDB                              │      │
└──────────────────────────┬──────────────────────────────────┘      │
                           │                                         │
                           ▼                                         │
                    ┌─────────────┐                                  │
                    │  InfluxDB   │                                  │
                    │             │                                  │
                    │ Measurements:                                  │
                    │ • electricity_price                            │
                    │ • carbon_intensity                             │
                    │ • adax_temperature                             │
                    └──────┬──────┘                                  │
                           │                                         │
                           ▼                                         ▼
                    ┌─────────────────────────────────────────────────────┐
                    │                      Grafana                        │
                    │                                                     │
                    │  Datasources:                                       │
                    │  • InfluxDB (historical energy/carbon/temp data)    │
                    │  • Prometheus (Kepler power metrics)                │
                    │  • Infinity (BMC Redfish real-time)                 │
                    │  • Tibber (real-time prices via JSON)               │
                    │  • ElectricityMaps (real-time carbon)               │
                    └─────────────────────────────────────────────────────┘
```

## Components

### Data Sources

| Source | Type | Data Collected | Update Frequency |
|--------|------|----------------|------------------|
| **Tibber** | GraphQL API | Electricity prices, grid zone | 15 min |
| **ElectricityMaps** | REST API | Carbon intensity, renewable mix | 15 min |
| **Adax** | REST API | Room temperatures, heater status | 15 min |
| **BMC Redfish** | REST API | Server temps, power, fans | Real-time |
| **Kepler** | Prometheus | Container/node power consumption | Real-time |

### Dashboards

1. **Kepler Dashboard** - Power consumption metrics with cost and carbon calculations
2. **Dynamic Energy Dashboard** - Real-time energy pricing and forecasts
3. **VM Energy Dashboard** - Per-VM power consumption and carbon footprint
4. **BMC Hardware Dashboard** - Server hardware sensors and datacenter temperature

## Prerequisites

- OpenShift cluster
- Grafana Operator installed
- OpenShift Power Monitoring Operator installed (provides Kepler)
- API tokens for:
  - Tibber (https://developer.tibber.com/)
  - ElectricityMaps (https://api-portal.electricitymaps.com/)
  - Adax (via Adax WiFi app)
- BMC/IPMI access to server

## Installation

1. **Create namespace:**
   ```bash
   oc create namespace grafana
   ```

2. **Update secrets in `data-collector-cronjob.yaml`:**
   ```yaml
   stringData:
     TIBBER_TOKEN: "<YOUR_TIBBER_API_TOKEN>"
     ELECTRICITYMAPS_TOKEN: "<YOUR_ELECTRICITYMAPS_API_TOKEN>"
     INFLUXDB_TOKEN: "<YOUR_INFLUXDB_TOKEN>"
     ADAX_CLIENT_ID: "<YOUR_ADAX_CLIENT_ID>"
     ADAX_CLIENT_SECRET: "<YOUR_ADAX_CLIENT_SECRET>"
   ```

3. **Update BMC credentials in `bmc-datasource.yaml`:**
   ```yaml
   allowedHosts:
     - "https://<YOUR_BMC_IP_ADDRESS>"
   secureJsonData:
     basicAuthPassword: "<YOUR_BMC_PASSWORD>"
   basicAuthUser: "<YOUR_BMC_USERNAME>"
   ```

4. **Deploy all components:**
   ```bash
   oc apply -f influxdb-deployment.yaml
   oc apply -f grafana-instance.yaml
   oc apply -f prometheus-datasource.yaml
   oc apply -f influxdb-datasource.yaml
   oc apply -f tibber-datasource.yaml
   oc apply -f electricitymaps-datasource.yaml
   oc apply -f bmc-datasource.yaml
   oc apply -f adax-datasource.yaml
   oc apply -f data-collector-configmap.yaml
   oc apply -f data-collector-cronjob.yaml
   oc apply -f kepler-dashboard.yaml
   oc apply -f dynamic-energy-dashboard.yaml
   oc apply -f vm-energy-dashboard.yaml
   oc apply -f bmc-dashboard.yaml
   ```

5. **Trigger initial data collection:**
   ```bash
   oc create job --from=cronjob/energy-data-collector initial-run -n grafana
   ```

## Configuration

### ElectricityMaps Zone

The system automatically detects your electricity grid zone from Tibber's `priceAreaCode` and maps it to ElectricityMaps zones:

| Tibber Zone | ElectricityMaps Zone |
|-------------|---------------------|
| SE1-SE4 | SE-SE1 to SE-SE4 |
| NO1-NO5 | NO-NO1 to NO-NO5 |
| DK1-DK2 | DK-DK1 to DK-DK2 |
| FI | FI |

### Adax Room Filtering

By default, only the "garage" room is monitored (used as datacenter temperature). To change this, modify `process_adax_data()` in `data-collector-configmap.yaml`.

## Metrics

### InfluxDB Measurements

**electricity_price:**
- `total` - Total price (SEK/kWh)
- `energy` - Energy component
- `tax` - Tax component
- `level` - Price level (VERY_CHEAP, CHEAP, NORMAL, EXPENSIVE, VERY_EXPENSIVE)

**carbon_intensity:**
- `value` - gCO2eq/kWh
- `fossil_fuel_pct` - Percentage from fossil fuels
- `fossil_free_pct` - Percentage fossil-free
- `renewable_pct` - Percentage renewable

**adax_temperature:**
- `temperature` - Current temperature (°C)
- `target_temperature` - Target temperature (°C)
- `heating_enabled` - Heater status (0/1)

### Kepler Metrics (Prometheus)

- `kepler_container_joules_total` - Energy consumed per container
- `kepler_node_core_joules_total` - CPU core energy
- `kepler_node_dram_joules_total` - DRAM energy
- `kepler_node_platform_joules_total` - Total platform energy

## License

MIT

## Contributing

Contributions welcome! Please submit pull requests with any improvements or additional integrations.
