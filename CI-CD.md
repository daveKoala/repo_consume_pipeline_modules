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



Quality and secuirty

There are 3rd prty tools like Sonarqube, PHP stan, etc

What i need to concider is that these hostroically have caused us issues. E.g. someone did not pay a bill so the service kept failing.

What i need is a way of turning on and off steps depending on our appitite for risk in these situations.


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
- Triggers: `.harness/triggers/Push_to_feature.yaml`, `Push_to_release.yaml`, `PR_to_release.yaml`