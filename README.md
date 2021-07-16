# Sense Energy Prometheus Exporter

---

This is a prometheus exporter for the [Sense](https://sense.com) energy monitor products.

Internally it uses the [Sense Energy Monitor API Interface](https://github.com/scottbonline/sense) client.

## Usage
Environment Variables available:

| Variable | Description | Options | Default |
|---|---|---|---|
| `EXPORTER_LOG_LEVEL` | Controls verbosity of logs | `DEBUG`, `INFO`, `WARNING`, `ERROR` | `INFO` |
| `SENSE_ACCOUNT_NAME_<n>` | The name to identify the account by. `<n>` is an integer that links the other env vars containing `<n>`. | | |
| `SENSE_ACCOUNT_USERNAME_<n>` | The username (email address) used to login to the account. `<n>` is an integer that links the other env vars containing `<n>`. | | |
| `SENSE_ACCOUNT_PASSWORD_<n>` | The password used to login to the account. `<n>` is an integer that links the other env vars containing `<n>`. | | |
| `TZ` | The timezone to use for the container. Use the string in `TZ database name` column of the [List of TZ Database Time Zones](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones). | | None (container defaults to UTC time)|
| `EXPORTER_PORT` | The port to listen on | | `9993` |
| `EXPORTER_BIND_HOST` | The address to listen on | | `0.0.0.0` |
| `EXPORTER_NAMESPACE` | The prefix to use for prometheus metrics | | `sense_energy` |
| `CONFIG_FILE` | The container filepath to the config file | | `/etc/sense_energy_prometheus_exporter/config.yaml` |


### Running locally
An example of using this exporter for 3 different Sense devices/accounts:

```shell
docker run --rm \
-e EXPORTER_LOG_LEVEL="DEBUG" \
-e SENSE_ACCOUNT_NAME_1="A/C" \
-e SENSE_ACCOUNT_USERNAME_1="account1@test.com" \
-e SENSE_ACCOUNT_PASSWORD_1='~!my_password_1!~' \
-e SENSE_ACCOUNT_NAME_2="Backyard Subpanel" \
-e SENSE_ACCOUNT_USERNAME_2="account2@test.com" \
-e SENSE_ACCOUNT_PASSWORD_2='~!my_password_2!~' \
-e SENSE_ACCOUNT_NAME_3="Home" \
-e SENSE_ACCOUNT_USERNAME_3="account3@test.com" \
-e SENSE_ACCOUNT_PASSWORD_3='~!my_password_3!~' \
-e TZ="America/Denver" \
-it --network bridge \
-p"9993:9993" \
--mount type=bind,source="$(pwd)"/volumes,target=/etc/sense_energy_prometheus_exporter \
ejsuncy/sense_energy_prometheus_exporter:latest
```

Then visit in your browser:
```shell
http://0.0.0.0:9993/metrics
```

You can find sample metric output in [data/metrics.txt](data/metrics.txt).

### Running on kubernetes
A sample manifest:
```yaml
# Service
apiVersion: v1
kind: Service
metadata:
  name: prometheus-exporters-sense
spec:
  type: LoadBalancer
  ports:
    - name: sense
      port: 80
      targetPort: 9993
      protocol: TCP
  selector:
    app: prometheus-exporters-sense
---

# Secrets
apiVersion: v1
kind: Secret
metadata:
  name: secrets-sense
type: Opaque
data:
  # obtain b64 encoded password with `echo -n "<password>" | b64`
  SENSE_ACCOUNT_PASSWORD_1: "<b64-encoded password here>"
  SENSE_ACCOUNT_PASSWORD_2: "<b64-encoded password here>"
  SENSE_ACCOUNT_PASSWORD_3: "<b64-encoded password here>"
---
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: prometheus-exporters-sense
  name: prometheus-exporters-sense
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus-exporters-sense
  template:
    metadata:
      labels:
        app: prometheus-exporters-sense
    spec:
      containers:
      - image: ejsuncy/sense_energy_prometheus_exporter:latest
        name: sense
        resources:
          limits:
            cpu: 1000m
            memory: 1500Mi
          requests:
            cpu: 500m
            memory: 1G
        ports:
          - name: http
            containerPort: 9993
            protocol: TCP
        env:
          - name: EXPORTER_LOG_LEVEL
            value: DEBUG
          - name: SENSE_ACCOUNT_NAME_1
            value: A/C
          - name: SENSE_ACCOUNT_USERNAME_1
            value: account1@test.com
          - name: SENSE_ACCOUNT_NAME_2
            value: Backyard Subpanel
          - name: SENSE_ACCOUNT_USERNAME_2
            value: account2@test.com
          - name: SENSE_ACCOUNT_NAME_3
            value: Home
          - name: SENSE_ACCOUNT_USERNAME_3
            value: account3@test.com
          - name: TZ
            value: America/Denver
        envFrom:
          - secretRef:
              name: secrets-sense
```

Of course, you should fine-tune the memory and cpu requests and limits.
Also note that multiple replicas for High Availability (HA) may get you rate-limited or 
have your connections dropped since there's currently no built-in leader election.

You might configure your prometheus scrape settings in a configmap as such:
```yaml
apiVersion: v1
data:
  prometheus_config: |
    global:
      scrape_interval: 15s
      external_labels:
        monitor: 'k8s-prom-monitor'
    scrape_configs:
      - job_name: 'sense'
        metrics_path: /metrics
        scrape_interval: 3m
        scrape_timeout: 1m
        static_configs:
          - targets:
              - 'prometheus-exporters-sense'
kind: ConfigMap
metadata:
  name: prometheus-config
```

**Again, be careful of the scrape interval.** The internal client does some self-throttling but make sure you don't
swamp the sense servers.

## Development
### Building
Assume all development starts with this virtual environment:
```shell
make venv
source .venv/bin/activate
```

### Containerizing
Clone this repository and containerize for your machine, tagging the image however you want:
```shell
docker build -t ejsuncy/sense_energy_prometheus_exporter:latest .
```

Or build for other architectures (for example, if you're developing on an ARM mac but deploying to AMD linux kubernetes):
```shell
docker buildx build -t ejsuncy/sense_energy_prometheus_exporter:latest --platform linux/amd64 .
```

### Releasing & Publishing
The [github cli](https://github.com/cli/cli) is required for making releases.
Note: this will make a release for the current state of the default branch in origin, so commit and push
changes before making the release.

Tag, build, and push for multiple arch with the release tag listed in [VERSION.txt](VERSION.txt):
```shell
make release
```