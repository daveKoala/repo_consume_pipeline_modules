# CI/CD pipeline description

I amd trying to create an idealised delivery pipeline, to prove Harness as a demo.

The reference pipeline should include:
* Pre build tests and validation
* build and artefact creation
* deployment to non-production environments
* build/version verification
* health/readiness checks
* migration status checks
* non-destructive API validation
* controlled mutation validation in non-production
* optional advisory checks such as OpenAPI fuzzing and performance smoke tests
* clear blocking vs advisory behaviour
This should be treated as a delivery pattern PoC, not a tool war.
The output should help us answer:
* Can teams own and understand this pipeline?
* Does it provide better deployment confidence?
* Is it easier to support?
* Does it meet audit and governance needs?
* Should the pattern be implemented in Harness, Bitbucket, or both depending on service complexity?


On PR to `release/*` branch we will run 
pre-build steps

Build steps

Deploy steps

Post deploy steps

As this is a demo the indiviual steps can just be `echo "<step name>"`



## Quality and secuirty

There are 3rd prty tools like Sonarqube, PHP stan, etc that we will use.

What i need to concider is that these hostroically have caused us issues. E.g. someone did not pay a bill so the service kept failing.

What i need is a way of turning on and off steps depending on our appitite for risk in these situations.

### Risk-appetite toggles (implemented)

Each 3rd-party tool has a pipeline variable set per run, with three modes:

| Mode | Behaviour | Use when |
|------|-----------|----------|
| `off` | step is skipped entirely | vendor outage / unpaid bill — don't let it block delivery |
| `advisory` | runs, but a failure does **not** block | want signal without risk to the pipeline |
| `blocking` | runs, a failure **blocks** | high confidence in the tool, want it enforced |

Variables: `sonarqube_mode`, `phpstan_mode` (default `advisory`). Add one per
tool you onboard.

- `off` is handled by a stage `when` condition (`<+pipeline.variables.X_mode> != "off"`) → step is skipped.
- `advisory` vs `blocking` is decided inside the step: in advisory mode a tool
  failure is swallowed (`exit 0`); in blocking mode the real exit code stands.

So if SonarQube goes down (or the bill lapses), set `sonarqube_mode = off` for
the run and the pipeline still ships.


## Environments & promotion (demo)

Four environments: **DEV → TEST → UAT → LIVE**.

| Env | How it deploys | Gate |
|-----|----------------|------|
| DEV | Push to `feature/**` (any developer) | none — auto |
| TEST | Create / push / PR to `release/**` | none — auto |
| UAT | After TEST | manual approval (Approve UAT) |
| LIVE | After UAT | manual approval (Approve LIVE) |

Every deploy echoes build ID, execution ID, branch and commit SHA, reads back
the `build-info.txt` artefact, then runs verify (version / health).

### Branch flows

```
feature/**  --push-->  pre-build -> build -> Deploy DEV (verify)

release/**  --push/PR->  pre-build -> build -> Deploy TEST (verify + advisory)
                                              -> [Approve UAT]  -> Deploy UAT (verify)
                                              -> [Approve LIVE] -> Deploy LIVE (verify)
```

Stage selection is driven by `when` conditions on the branch
(`codebase.branch` for push, `codebase.targetBranch` for PR). Stages that do
not match are skipped.

### Try it

- DEV: `git checkout -b feature/demo && git push` → runs DEV only.
- TEST→LIVE: `git checkout -b release/1.0 && git push` → runs TEST, then waits
  on the **Approve UAT** gate, then **Approve LIVE**.

### What this demonstrates

- DEV is open: any developer can push and deploy.
- The CD pattern (pre-build → build → deploy → verify) runs end to end.
- Promotion to UAT and LIVE is controlled by explicit manual gates.

Files:
- Pipeline: `.harness/pipelines/repo_consume_pipeline_modules-1782394575309.yaml`
- Templates: `.harness/templates/Deploy_Env.yaml` (deploy+verify+advisory), `Approve_Env.yaml` (manual gate)
- Triggers: `.harness/triggers/Push_to_feature.yaml`, `Push_to_release.yaml`, `PR_to_release.yaml`

### Reusable blocks

The 4 deploy stages and 2 approval stages are one template each:

| Template | Used by | Inputs |
|----------|---------|--------|
| `Deploy_Env` | DEV / TEST / UAT / LIVE | `envName`, `runAdvisory`, branch `when` |
| `Approve_Env` | UAT / LIVE gates | `targetEnv`, branch `when` |

Change the deploy or verify logic once in the template and every environment
inherits it. All placeholder steps are `Run` scripts with a `# TODO` line ready
for the real command.