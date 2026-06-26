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


## Pipelines

Two pipelines, split by branch flow. Both are built almost entirely from stage
templates (Pre_Build / Build / Deploy_Env / Approve_Env) so the two share one
set of building blocks — change a block once, both inherit it.

| Pipeline | Trigger | Flow |
|----------|---------|------|
| **Develop** | push to `feature/**` | Pre-build → Build → **[Approve DEV]** → Deploy DEV |
| **Release** | push / PR to `release/**` | Pre-build → Build → Deploy TEST → **[Approve UAT]** → Deploy UAT → **[Approve LIVE]** → Deploy LIVE |

## Environments & promotion (demo)

Four environments: **DEV → TEST → UAT → LIVE**.

| Env | Pipeline | How it deploys | Gate |
|-----|----------|----------------|------|
| DEV | Develop | Push to `feature/**` runs pre-build + build auto | manual approval (Approve DEV) before deploy |
| TEST | Release | Create / push / PR to `release/**` | none — auto |
| UAT | Release | After TEST | manual approval (Approve UAT) |
| LIVE | Release | After UAT | manual approval (Approve LIVE) |

On a feature branch the tests + build run automatically on every push, but
nothing hits DEV until a developer clicks **Approve DEV**. Devs don't have to
think — push and it builds — but the deploy stays a deliberate click.

Every deploy echoes build ID, execution ID, branch and commit SHA, reads back
the `build-info.txt` artefact, then runs verify (version / health).

### Branch flows

```
feature/**  --push-->  pre-build -> build -> [Approve DEV] -> Deploy DEV (verify)

release/**  --push/PR->  pre-build -> build -> Deploy TEST (verify + advisory)
                                              -> [Approve UAT]  -> Deploy UAT (verify)
                                              -> [Approve LIVE] -> Deploy LIVE (verify)
```

Each pipeline is scoped to its branch family by its trigger. Deploy/approve
stages also carry a `when` condition on the branch (`codebase.branch` for push,
`codebase.targetBranch` for PR) as a belt-and-braces guard.

### Try it

- DEV: `git checkout -b feature/demo && git push` → Develop pipeline runs
  pre-build + build, then waits on the **Approve DEV** gate.
- TEST→LIVE: `git checkout -b release/1.0 && git push` → Release pipeline runs
  TEST, then waits on **Approve UAT**, then **Approve LIVE**.

### What this demonstrates

- Devs get fast feedback: push to a feature branch auto-runs tests + build.
- DEV deploy is a deliberate, gated click — not automatic.
- The CD pattern (pre-build → build → deploy → verify) runs end to end.
- Promotion to UAT and LIVE is controlled by explicit manual gates.
- Both pipelines are assembled from the same stage templates — minimal drift.

Files:
- Pipelines:
  `.harness/pipelines/develop_repo_consume_pipeline_modules.yaml` (Develop),
  `.harness/pipelines/repo_consume_pipeline_modules-1782394575309.yaml` (Release)
- Templates: `.harness/templates/Pre_Build.yaml` (tests + validation + quality scans),
  `.harness/templates/Build.yaml` (compile + artefact),
  `.harness/templates/Deploy_Env.yaml` (deploy+verify+advisory),
  `.harness/templates/Approve_Env.yaml` (manual gate)
- Triggers: `.harness/triggers/Push_to_feature.yaml` (→ Develop),
  `Push_to_release.yaml`, `PR_to_release.yaml` (→ Release)

### Reusable blocks

Every stage in both pipelines is a template ref:

| Template | Used by | Inputs |
|----------|---------|--------|
| `Pre_Build` | Develop + Release | none (reads `sonarqube_mode` / `phpstan_mode` pipeline vars) |
| `Build` | Develop + Release | none |
| `Deploy_Env` | DEV / TEST / UAT / LIVE | `envName`, `runAdvisory`, branch `when` |
| `Approve_Env` | DEV / UAT / LIVE gates | `targetEnv`, branch `when` |

Change the deploy or verify logic once in the template and every environment
inherits it. All placeholder steps are `Run` scripts with a `# TODO` line ready
for the real command.

> Note: the **Build** stage must keep the identifier `Build` in each pipeline —
> Deploy_Env's verify step reads the artefact via
> `<+pipeline.stages.Build.spec.execution.steps.Create_Artefact.output...>`.