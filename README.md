# DevSecOps Pipeline for OWASP Juice Shop

## 📌 Project Overview
This project implements an automated, end-to-end DevSecOps CI/CD pipeline for **OWASP Juice Shop**. The pipeline integrates multiple open-source security tools directly into GitHub Actions and automatically forwards all aggregated scan results to a self-hosted **DefectDojo** instance via a secure reverse tunnel.

---

## 🛠️ DevSecOps Pipeline Architecture & Tools

| Security Stage | Tool Used | Output Format | Description |
| :--- | :--- | :--- | :--- |
| **Secrets Scanning** | Gitleaks | JSON | Scans the repository history for hardcoded secrets, tokens, and keys. |
| **SAST** | Semgrep | JSON | Performs Static Application Security Testing against source code for vulnerabilities. |
| **SCA** | Trivy | JSON | Software Composition Analysis scanning application dependencies for known CVEs. |
| **SBOM** | CycloneDX | JSON | Generates a Software Bill of Materials tracking package inventory. |
| **DAST** | OWASP ZAP | XML / JSON | Dynamic Application Security Testing against a live running Juice Shop Docker container. |
| **Reporting** | DefectDojo | API Aggregation | Centralized dashboard ingesting scan outputs from all 5 tools. |

---

## 🔌 Local DefectDojo Setup & External Exposure

DefectDojo is hosted locally on port `8080`. To enable GitHub Actions (running on cloud runners) to communicate securely with the local DefectDojo instance, a secure SSH reverse tunnel was established using **Serveo**.

<img width="1856" height="390" alt="image" src="https://github.com/user-attachments/assets/44ba2e1c-35d9-45fa-8b78-3fce3eebc695" />

###  **Reverse SSH Tunnel Command**

ssh -i ~/.ssh/id_ed25519 -R angel-defectdojo:80:localhost:8080 serveo.net
<img width="1573" height="313" alt="image" src="https://github.com/user-attachments/assets/238b3a82-8320-43b9-9411-c496c35c3d03" />


- ** Local Target:** http://localhost:8080 (Local DefectDojo instance)
- **Exposed Endpoint**: https://angel-defectdojo.serveousercontent.com**


## **GitHub Repository Secrets**
<img width="1549" height="820" alt="image" src="https://github.com/user-attachments/assets/ff9e59c5-13ac-458d-a3a7-e37386b4e1da" />

The external URL and credentials were mapped securely using GitHub Secrets:

- DEFECTDOJO_URL: https://angel-defectdojo.serveousercontent.com

- DEFECTDOJO_API_KEY: API token generated from DefectDojo

- DEFECTDOJO_ENGAGEMENT_ID: Target engagement ID (1)



## 📊 **Pipeline Proof & Execution Results**
<img width="695" height="267" alt="image" src="https://github.com/user-attachments/assets/e1659151-76b8-4a5a-be59-99abec519b6a" />

GitHub Actions Successful Build
- Commit Hash: 9ecd839 ("Fix ZAP permissions and update workflow pipeline")
- Workflow Run: #14 (Status: Passed / Green Checkmark)

<img width="1666" height="867" alt="image" src="https://github.com/user-attachments/assets/8fdb7445-aa7a-49d6-9cf8-2ead7d197780" />


## **DefectDojo Integrated Vulnerability Summary**

- Total Active Findings: 534
- High: 362 | Medium: 161 | Low: 7 | Info: 4

Scan Breakdown in DefectDojo Engagement:
 - Gitleaks Scan: 194 Findings
 - Semgrep JSON Report: 208 Findings
 - Trivy Scan: 121 Findings
 - OWASP ZAP Scan: 11 Findings
 - CycloneDX SBOM: Ingested (0 vulnerabilities, full package map)

<img width="1872" height="802" alt="image" src="https://github.com/user-attachments/assets/8c7b57ce-3f31-4bed-a59a-e9fa7ad67aef" />



**⚙️ GitHub Actions CI/CD Pipeline Workflow**


name: DevSecOps Security Pipeline

on:
  push:
    branches: [ "main", "master" ]
  pull_request:
    branches: [ "main", "master" ]

# 1. Add top-level permissions to prevent 403 API errors
permissions:
  contents: read
  issues: write

jobs:
  security-scans:
    name: Execute DevSecOps Scans
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Node.js Environment
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install Dependencies
        run: npm install --legacy-peer-deps || true

      # 1. Secrets Scanning (Gitleaks)
      - name: Secrets Scan (Gitleaks)
        run: |
          docker run --rm -v "$(pwd):/path" zricethezav/gitleaks:latest detect --source="/path" --report-format json --report-path /path/gitleaks_results.json || true

      # 2. Static Application Security Testing (SAST)
      - name: SAST Scan (Semgrep)
        run: |
          docker run --rm -v "$(pwd):/src" returntocorp/semgrep semgrep scan --config=auto --json -o /src/sast_results.json || true

      # 3. Software Composition Analysis (SCA)
      - name: SCA Scan (Trivy)
        run: |
          docker run --rm -v "$(pwd):/root/src" aquasec/trivy fs /root/src --format json -o /root/src/sca_results.json || true

      # 4. Software Bill of Materials (SBOM)
      - name: Generate SBOM (CycloneDX)
        run: |
          npx @cyclonedx/cyclonedx-npm --output-file sbom.json || true

      # 5. Dynamic Application Security Testing (DAST)
      - name: Run Juice Shop Container
        run: docker run -d --name juice-shop -p 3000:3000 bkimminich/juice-shop

      - name: Wait for App to Start
        run: sleep 20

      # 2. Configure ZAP with token and disable automated issue creation
      - name: DAST Scan (OWASP ZAP)
        uses: zaproxy/action-baseline@v0.14.0
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          target: 'http://localhost:3000'
          artifact_name: 'zap_artifacts'
          cmd_options: '-J report_json.json -r report_html.html -x report_xml.xml'
          allow_issue_writing: false
          fail_action: false

      - name: Adjust File Permissions and Resolve ZAP Report
        if: always()
        run: |
          sudo chmod -R 777 .
          if [ -f "report_xml.xml" ]; then cp report_xml.xml dast_results.xml; fi
          if [ -f "report_json.json" ]; then cp report_json.json dast_results.json; fi

      # 6. Push All 5 Scan Reports Directly to DefectDojo using Secrets
      - name: Push Reports Directly to DefectDojo
        if: always()
        env:
          DEFECTDOJO_URL: ${{ secrets.DEFECTDOJO_URL }}
          API_KEY: ${{ secrets.DEFECTDOJO_API_KEY }}
          ENGAGEMENT_ID: ${{ secrets.DEFECTDOJO_ENGAGEMENT_ID }}
        run: |
          upload_scan() {
            FILE=$1
            TYPE=$2
            if [ -f "$FILE" ]; then
              echo "----------------------------------------"
              echo "Uploading $FILE as $TYPE to DefectDojo..."
              RESPONSE=$(curl -s -k -w "\nHTTP_CODE:%{http_code}" -X POST "$DEFECTDOJO_URL/api/v2/import-scan/" \
                -H "Authorization: Token $API_KEY" \
                -F "scan_type=$TYPE" \
                -F "engagement=$ENGAGEMENT_ID" \
                -F "active=true" \
                -F "verified=false" \
                -F "minimum_severity=Info" \
                -F "close_old_findings=false" \
                -F "file=@$FILE")
              echo "Response: $RESPONSE"
            else
              echo "File $FILE not found, skipping upload."
            fi
          }

          upload_scan "gitleaks_results.json" "Gitleaks Scan"
          upload_scan "sast_results.json" "Semgrep JSON Report"
          upload_scan "sca_results.json" "Trivy Scan"
          upload_scan "sbom.json" "CycloneDX Scan"
          upload_scan "dast_results.xml" "ZAP Scan"

      # 7. Upload Artifacts to GitHub Actions
      - name: Upload Security Reports Artifacts
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: devsecops-scan-reports
          path: |
            gitleaks_results.json
            sast_results.json
            sca_results.json
            sbom.json
            dast_results.xml
            dast_results.json
          if-no-files-found: warn
