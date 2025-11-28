To provide you with a working Proof of Concept (POC) for the complete DevSecOps monitoring solution with GitHub Actions, I'll walk through each step with examples. This will cover:

1. **GitHub Action Template**
2. **App Repo Workflow**
3. **GitHub Runner Configuration**
4. **Database Configuration for Metrics**
5. **API Integration**
6. **Monitoring Dashboard Configuration**

Each example will have code and steps for you to deploy and demonstrate the solution. I will walk through the implementation to ensure everything is self-contained for a working POC.

### 1. **GitHub Action Template (Centralized)**

This is the centralized GitHub Actions template stored in a GitHub organization that other repos will use.

```yaml
# .github/workflows/ci-template.yml
name: CI Template

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v2

      - name: Set up JDK 11
        uses: actions/setup-java@v2
        with:
          java-version: '11'
      
      - name: Build
        run: ./gradlew build

      - name: Unit Test
        run: ./gradlew test

      - name: SonarQube Scan
        uses: sonarsource/sonarcloud-github-action@master
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      
      - name: JFrog Upload
        run: jfrog rt upload "build/libs/*" "my-repo/"

      - name: Optional - JFrog X-Ray
        if: ${{ secrets.JFROG_XRAY == 'true' }}
        run: jfrog rt xr "my-repo/*"  # Run optional X-Ray scan

      - name: Deployment
        if: ${{ secrets.DEPLOYMENT == 'true' }}
        run: ./deploy.sh  # Customize deployment step

```

### Key Details:

* **Mandatory stages**: Checkout, Build, Unit Test, SonarQube, JFrog Upload.
* **Optional stages**: JFrog X-Ray, Deployment (conditionally included).
* You can control optional stages with `secrets` in GitHub.

### 2. **App Repo Workflow (Using Template)**

In each app repository, the workflow will reference this centralized template.

```yaml
# .github/workflows/app-repo-workflow.yml
name: App Workflow

on:
  push:
    branches:
      - main

jobs:
  deploy:
    uses: my-org/ci-template/.github/workflows/ci-template.yml@main  # Use the centralized template
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      JFROG_XRAY: ${{ secrets.JFROG_XRAY }}
      DEPLOYMENT: ${{ secrets.DEPLOYMENT }}
```

### Key Details:

* The app repo simply references the centralized template, and environment-specific secrets (like `SONAR_TOKEN`) are passed into it.

### 3. **GitHub Runner Configuration**

Self-hosted runners can be used for greater control over resources and scaling.

1. **Create a runner configuration:**

   On your runner machine (e.g., an EC2 instance, Azure VM, or local server), run the following commands:

```bash
# 1. Download GitHub Runner
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64-2.278.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.278.0/actions-runner-linux-x64-2.278.0.tar.gz
tar xzf ./actions-runner-linux-x64-2.278.0.tar.gz

# 2. Configure Runner
./config.sh --url https://github.com/your-org --token YOUR_TOKEN

# 3. Start the runner
./run.sh
```

2. **Configure the GitHub Runner on the Repo:**

   * In your GitHub repo, go to **Settings** > **Actions** > **Runners**.
   * Add a self-hosted runner and obtain the `YOUR_TOKEN` from this page.

The runner will automatically register with GitHub and start executing workflows when triggered.

### 4. **Database Configuration for Metrics**

You can store metrics in **InfluxDB** (time-series database) or **PostgreSQL** (relational database). I'll show you how to set up **PostgreSQL** and integrate it into the solution.

1. **Create PostgreSQL Database**:

```bash
# Create a PostgreSQL database and user
sudo -u postgres psql
CREATE DATABASE github_metrics;
CREATE USER github_user WITH ENCRYPTED PASSWORD 'yourpassword';
GRANT ALL PRIVILEGES ON DATABASE github_metrics TO github_user;
```

2. **Connect to PostgreSQL**:
   Create a table to store pipeline metrics (e.g., DORA metrics, pipeline times).

```sql
-- Create a table for storing pipeline metrics
CREATE TABLE pipeline_metrics (
    id SERIAL PRIMARY KEY,
    repo_name TEXT,
    workflow_name TEXT,
    deployment_frequency INT,
    lead_time INT,
    change_failure_rate FLOAT,
    restore_time INT,
    pipeline_duration INT,
    deployment_platform TEXT,
    stage_name TEXT,
    stage_status TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

3. **PostgreSQL Integration for Data Collection**:

In your GitHub Actions templates, add steps to log metrics to PostgreSQL after each workflow run.

```yaml
- name: Log Metrics to PostgreSQL
  run: |
    psql -h ${{ secrets.PG_HOST }} -U ${{ secrets.PG_USER }} -d github_metrics -c "
    INSERT INTO pipeline_metrics (repo_name, workflow_name, deployment_frequency, lead_time, change_failure_rate, restore_time, pipeline_duration, deployment_platform, stage_name, stage_status)
    VALUES ('${{ github.repository }}', '${{ github.workflow }}', ${DEPLOYMENT_FREQUENCY}, ${LEAD_TIME}, ${CHANGE_FAILURE_RATE}, ${RESTORE_TIME}, ${PIPELINE_DURATION}, '${{ deployment_platform }}', '${{ stage_name }}', '${{ stage_status }}');
    "
```

### 5. **API Integration to Fetch Data**

You will use GitHub's REST and GraphQL APIs to fetch deployment, workflow, and metrics data.

1. **REST API to Fetch Workflow Runs:**

```bash
curl -H "Authorization: token YOUR_GITHUB_TOKEN" \
  "https://api.github.com/repos/your-org/your-repo/actions/runs"
```

2. **GraphQL API to Fetch Detailed Workflow Data:**

```graphql
query {
  repository(owner: "your-org", name: "your-repo") {
    workflows(first: 10) {
      nodes {
        id
        name
        run {
          id
          status
          conclusion
        }
      }
    }
  }
}
```

This query allows you to pull detailed workflow status and metrics.

### 6. **Monitoring Dashboard Configuration**

Here, we'll use **Power BI** or **Grafana** for visualizing the data collected from the GitHub workflows.

#### Power BI Setup:

1. Connect Power BI to **PostgreSQL** by setting up the PostgreSQL connector:

   * Go to **Home** > **Get Data** > **PostgreSQL Database**.
   * Enter connection details (e.g., `localhost` for the host, `github_metrics` for the database name).
   * Import the `pipeline_metrics` table into Power BI.
2. **Build the Dashboard**:

   * **DORA Metrics**: Use the `deployment_frequency`, `lead_time`, and `change_failure_rate` fields to create a report.
   * **Deployment Trends**: Group by `deployment_platform` and display frequency over time.
   * **Pipeline Duration**: Visualize the average pipeline time using the `pipeline_duration` field.

#### Grafana Setup:

1. **Install Grafana**:

```bash
# On Ubuntu:
sudo apt-get install -y grafana
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

2. **Connect Grafana to PostgreSQL**:

   * Go to **Grafana UI** and configure **PostgreSQL** as a data source.
   * Use the same connection details you used in Power BI.

3. **Create Grafana Dashboards**:

   * Import the **Pipeline Metrics** data from PostgreSQL into Grafana.
   * Set up **Time Series** and **Bar** charts for DORA metrics and deployment trends.

### Final POC Demo

* **GitHub Action Template**: Defines your CI/CD steps.
* **App Repo**: Demonstrates how the app repo uses the template.
* **GitHub Runner**: Configured to execute workflows.
* **PostgreSQL Database**: Collects and stores metrics.
* **API Integration**: Fetches detailed workflow data.
* **Monitoring Dashboard**: Displays DORA metrics and pipeline performance.

This setup provides a working demonstration of the solution. You can scale it by adding more repositories and workflows while keeping the monitoring solution flexible and centralized.
