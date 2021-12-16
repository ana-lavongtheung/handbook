# Continuous integration playbook

- **Maintainers**: [DevX Team](../../enablement/dev-experience/index.md).
- **Audience**: any software engineer, no prior infrastructure knowlegde required.
- **TL;DR** This document sums up what to do in various scenarios that can block the CI.

See also: [Continuous Integration](https://docs.sourcegraph.com/dev/background-information/continuous_integration) in our product documentation.

## Prerequisites

In order to handle problems with the CI, the following elements are necessary:

- Have access to the `sourcegraph-ci` _project_ on Google Cloud Platform.
  - See [#it-tech-ops](https://sourcegraph.slack.com/archives/C01CSS3TC75)
- Have the CLI `gcloud` tool installed and have authenticated yourself.
  - See [Gain access to the cluster](../../deployments/debugging/tutorial.md)
- Have `kubectl` installed.
  - 💡 You can set the default namespace to `buildkite` to avoid always appending the `-n buildkite` flag to `kubectl` commands.
    - `kubectl config set-context --current --namespace=buildkite`
  - (Optional) Install [K9s](https://k9scli.io) for easier interactions with the _pods_.

## Overview

The CI is what enables us to feel confident when delivering our changes to our users, and is one of the key components enabling Sourcegraph to deliver quality software. While the DevX team is in charge of managing the CI as a tool, it is essential for every engineer to be able to unblock themselves if there is a problem in order be autonomous.

This page lists common failures scenarios and provide a step by step guide to get the CI back in an operational state.

## Scenarios

### Build has failed on the `main` branch

- Severity: _minor_
- Impact: that commit won't be deployed on DogFood and `sourcegraph.com` until an ulterior build passes.
- Possible causes:
  - The `main` branch runs additional checks compared to Pull Requests builds. So it's possible that one of those checks failed.
    - 💡 The checks are dynamically generated by our [pipeline generation tool](https://docs.sourcegraph.com/dev/background-information/continuous_integration). The `main` branch has notably much more exhaustive checks than other branches.
  - The `main` branch have changes that weren't in the Pull Request branch and those changes are causing a failure.
  - The `main` branch is failing due to a previous build.

#### Actions

1. Check your build on [Buildkite](https://buildkite.com/sourcegraph/sourcegraph/builds?branch=main).
   - Find its link directly in the [#buildkite-main](https://sourcegraph.slack.com/archives/C02FLQDD3TQ) channel.
   - 💡 Or run `sg ci status` in your shell, with the `main` branch checked out.
2. Search for the failing steps, and browse the logs (💡 run `sg ci logs` in your shell, with the `main` branch checked out) .
   - Look for a failure explanation: it can be a test that failed or a command that return a non zero exit code.
3. Check the previous builds on the `main` branch on [Buildkite](https://buildkite.com/sourcegraph/sourcegraph/builds?branch=main)
   1. Are they failing with the same exact error?
      - **Yes**: see the [Builds are failing in the `main` branch with the same error](#builds-are-all-failing-on-the-main-branch-with-the-same-error)
      - **No**: see next point.
4. Is that a real failure or a flake?
   1. Restart that step. Maybe it will fail again, but if it doesn't it'll save you time.
      - 💡 You can go to 3. while it runs.
   1. See [Is that a failure or a flake scenario](#is-this-a-failure-or-a-flake)
   1. Did restarting it fixed the problem?
      - **Yes**: that's a flake. See the [Spotted a flake scenario](#spotted-a-flake)
      - **No**: see next point.
   1. Does the failure points to problem with the code that was shipped on that commit?
      1. Yes, and it's a very quick fix that can get merged promptly:
         1. Write a short message on [#buildkite-main](https://sourcegraph.slack.com/archives/C02FLQDD3TQ) and tell others that you're fixing it.
         1. Submit the fix with another PR and get it merged as soon as possible.
      1. Yes, but it's not easily and/or quickly fixed
         1. Revert the incriminating Pull Request.
         1. [Open a GitHub issue](https://github.com/sourcegraph/sourcegraph/issues/new?assignees=&labels=testing%2Cflake&template=flaky_test.md&title=Flake%3A+%24TEST_NAME+disabled) mentioning the build and the context to explain to the team owning that test what happened.
         1. Checkout the PR branch.
         1. Rebase it so it includes the changes that broke it when merged in the `main` branch.
         1. Rename the branch to `main-dry-run/your-branch` in order to get the CI to run the same exact checks it does on the `main` branch.
      1. No, but it seems to fail in step or code from another team.
         1. Reach out a member of the team responsible for that test.
         2. go for a. or b. from the previous points.
   1. No, and there is suspicion of a flake.
      - **Yes**: that's a flake. See the [Spotted a flake scenario](#spotted-a-flake)

### Builds are all failing on the `main` branch with the same error

- Severity: _major_
- Impact: no commits are being deployed on DogFood and `sourcegraph.com` until the problem is resolved. Cutting a release is impossible.
- Possible causes:
  - A previous Pull Request introduced a change that causes a test to fail.
  - A previous Pull Request introduced a change that modified state in an unexpected way and broke the CI.
  - An external dependency is not available anymore and is causing builds to fail.
  - Some rate limiting API is throttling us and causing builds to fail.

#### Actions

1. Identify the error in common with the recent builds on [Buildkite](https://buildkite.com/sourcegraph/sourcegraph/builds?branch=main).
   - 💡 See [How to use loki here](#actions-4)
1. Find the build where the problem appeared for the first time.
   - 💡 Often it's the first build that became red, but check that the error is the same to be sure.
1. Is this an external failure or an internal one?
   - 💡 External failures are about downloading a dependency like a package in a script or a in a Dockerfile. Often they'll manifest in the form of an HTTP error.
   - 💡 If unsure, ask for help on [#dev-chat](https://sourcegraph.slack.com/archives/C07KZF47K).
   - **Yes**, it's an external failure:
     1. See the [SSH into an agent scenario](#ssh-into-an-agent)
     1. Try to reproduce the faulty HTTP request so you can observe what's the problem. Is it the same failure?
        - **Yes**: Do you know how to fix it? If **no** escalate by creating an incident (`/incident` on Slack).
        - **No**: escalate by creating an incident (`/incident` on Slack).
   - **No**, it's an internal failure:
     1. Is it involving faulty state in the agents? (a given tool is not found where it should have been present, or have incorrect version)
        - See the [SSH into an agent scenario](#ssh-into-an-agent)
     1. Try to find an agent that recently successfully ran the faulty step (look for a green build on the `main` branch)
        1. Can you see a difference? If **yes** take note.
     1. Do you know how to fix it?
        - **Yes**: apply the fix.
        - **No**: Restart the agents to see if it fixes the problem. See [Restarting the agents](#restarting-the-agents)
          - Does it fix the problem? If no, escalate by creating an incident (`/incident` on Slack).

### Build are failing on the `main` branch with different errors

- Severity: _major_
- Impact: no commits are being deployed on DogFood and `sourcegraph.com` until the problem is resolved. Cutting a release is impossible.
- Possible causes:
  - A previous Pull Request introduced a change that causes a test to fail.
  - A previous Pull Request introduced a change that modified state in an unexpected way and broke the CI.
  - An external dependency is not available anymore and is causing builds to fail under certain conditions.
  - Some rate limiting API is throttling us and causing builds to fail.

#### Actions

1. Escalate by creating an incident (`/incident` on Slack).
1. Get some help.
1. Downscale the agents to be able to observe exactly what's going on.
   1. Update the `autoscaler` manifest to run a single agent
      1. `cd sourcegraph/infrastructure`
      1. Edit the [manifest](https://sourcegraph.com/github.com/sourcegraph/infrastructure/-/blob/buildkite/kubernetes/buildkite-autoscaler/buildkite-autoscaler.Deployment.yaml?L32-35) to set `1` on both maximum and mininimum agent count.
      1. `cd buildkite/kubernetes/buildkite-autoscaler`
      1. Run `kubectl apply -n buildkite -f buildkite-autoscaler.Deployment.yaml`
      1. Use`kubectl get pods -n buildkite -w` to observe the currently running agents (`k9s` works here too).
   1. From there, any change and build will run on a single agent, allowing you to observe the behaviour live.

### Spotted a flake

- Severity: _minor_
- Impact: Some builds will fail randomly, creating noise and slowing down the engineering team
- Possible causes:
  - Tests relying on timing.
  - Race conditions.
  - End to end tests are delicate by nature and can fail randomly due to the complexity of all involved components.
  - State dependent test is not properly teared down and fails.

#### Actions

1. What kind of step is failing?

- Is this an End-to-end tests?
  - 💡 E2E tests are fragile by nature, there is no way around it.
  - Take note.
- Is this a Docker image build step?
  - 💡 This should really not be happening.
  - Is the error about the Docker daemon?
    - **Yes**, this is a CI infrastructure flake. Ping `@dev-experience-support` on Slack in the [#buildkite-main](https://sourcegraph.slack.com/archives/C02FLQDD3TQ) or [#dev-experience](https://sourcegraph.slack.com/archives/C01N83PS4TU) channels.
    - **No**: reach out to the team owning that Docker image _immediately_.
- Anything else
  - Take note of the failing step and go to next point.

1. Is that flake related to the CI infrastructure?

- The CI infrastructure often involves:
  - Docker daemon not being reachable.
  - Missing tools that we use to run the steps, such as `go`, `node`, `comby`, ...
  - Errors from `asdf`, which is used to manage the above tools.
- **Yes**: ping `@dev-experience-support` on Slack in the [#buildkite-main](https://sourcegraph.slack.com/archives/C02FLQDD3TQ) or [#dev-experience](https://sourcegraph.slack.com/archives/C01N83PS4TU) channels.
  - If nodoby is online to help:
    - Reach out for help in [#dev-chat](https://sourcegraph.slack.com/archives/C07KZF47K)
    - Try [Restarting the agents](#restarting-the-agents) and restart the build.

1. Is that flake related to the code:

- See the process describe in the [flaky tests page](https://docs.sourcegraph.com/dev/background-information/testing_principles#flaky-tests)

### Is this a failure or a flake?

- Gravity: _minor_
- Impact: Some builds will fail randomly, creating noise and slowing down the engineering team
- Possible causes:
  - Tests relying on timing.
  - Race conditions.
  - End to end tests are delicate by nature and can fail randomly due to the complexity of all involved components.
  - State dependent test is not properly teared down and fails.

#### Actions

1. Immediately restart the faulty step.
   - 💡 It will save you time while you're looking at the logs.
   - Is the step passing now?
     - **Yes**: See [Spotted a flake scenario](#spotted-a-flake)
     - **No**: Give it another try, and see next point.
1. Check on [Grafana](https://sourcegraph.grafana.net/explore?orgId=1&left=%5B%22now-12h%22,%22now%22,%22grafanacloud-sourcegraph-logs%22,%7B%22refId%22:%22A%22,%22expr%22:%22%7Bapp%3D%5C%22buildkite%5C%22%7D%22%7D%5D) if there are any occurrences of the failures that were previously observed:
1. Go the the "Explore" section
1. Make sure to select `grafanacloud-sourcegraph-logs` in the dropdown at the top of page.
1. Scope the time window to `7 Days` to make sure to find previous occurrences if there are any
1. Enter a query such as `{app="buildkite"} |= "your error message"` where "your error message" is a string that identiy approximately the failure cause observed in the failing step.
1. Is there a build that failed exactly like this?
   - **Yes**:
     1. 💡 Double check that you're looking at that the same step by inspecting the labels of message (click on the line to make them visible)
     1. **Yes**, that's a flake. See the [Spotted a flake scenario](#spotted-a-flake)
   - **No**: it's not a flake, reach out the team owning those tests.

### Restarting the agents

- Gravity: _minor_
- Impact: May fail ongoing builds, but that's fine.
- Possible causes:
  - Manual restart by an engineer.
  - Newer version of the agents needs to be deployed.

#### Actions

1. Use `kubectl get pods -n buildkite -w` to observe the currently running agents (`k9s` works here too).
1. In a different terminal, run `kubectl -n buildkite rollout restart deployment buildkite-agent`.
1. Wait a bit to see the agents restarting completely.
1. Restart the faulty build and observe it the problem is fixed or not.
   - If necessary: escalate by creating an incident (`/incident` on Slack).

### SSH into an agent

- Gravity: none
- Impact: none (unless a destructive action is performed)
- Possible cause:
  - Need to investigate a problem and suspect the agent is at fault

#### Actions

1. Find the pod you want to SSH into with one of the following methods:
   1. Use `kubectl get pods -n buildkite -w` to observe the currently running agents and get the pod name (`k9s` works here too).
   2. From a Buildkite build page, click the "Timeline" tab of a job and see the entry for "Accepted Job". The "Host name" in the entry is also the name of the pod that the job was assigned to.
2. Use `kubectl exec -n buildkite -it buildkite-agent-xxxxxxxxxx-yyyyy -- bash` to open a shell on the Buildkite agent.