You're right — I missed including the `set-output` command for capturing and tracking the success/failure of each stage. Here's how you can implement it in your GitHub Actions template.

The `set-output` command is used to pass information from one step to another within the same job. You can use it to capture the status of each stage (whether it succeeded or failed) and also to log it to your monitoring system (PostgreSQL, for example).

### Revised Example with `set-output` for Tracking Stage Success/Failure:

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
        id: checkout
        run: |
          echo "Checking out repository..."
        continue-on-error: false
        # Capture the status of the checkout step
        - name: Set Checkout Status
          id: checkout_status
          run: echo "::set-output name=status::$(if [ $? -eq 0 ]; then echo 'success'; else echo 'failure'; fi)"

      - name: Set up JDK 11
        uses: actions/setup-java@v2
        with:
          java-version: '11'
        id: setup_jdk
        run: |
          echo "Setting up JDK 11..."
        continue-on-error: false
        - name: Set JDK Setup Status
          id: setup_jdk_status
          run: echo "::set-output name=status::$(if [ $? -eq 0 ]; then echo 'success'; else echo 'failure'; fi)"

      - name: Build
        run: ./gradlew build
        id: build_step
        continue-on-error: false
        - name: Set Build Status
          id: build_status
          run: echo "::set-output name=status::$(if [ $? -eq 0 ]; then echo 'success'; else echo 'failure'; fi)"

      - name: Unit Test
        run: ./gradlew test
        id: unit_test
        continue-on-error: false
        - name: Set Unit Test Status
          id: unit_test_status
          run: echo "::set-output name=status::$(if [ $? -eq 0 ]; then echo 'success'; else echo 'failure'; fi)"

      - name: SonarQube Scan
        uses: sonarsource/sonarcloud-github-action@master
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        id: sonarq
        continue-on-error: false
        - name: Set SonarQube Status
          id: sonar_status
          run: echo "::set-output name=status::$(if [ $? -eq 0 ]; then echo 'success'; else echo 'failure'; fi)"

      - name: JFrog Upload
        run: jfrog rt upload "build/libs/*" "my-repo/"
        id: jfrog_upload
        continue-on-error: false
        - name: Set JFrog Upload Status
          id: jfrog_upload_status
          run: echo "::set-output name=status::$(if [ $? -eq 0 ]; then echo 'success'; else echo 'failure'; fi)"

      - name: Optional - JFrog X-Ray
        if: ${{ secrets.JFROG_XRAY == 'true' }}
        run: jfrog rt xr "my-repo/*"
        id: jfrog_xray
        continue-on-error: false
        - name: Set JFrog X-Ray Status
          id: jfrog_xray_status
          run: echo "::set-output name=status::$(if [ $? -eq 0 ]; then echo 'success'; else echo 'failure'; fi)"

      - name: Deployment
        if: ${{ secrets.DEPLOYMENT == 'true' }}
        run: ./deploy.sh
        id: deployment_step
        continue-on-error: false
        - name: Set Deployment Status
          id: deployment_status
          run: echo "::set-output name=status::$(if [ $? -eq 0 ]; then echo 'success'; else echo 'failure'; fi)"

      - name: Log Stage Statuses to PostgreSQL
        run: |
          psql -h ${{ secrets.PG_HOST }} -U ${{ secrets.PG_USER }} -d github_metrics -c "
          INSERT INTO pipeline_metrics (repo_name, workflow_name, deployment_frequency, lead_time, change_failure_rate, restore_time, pipeline_duration, deployment_platform, stage_name, stage_status)
          VALUES ('${{ github.repository }}', '${{ github.workflow }}', ${DEPLOYMENT_FREQUENCY}, ${LEAD_TIME}, ${CHANGE_FAILURE_RATE}, ${RESTORE_TIME}, ${PIPELINE_DURATION}, '${{ deployment_platform }}', 'Checkout', '${{ steps.checkout_status.outputs.status }}'),
          ('${{ github.repository }}', '${{ github.workflow }}', ${DEPLOYMENT_FREQUENCY}, ${LEAD_TIME}, ${CHANGE_FAILURE_RATE}, ${RESTORE_TIME}, ${PIPELINE_DURATION}, '${{ deployment_platform }}', 'Build', '${{ steps.build_status.outputs.status }}'),
          ('${{ github.repository }}', '${{ github.workflow }}', ${DEPLOYMENT_FREQUENCY}, ${LEAD_TIME}, ${CHANGE_FAILURE_RATE}, ${RESTORE_TIME}, ${PIPELINE_DURATION}, '${{ deployment_platform }}', 'Unit Test', '${{ steps.unit_test_status.outputs.status }}');
          "
```

### Key Points:

1. **Stage Success/Failure Tracking**:

   * For each step (like Checkout, Build, etc.), I’ve added a `set-output` command at the end of the step to track the success (`success`) or failure (`failure`) of the respective stage.
   * The `continue-on-error: false` ensures that the job fails if a step fails.
2. **PostgreSQL Logging**:

   * After all stages, the status of each stage is logged into the **PostgreSQL** database.
   * The status is passed as an output from each step (e.g., `steps.checkout_status.outputs.status`).

### Explanation of Key Commands:

* **set-output**:
  This command is used to capture the result of each step and store it as an output variable (`status`). It captures whether the step succeeded or failed and passes this status to later steps in the job.

```bash
echo "::set-output name=status::$(if [ $? -eq 0 ]; then echo 'success'; else echo 'failure'; fi)"
```

* **PostgreSQL Log Insertion**:
  After the completion of all steps, a new entry is inserted into the **PostgreSQL** database with the status of each stage.

```bash
psql -h ${{ secrets.PG_HOST }} -U ${{ secrets.PG_USER }} -d github_metrics -c "
INSERT INTO pipeline_metrics (repo_name, workflow_name, deployment_frequency, lead_time, change_failure_rate, restore_time, pipeline_duration, deployment_platform, stage_name, stage_status)
VALUES ('${{ github.repository }}', '${{ github.workflow }}', ${DEPLOYMENT_FREQUENCY}, ${LEAD_TIME}, ${CHANGE_FAILURE_RATE}, ${RESTORE_TIME}, ${PIPELINE_DURATION}, '${{ deployment_platform }}', 'Checkout', '${{ steps.checkout_status.outputs.status }}');
"
```

* **`continue-on-error`**:
  This flag ensures that even if a step fails, the next steps can still be executed, but in this case, it is set to `false` to stop the workflow if a step fails.

---

### Adjustments for Optional Stages:

For optional stages, you can use conditions (`if`) to determine whether a step should be included or not in the workflow. The optional steps will only be executed if their corresponding condition (`JFROG_XRAY == 'true'`, for instance) is met.

You can capture the status of these optional steps using the same `set-output` mechanism and log it to the monitoring system.

---

With this implementation, you'll be able to track the success/failure status of each stage (both mandatory and optional) and log these statuses into your database for later reporting and visualization.
