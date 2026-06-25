# Automated Database Deployment — Liquibase · Jenkins · OpenShift

End-to-end CI/CD automation for **Oracle database changes** in a regulated banking environment. A Bitbucket pull-request merge drives Liquibase changesets through DEV → INT → UAT, and a change-managed Production flow deploys inside an isolated, data-resident zone — all from a single Jenkins pipeline selected by an `ACTION` parameter.

> The diagrams in this repo are self-contained HTML files. Open them in any browser — no build step, no internet required.

---

## Why this exists

- **One pipeline, many actions.** Deploy, Create Release RC, Production release, Rollback and Deployment History are all the same Jenkinsfile, chosen at runtime.
- **Two connection topologies.** DEV is a direct certificate connection (outside the secure zone). INT and above never let the CI plane touch the database — a containerized OpenShift job runs the update from inside the zone.
- **Security first.** Password login is locked out on UAT and Production databases. A pre-configured *tech user* authenticates by **certificate only**, with minimal sufficient grants. Secrets are stored as namespace-scoped sealed secrets.
- **Audit-ready.** Every production release is written to a Confluence master table (implementer, change request, change window, RC version, schemas affected, logs, status, errors, date).

---

## How it works

### 1. DEV plane (outside the secure zone)
- Developers deploy from their IDE (IntelliJ + Liquibase Maven plugin) using `pom.xml` and `liquibase-{env}.properties`, or by pushing to a feature branch that triggers a pipeline.
- Liquibase **locks** serialize concurrent runs; **checksums** skip already-applied changesets.
- DEV deploys are direct — no artifact is built or pushed to Artifactory.

### 2. Release pipeline
- Triggered by a PR merge (Bitbucket webhook) **or** a manual run.
- An **entitlement check** confirms the initiator is in the DevOps group before UAT/Production flows are allowed.
- Input options: deploy against a **Jira ticket** (logs attached back to Jira) or run a **Production release**.

### 3. Schema-aware parallel deployment
- The repo is laid out per schema under each release folder; the master changelog always points at the latest release.
- A `git diff` detects affected schemas; only those deploy.
- Stages launch in **parallel** but are serialized by Jenkins pipeline **locks** because schema objects have cross-dependencies.

### 4. INT & UAT — secure region
- Jenkins packages SQL + Liquibase files into a `tar` and uploads it to Artifactory (`pom.xml` excluded).
- Artifactory and OpenShift sit in the **same zone but separate planes**, connected by a certificate **SSL handshake**.
- A reusable container image pulls the tar and runs `liquibase update` against Oracle via the cert-only tech user.

### 5. Production — regulated, isolated zone
- After code freeze, `Create Release RC` builds a frozen tar (e.g. `26.x.x-RC1`) and logs the RC number to Confluence.
- `Production release` takes a **Change Number** + RC name, then passes two gates:
  - **Implementer check** — executor must match the change-request implementer.
  - **ServiceNow checks** — change window + production checklist via ServiceNow APIs.
- On pass, a prod OpenShift **service account** spins a container job, pulls the RC (sealed-secret creds) and deploys.

### 6. Release isolation with Delphix
- A single shared DB across 10+ teams was the bottleneck. **Delphix virtual databases** give each non-current release an isolated, on-demand replica, synced from source by button click or API.

### 7. Reuse by consumer teams
- OpenShift objects (jobs, sealed secrets) ship as a **Helm package**.
- The pipeline is published as a Jenkins **shared library** — teams reference `@Library('db-deploy')`.
- Teams own only their project structure and DB config files.

---

## Repository contents

| File | What it is |
|------|-----------|
| `export/DB Deployment Topology.html` | Full architecture / deployment topology diagram (standalone) |
| `export/Jenkins Pipeline Workflow.html` | Blue Ocean–style single-pipeline workflow (standalone) |
| `export/*.pptx` | Slide versions of each diagram |
| `*.dc.html` | Editable source of each diagram |

### Viewing the diagrams
Open any file in `export/` directly in a browser, or use the GitHub Pages link once published (see below).

---

## Tech stack

`Liquibase` · `Jenkins` (shared libraries, Blue Ocean) · `Bitbucket` · `Artifactory` · `Docker` · `OpenShift` (Jobs, Helm, Sealed Secrets) · `Oracle` · `Delphix` · `ServiceNow` · `Jira` · `Confluence`

---

## License

Add a license of your choice (e.g. MIT) if you intend others to reuse this.
