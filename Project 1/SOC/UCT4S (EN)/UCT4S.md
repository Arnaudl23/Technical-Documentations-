# UCT4S (EN)

# Context & Objectives

In a SIEM environment, the **reliability of detection rules** is a crucial issue.

Splunk updates, indexing modifications, new pipelines, infrastructure changes or log format changes can lead to detection problems.

To meet this need, we have designed **UCT4S (Use Case Tester for Splunk):**

An on-premise, CI/CD-oriented platform capable of automatically injecting realistic synthetic logs (generated via **EventNG**), running the associated SPL searches, and validating that usecases are still working as expected.

The final objective is to integrate this validation into an **internal CI/CD pipeline (Forgejo)** so that **each usecase modification triggers an automatic end-to-end validation**.

# Overall project architecture

```bash
UCT4S/
│
├── config.yaml
├── requirements.txt
│
├── .forgejo/
│   └── workflows/
│       └── usecase.yml           # Pipeline CI/CD
│
├── scripts/
│   ├── test_usecases.py          # Execute all usecases
│   ├── run_single_usecase.py     # Execute one usecase only
│   └── eventng/
│       ├── eventng.py            # Generate synthetic logs
│       └── tokens/
│           ├── users.txt
│           ├── hostnames.txt
│           └── ...
│
├── usecases/
│   ├── smoke_basic.yml
│   ├── ad_bruteforce.yml
│   └── ...
│
├── templates/
│   ├── bruteforce/
│   │   ├── bruteforce.log
│   │   └── bruteforce.yml
│   └── ...
│
└── webui/
    ├── app.py                    # Link beetween backend and Web UI
    ├── templates/
    │   └── add_usecase.html      # Web Page 
    └── static/
        └── css/style.css

```

# Preparing Splunk

## Creating an API REST account

To create a dedicated user, go to `Settings → Access Controls → Users → New User`

| Username | Password | Role |
| --- | --- | --- |
| api-rest | ****************** | Power |

**Connectivity test :**

```bash
curl -k -u api-rest:PASSWORD \
  https://spunk.soc.labo:8089/services/server/info?output_mode=json
```

## HEC token activation and creation

### Activate HEC

Go to `Settings → Data Inputs → HTTP Event Collector → Global Settings` and enter the following:

| Parameter | Value |
| --- | --- |
| **All Tokens** | `Enabled` |
| **HTTP Port Number** | `8088` |

### Create token

Go to `Settings → Data Inputs → HTTP Event Collector → New Token` and enter the following:

| Parameter | Value |
| --- | --- |
| Name | `UCT4S` |
| Indexer Acknowledgement | `No` |
| Description | `Token use by UCT4S` |
| Source type | `uct4s:fallback` |
| Allowed Indexes | `empty` |

**Test:**

```bash
curl -k -H "Authorization: Splunk ************************************" \
  -d '{"event":"hello","sourcetype":"ci:test"}' \
  http://spunk.soc.labo:8088/services/collector
```

Expected response:

```bash
{"text":"Success","code":0}
```

### Creating the test index

Go to `Settings → Indexes → New Index` to create the dedicated index: `usecase_test`

# Synthetic log generation

## Why EventNG

Originally, our UCT4S platform was to use **Splunk EventGen**, the official tool for generating synthetic logs from templates.

However, EventGen poses several problems in modern environments:

- Incompatibility with recent versions of Python
- Obsolete dependencies
- Incomplete documentation

After several unsuccessful attempts to integrate EventGen into our CI/CD pipeline, we decided to **develop a minimalist home-grown generator** in Python, tailored to our needs: **EventNG**.

## General operation of EventNG

### 1. Reading a **real log file**

EventNG requires **no special** log **syntax**. All you need is a text file containing real Windows, Linux, network logs, etc.

Example: 

```python
12/05/2025 09:28:08.711 AM LogName=Security EventCode=4625 ...
Source Network Address: 10.0.0.1
...
```

### 2. Application of transformation rules

EventNG will intelligently modify various fields in the logs according to what is specified in the template configuration file in YAML**.** 

**Example of template :**

```bash
template:
  sample_file: bruteforce.log

time:
  enable: true
  regex: '^(.*?)\sLogName='
  format: "%m/%d/%Y %I:%M:%S.%f AM"
  step_seconds: 1

tokens:
  - regex: "Source Network Address:\\s*(\\S+)"
    type: subnet-static
    subnet: "172.20.10.0/24"

  - regex: "Account Name:\\s*(administrator@ad\\.infra\\.lab)"
    type: "list-static"
    source: "users.txt"
```

EventNG features transformation functions such as : 

- **Timestamp modification**:
    
    EventNG extracts the timestamp via a regex, then applies :
    
    - a current timestamp
    - a progressive increment (e.g. +1 second per event)
- **Flexible token system:**
    
    EventNG uses regex rules to dynamically replace certain values.
    
    | Type | Description |
    | --- | --- |
    | **static** | Replaces with a fixed value |
    | **list** | Selects a random element from a file containing a list`(tokens/users.txt`) |
    | **list-static** | Ditto, but sets the value for the entire log batch |
    | **subnet** | Generates a random IP in a subnet (e.g. `172.20.20.0/24`) |
    | **subnet-static** | Ditto, but sets value for all logs |

# UCT4S : Usecase test engine

## Main scripts

UCT4S consists of two scripts:

| Script | Role |
| --- | --- |
| `test_usecases.py` | Executes a test for all usecases |
| `run_single_usecase.py` | Runs a test for a single usecase |

## Configuration file

The main UCT4S configuration file is `config.yaml`:

```bash
splunk:
  hec_url: "http://192.168.1.1:8088/services/collector"
  hec_token: "********"
  hec_tls_verify: false

  rest_url: "https://192.168.1.1:8089"
  username: "api-rest"
  password: "********"
  rest_tls_verify: false

  test_index: "usecase_test"
```

## Test life cycle

### **1. Loading the configuration**

The various tokens and identifiers present in `config.yaml` are loaded for connection to Splunk.

### **2. Reading the usecase YAML file**

Each UCT4S usecase is described by **a unique YAML file**. This file defines :

- usecase metadata:
    
    
    | Field | Description |
    | --- | --- |
    | `id` | Unique usecase identifier (used in logs, reports, CIs) |
    | `name` | Readable name of the usecase |
    | `description` | Functional description of the scenario tested |
    | `enabled` | Enables or disables usecase without deleting the file |
- Which EventNG template is used,
- In which index logs are injected and with which options (host, sourcetype)
- Which SPL search is performed,
- What result is expected to validate the detection.

**Example of usecase (ex: `ad_bruteforce.yml`) :** 

```bash
id: ad_bruteforce
name: "Bruteforce detection"
description: "Simulated detection of bruteforce on active dirctory"

enabled: true

data:
  mode: "eventng"
  template: "templates/bruteforce/bruteforce.yml"
  index: "usecase_test"
  sourcetype: "WinEventLog:Security"
  host: "uct4s"

splunk:
  earliest: "-10m"
  latest: "now"
  search: |
    index="usecase_test"
    | rex "EventCode=(?<EventCode>\d+)"
    | rex "Account Name:\s*(?<Account_Name>[^ ]+)"
    | rex "Source Network Address:\s*(?<Source_Network_Address>[^\s]+)"
    | search EventCode=4625 Source_Network_Address!=""
    | stats count max(_time) as last_attempt_time by Account_Name Source_Network_Address
    | where count > 10
    | stats count
expectation:
  min_count: 1
```

### **3. Log injection into Splunk (HEC)**

UCT4S asks EventNG to **generate synthetic events**, then sends them to Splunk.

### **4. Synchronization: wait until Splunk has indexed the logs**

To avoid false negatives, UCT4S performs an "indexing barrier":

- It sends a **special marker event**, such as :
    
    ```bash
    uct4s_sync = "uct4s_marker_173846..."
    ```
    
- It queries Splunk until this event appears in the targeted index.

This ensures that when the SPL search starts, all usecase logs are actually indexed.

### **5. Executing the SPL search**

UCT4S executes the search via Splunk's REST API. 

The `earliest` / `latest` parameters in the usecase file define the analysis window.

### **6. Result retrieval and evaluation**

UCT4S compares the number of events detected by Splunk with the expected minimum:

- **SUCCESS** : The search has identified at least `min_count` events.
- **FAILED** : The detection did not return the expected result.

# Basic validation usecase

The `smoke_test` usecase is used to check **3 basic things**:

1. **Is Splunk HEC working?**
    
    Can an event be sent via `/services/collector`?
    
2. **Does the Splunk REST API work?**
    
    Can a search be launched via `/services/search/jobs`?
    
3. **Is the event sent via HEC indexed?**
    
    Can this event be found using a simple SPL search?
    
    ```python
    index=usecase_test hec_smoke_test=1
    ```
    

# CI/CD with Forgejo

## Objective

- Automatically trigger UCT4S tests,
- Refuse a usecase if tests fail,
- Guarantee detection quality over time.
- Allow the addition of a usecase via an intuitive web interface.

## Installing Forgejo

### Update and basic tools

We start by updating the machine and installing the necessary tools:

```bash
sudo dnf update -y
sudo dnf install -y git vim curl wget tar
```

### Creating the `forgejo` system user

Create a dedicated system user for Forgejo. This will isolate the service and prevent it from running as `root`.

```bash
sudo useradd \
  --system \
  --shell /bin/bash \
  --comment 'Forgejo Git Service' \
  --home-dir /var/lib/forgejo \
  forgejo
```

### Creating working directories

Prepare data and configuration directories, with appropriate rights:

```bash
# Forgejo data (repo, DB, logs, etc.)
sudo mkdir -p /var/lib/forgejo
sudo chown forgejo:forgejo /var/lib/forgejo

# System config (optionnal)
sudo mkdir -p /etc/forgejo
sudo chown forgejo:forgejo /etc/forgejo
```

### Installing the Forgejo binary

Place the binary in `/usr/local/bin` so that it's in the `PATH`:

```bash
sudo mkdir -p /usr/local/bin
cd /usr/local/bin

sudo curl -L \
https://code.forgejo.org/forgejo/forgejo/releases/download/v10.0.0/forgejo-10.0.0-linux-amd64 \
-o forgejo

sudo chmod +x /usr/local/bin/forgejo
```

### Preparing the Forgejo data structure

Create the necessary directories: 

```bash
sudo mkdir -p /var/lib/forgejo/custom/conf
sudo chown -R forgejo:forgejo /var/lib/forgejo
```

Create a file `/var/lib/forgejo/custom/conf/app.ini`: 

```bash
[server]
APP_DATA_PATH = /var/lib/forgejo/data
DOMAIN = localhost
HTTP_PORT = 3000
ROOT_URL = http://localhost:3000/
SSH_DOMAIN = localhost
DISABLE_SSH = false
RUN_MODE = prod

[database]
DB_TYPE = sqlite3
PATH = /var/lib/forgejo/data/forgejo.db

[repository]
ROOT = /var/lib/forgejo/data/repositories

[log]
ROOT_PATH = /var/lib/forgejo/log
```

Then :

```bash
sudo chown forgejo:forgejo /var/lib/forgejo/custom/conf/app.ini
```

Make sure the entire tree structure exists:

```bash
sudo mkdir -p /var/lib/forgejo/data
sudo mkdir -p /var/lib/forgejo/data/repositories
sudo mkdir -p /var/lib/forgejo/log
sudo mkdir -p /var/lib/forgejo/custom/conf
sudo chown -R forgejo:forgejo /var/lib/forgejo
sudo restorecon -RF /var/lib/forgejo  # If SELinux enforcing
```

### Creating the systemd Forgejo service

Create a file `/etc/systemd/system/forgejo.service` and add the following:

```bash
[Unit]
Description=Forgejo - Self-hosted Git Service
After=network.target

[Service]
Type=simple
User=forgejo
Group=forgejo
WorkingDirectory=/var/lib/forgejo
Environment=USER=forgejo HOME=/var/lib/forgejo FORGEJO_WORK_DIR=/var/lib/forgejo
ExecStart=/usr/local/bin/forgejo web --work-path /var/lib/forgejo
Restart=always

[Install]
WantedBy=multi-user.target
```

- `FORGEJO_WORK_DIR` + `-work-path` tell Forgejo where to find data (data, config, logs).
- `WorkingDirectory=/var/lib/forgejo`: the process runs in this directory.

Activate and start the service: 

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now forgejo
```

 At this point, Forgejo starts with its default config.

## Installing Forgejo-Runner

The **Forgejo Runner** is the CI/CD pipeline execution agent.

- It retrieves code from the repository
- installs the required environment (Python, dependencies)
- Automatically executes the steps defined in the pipeline (Splunk usecase tests).

It validates each usecase in an isolated, reproducible and automated way, without human intervention. The runner's results determine whether a modification is accepted or rejected.

### Creating the `forgejo-runner` user

A dedicated system user is created for Forgejo. This allows the service to be isolated and not run as `root`.

```bash
sudo useradd \
  --system \
  --shell /bin/bash \
  --comment 'Forgejo Git Service' \
  --home-dir /var/lib/forgejo \
  forgejo
```

### Download the runner binary

```bash
mkdir -p /var/lib/forgejo-runner/bin
cd /var/lib/forgejo-runner/bin
curl -L https://code.forgejo.org/forgejo/runner/releases/download/v3.3.0/forgejo-runner-3.3.0-linux-amd64 -o forgejo-runner
chmod +x forgejo-runner
```

### Configuration

To generate a default configuration, run this command from `/var/lib/forgejo-runner`:

```bash
./bin/forgejo-runner generate-config > /var/lib/forgejo-runner/config.yml
```

Then edit the file `/var/lib/forgejo-runner/config.yml` and add : 

```yaml
log:
  level: info

runner:
  name: "runner-lab"
  token: "TOKEN"                       # Retrieved from Forgejo
  url: "http://192.168.1.100:3000"     # Forgejo URL
  labels:
    - "docker"
```

### Save runner

```bash
cd /var/lib/forgejo-runner
./bin/forgejo-runner register
```

And enter this:

| Parameter | Value |
| --- | --- |
| **Forgejo instance URL** | `http://192.168.1.100:3000` |
| **Runner token** | `YOUR_TOKEN` |
| **Runner name** | `runner-lab` |
| **Runner labels** | `docker` |

### Launch the runner daemon

Still in `/var/lib/forgejo-runner`:

```bash
./bin/forgejo-runner daemon --config /var/lib/forgejo-runner/config.yml
```

Create the file `/etc/systemd/system/forgejo-runner.service` and add the following: 

```bash
[Unit]
Description=Forgejo Actions Runner
After=network.target

[Service]
User=forgejo-runner
Group=forgejo-runner
WorkingDirectory=/var/lib/forgejo-runner
ExecStart=/var/lib/forgejo-runner/bin/forgejo-runner daemon --config /var/lib/forgejo-runner/config.yml
Restart=always

[Install]
WantedBy=multi-user.target
```

## Web UI UCT4S

The Web UI allows you to :

- view existing usecases,
- create new usecases using a form,
- test one or more usecases,
- automatic validation before commit,
- automated Git commit/push on success.

### How it works

- If a test fails: the YAML is deleted,
- If test passes: `git add / commit / push`

### Installation of web interface dependencies

```bash
sudo dnf install python3-pip -y
python3 -m pip install flask pyyaml
```

### HTML page

The `webui/templates/add_usecase.html` page:

- Displays the list of existing usecases in a **scrollable table** (container with scrollbar, page no longer stretches even with many usecases),
- Displays Flask **error/success messages**,
- Features :
    - one button to **test all usecases at once**,
    - one button per line to **test an existing usecase** individually,
    - a form for **creating a new usecase**,
- Incorporates a **dark/light theme** with memorization of choice in `localStorage`,
- Displays a **loading overlay** when sending forms.

### Flask backend

This Flask script`(webui/app.py`) now handles several usecase-related functions:

1. Loads the list of existing YAML usecases,
2. loads and saves **test statuses** in a `usecase_statuses.yml` file 
3. Displays the HTML page `add_usecase.html` with :
    - Usecase table,
    - Success/error messages,
    - Test statuses,
4. When the form is submitted, depending on the action requested:
    - **Test all existing usecases**,
    - **Test an existing usecase**,
    - **Create and test a new usecase**,
    
    And updates statuses in `usecase_statuses.yml`,
    

### Creating a token in Forgejo

In the Forgejo interface:

1. `Settings → Applications → Personal access tokens → New token.`
2. Give a name (e.g. `webui`) and scopes **write** on repositories.
3. Validate and **copy the token**.

### Store credentials on the server

On the server, with the user launching Flask:

```bash
git config --global credential.helper store
```

Edit the `~/.git-credentials` file, adapting :

```bash
http://YOUR_USER:YOUR_TOKEN@192.168.1.100:3000/forgejo/UCT4S.git
```

### Create the systemd service for the Web UI

Create the file `/etc/systemd/system/uct4s-webui.service`: 

```bash
[Unit]
Description=UCT4S Usecase Web UI (Flask)
After=network.target

[Service]
User=admin
Group=admin
WorkingDirectory=/home/admin/UCT4S
ExecStart=/usr/bin/python3 /home/admin/UCT4S/webui/app.py
Restart=always
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
```

### Enable and start the Web UI

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now uct4s-webui
sudo systemctl status uct4s-webui
```
