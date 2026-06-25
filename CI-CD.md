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