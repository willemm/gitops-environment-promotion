# Base tenets and scenario's

This model of promoting GitOps is based on a number of tenets; assumptions about how an application should and will be deployed and promoted between environments throughout its lifecycle.

From this, a number of scenario's can be derived, which are described below.

Note that this list will likely be incomplete, so any questions or comments are welcome.

## Base tenets, observations and assumptions

* Deployments should be as similar as possible between environments.
  Especially between staging and production.  A case could be made that staging could have, for example, fewer replica's,
  but this always comes with the tradeoff/risk that the implications of this setting will only be noticed once the application
  is running on production.
* Some changes are environment specific, such as database settings or external servers.  This is unfortunate but unavoidable.
* Any changes that are not environment-specific, including the application version, are expected to propagate between all environments, sooner rather than later.
* No combination of settings, application version included, may be promoted to production if that exact combination has not been tested on previous environments (excluding unavoidable environment specific settings, of course)
  (This is based on the observation, from experience, that there is a non-zero risk that ostensibly unrelated settings 
  will have dependencies on other changes, including code changes and therefore application versions).
* The Assumption is that only a single version of the application should ever be running on production, with the exception of staggered rollout scenario's.

From this, the following promotion model is deduced

## Promotion model

* All changes done by humans are done on a single `main` branch (caveat: These could be done from feature branches in a more advanced model)
* Application build pipelines also update this branch with the just-built application version/image tag
* For automated promotions, each branch/environment is linked to a downstream branch.  See below for an example.
* If required, versions can be promoted between arbitrary environments.
* Promotions between branches are done with fast-forward merges (if approval is required for certain environments, this can be achieved with pull request approvals)

### What it should look like

If this is followed, the git-history of the gitops repository should be a single line (i.e. no branching lines), and each environment 'branch' will be on this single line.  (Careful, there may be confusion between the meanings of 'branch' in git.)

### Tradeoffs

With this model, it is considerably more difficult to bring a hotfix to production (i.e. to "overtake" other pending changes if required by a production incident)

A proposed solution to this is described in the scenario's.  In short, the idea is to 'insert' the hotfix into the single line mentioned above, at the point where production is at the moment of hotfixing.


## Scenario's

#### Notes about all scenario's

`main` is the root of all promotions.  It is the branch on which developers and build pipelines make the changes that are slated for deployment (by pull request or directly).

All git commands assumes that the local repo is kept in sync with remote, i.e. `git pull` and `git push` are implied.

Most of these commands would obviously be scripted in automated flows.

### Normal promotion flow

Scenario: A normal change is developed and should end up on production

* A change is done on the `main` branch
* Promote from `main` to `qa`: `git checkout qa; git merge --ff main`
* QA tests are done
* Parallel steps for integration and load test environments:
  * Promote from `qa` to `integration`: `git checkout integration; git merge --ff qa`
  * Run relevant tests for this environment
* Wait for all tests to be done
* Parallel steps to go through staging and end up in production:
  * Promote from `integration` to `staging-xx`: `git checkout staging-xx; git merge --ff integration`
  * Run tests on `staging-xx`
  * Promote from `staging-xx` to `prod-xx`: `git checkout prod-xx; git merge --ff staging-xx`

### Environment-specific change for production

Scenario: A change is required for the specific production-only settings

* Do the change in the `variants/prod` folder on the `main` branch
* Start the above normal promotion flow.
* The parallel staging/production steps could be serialized because of the increased incident risk.

### Promotion flow causes production issue

Scenario: The promotion flow was followed, but on production, this caused an incident.

* For all production branches:
  * Roll back the last commit: `git checkout prod-xx; git reset HEAD~1; git push --force`

### Production incident hotfix

Scenario: Production is at v1.0, staging is at 2.0 and qa and main are at 3.0.   At this point, a bug somewhere causes a production incident, which needs to be fixed asap.

Any time that such a hotfix is required, one can assume that any version running elsewhere in the pipeline is also compromised.  I.E. any test is invalidated, because the tested version will never make it to production any more.

In this scenario, the following steps are followed

* All currently running promotion flows are canceled
* A temporary branch is created from production: `git checkout prod-eu; git checkout -b hotfix/incident001`
* The required changes are made on the `hotfix/incident001` branch
* All environment branches are rolled back to prod version: `git checkout <env>; git reset origin/prod-eu`
* Promotion flow is started with `hotfix/incident001` as source: `git checkout qa; git merge --ff hotfix/incident001; ...`
  NB: This should be the normal flow, going through qa, staging, etc.
* As soon as this flow is finished successfully the incident can be closed.  Following steps are no longer time/sla critical.
* The hotfix changes are rolled into the `main` branch: `git checkout main; git rebase hotfix/incident001`
* Conflicts from this rebase must be resolved.  Be happy you can resolve them in development environment.
* Optional: Promotion flows that were canceled can be redone from start by finding the equivalent commit on main to start from (NB: this will have a different revision hash)


## Scenarios mentioned in the original article

A number of scenario's are mentioned in the original article, I'll go over some of them to see the differences in the branch-per-environment model

### Promote application version from one environment to another

When there are no other settings changes this is simply `git checkout staging-us; git merge --ff qa`

Otherwise, application versions and changes to k8s settings should always be assumed to be interdependent so this is a bad practice.

If you really want to do this, however, this is possible with a `git cherry-pick`, by searching for the commit in which the build pipeline committed the newest version.  This should be easy if the build pipeline formats its commit message, and trivial if the build pipeline also tags the gitops-repository when committing a build version.

After this, the normal promotion-flow will be broken, because `merge --ff` is no longer possible.  I think this can be fixed by doing one extra step in the promotion flow, namely `git reset <source branch>; git merge --ff <source branch>`

NB: This obviously applies to all out-of-order promotions

### Promote an application from prod-eu to prod-us along with the extra configuration

I'm not sure why you'd want to do that when it should go through staging-us, but it's simply: `git checkout prod-us; git merge --ff prod-eu`

Or, if you want to force that prod-us becomes the same even if it was ahead: `git checkout prod-us; git reset prod-eu` (NB: Local workspace will now have uncommitted changes)

### Make sure that QA has the same replica count as staging-asia

Seems quite an unlikely scenario.  Maybe if there are issues with staging-asia, involving the replica count, and qa must be used to test these issues?

Anyway, this should be done like any other environment-specific setting, by going through the main branch.  So, go to the `main` branch, copy the environment-specific replica setting from staging-asia to qa, and then running the default promotion flow to pass it to qa

### Make a global change to multiple (non-prod / prod / region) environments

Same as any other variant-specific setting.  Do changes on `main` branch, then do the normal flow.
Specifically for prod-only changes, consider serializing the different staging to prod promotions in the flow.
