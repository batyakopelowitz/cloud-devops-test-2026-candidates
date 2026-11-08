## Problem 4 – Failed release

**Risk**

- **mutable `latest` tag** - every build overwrites `latest`, so we can't tell what is deployed, and even `kubectl rollout undo` may pull the broken image again.
- **no deployed version record** - there is no known good version to return to.
- **incompatible migration** - if the migration is not backward compatible, the previous image breaks. Restoring the database from backup loses all data written since the migration.

**Fix**

Pipeline flow (Azure DevOps Server, self hosted agent):

1. **PR** - branch policy runs tests and an image build, merge only when green and reviewed.
2. **Build once** - on merge to `main`, build the image, tag it with the commit SHA, push, save the digest.
3. **Validate** - on the agent, run migrations on a temporary PostgreSQL container and smoke test both the new and the previous image.
4. **Approval** - `production` environment with an approval check. The approver checks the run: commit, changes, digest and whether there is a migration.
5. **Migrate + deploy** - on-demand Cloud SQL backup, run the migration, deploy by digest.
6. **Record** - save the previous and new digest.

Migrations follow **expand/contract**: a release only adds columns/tables; removing or renaming happens in a later release. So the previous image always works with the current schema.

```yaml
trigger:
  branches: { include: [ main ] }             # only main can reach production
pool: { name: gcp-linux-agents }              # self hosted Ubuntu agent

variables:
  IMAGE: me-west1-docker.pkg.dev/<project>/survey/api
  TAG: $(Build.SourceVersion)                 # commit SHA, never "latest"

# API only, web and worker use the same steps
stages:
- stage: Build
  jobs:
  - job: build
    steps:
    - script: |
        docker build -t $(IMAGE):$(TAG) . && docker push $(IMAGE):$(TAG)
        docker inspect --format='{{index .RepoDigests 0}}' $(IMAGE):$(TAG) > image.txt
    - publish: image.txt
      artifact: release

- stage: Production
  dependsOn: Validate                         # Validate stage not shown
  jobs:
  - deployment: deploy_api
    environment: production                   # approval check
    strategy:
      runOnce:
        deploy:
          steps:
          - script: |
              NEW=$(cat $(Pipeline.Workspace)/release/image.txt)
              PREV=$(kubectl get deploy api -o jsonpath='{.spec.template.spec.containers[0].image}')
              echo "previous=$PREV new=$NEW" > deploy-record.txt
              # PSEUDOCODE: backup Cloud SQL, run migration Job with $NEW
              kubectl set image deployment/api api=$NEW
              kubectl rollout status deployment/api --timeout=5m
          - publish: deploy-record.txt
            artifact: deploy-record
```

**Recovery**

1. Take the previous digest from the last `deploy-record`.
2. Re run the Production stage of that previous run (emergency: `kubectl set image` with that digest).
3. Leave the database as is, the expand only migration works with the previous image.
4. Check 5XX rate and smoke test, then fix forward with a normal PR.

**If the migration is incompatible**, the old image won't work and redeploying it doesn't restore the database. Prefer, in order: roll forward with a fix, a tested reverse migration, and only last restore the database (Problem 2), which loses recent changes.

**Safe implementation**

1. Before any change, record the digest that is running now: `kubectl get pods -l app=api -o jsonpath='{.items[0].status.containerStatuses[0].imageID}'`.
2. Create the `production` environment with the approval check, and add the branch policy on `main`.
3. Switch the pipeline to SHA tags and run one release. Check that `deploy-record` has the previous and new digest.

**Verification** (proposed, not tested)

- Deploy a build that returns 500 on purpose, roll back using the recorded digest, confirm the 5XX stop.
- Rename a column the API uses on purpose and confirm the Validate stage fails on the previous image test.