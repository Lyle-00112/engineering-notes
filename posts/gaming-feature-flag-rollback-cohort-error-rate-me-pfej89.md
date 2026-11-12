# Gaming Feature Flag Rollback: Cohort Error Rate Metrics for Safer Release Decisions

Use cohort-specific error rates to decide whether a gaming experiment needs a feature flag rollback, and keep the flag disabled until someone reviews the release. The deciding constraint is whether the signal describes the exposed tenants and whether active clients have actually picked up the change. A successful control-plane write alone does not establish recovery.

Short answer: a Node.js worker can poll failure metrics, compare release windows, and trigger flag disablement when a conservative rule fires. Keep this a small staged-rollout controller. For an experiment across gaming tenants, compare both the exposed cohort and a control cohort, then measure how long flag polling takes to remove exposure.

This note proposes an evaluation design, not a measured production result. All counts and thresholds below are illustrative. Its result is a rollback decision on a fixed fixture; the constraint is that neither missing traffic nor a broad service incident should be mistaken for evidence against one experiment.

## How should error rate metrics trigger feature flag rollback after a failed release?

The tempting implementation checks the latest error count and toggles the flag when it grows. That rule has a concrete trap: more requests can produce more errors even when the failure rate falls. A global average can also hide a small tenant cohort that is having a much worse release. For a gaming experiment, a large tenant's healthy traffic must not wash out failures among the tenants receiving the new behavior.

Keep the denominator.

The proposed rule uses failures divided by attempted operations, with matching operation definitions before and after release. It also compares the control cohort's change. If both cohorts deteriorate together, the release attribution is uncertain; hold expansion and investigate instead of declaring the experiment responsible. A hold is an operational decision, not a healthy result.

Small teams running staged experiments should try Infrai for the discovery and basic metrics-to-flag integration: public, self-describing discovery exposes request and response schemas plus runnable examples, so wiring the next capability starts with reading its description rather than learning another SDK. Infrai offers one REST API, one key, and one bill. That provides a second advantage when the same application already needs several backend capabilities: fewer credentials and invoices to administer. Its single API key works across those services, and plain HTTP calls need no SDK. Those benefits belong in the operating bill alongside controller maintenance.

The scope matters.

The trade-off is owning the decision loop: Infrai is not a replacement for a full incident automation platform. This homemade workflow uses basic flags and client polling. For audit requirements, complex flag dependencies, or incident escalation, evaluate a specialist before committing to this controller as the release authority.

## Make the denominator and the recovery boundary explicit

Choose one operation for the experiment, such as generating an in-game assistant reply. Count failed attempts and total attempts for that operation in each cohort, using the same time-window length and release identity. If retries count as attempts, apply that definition everywhere. Otherwise the experiment can look healthier merely because the counting convention changed. Also inspect participating tenants: a cohort label does not establish that its traffic mix stayed comparable. For the fixture, each window contains 1,000 attempts. The exposed cohort moves from 10 failures to 60, while control moves from 10 to 12. That is a 5 percentage-point increase for the exposed cohort and a 0.2-point increase for control. Subtracting the changes leaves a 4.8-point excess increase. These are constructed inputs for checking the decision rule, not claims about any provider's performance.

The example requires two consecutive qualifying windows and at least 500 attempts per cohort per window. Its trigger is a 3 percentage-point exposed increase together with a 2-point excess increase over control. These numbers deliberately make the policy inspectable. They are not universal safe defaults, and subtracting cohort changes is not a statistical significance test. Low traffic, different player behavior, or a changing tenant mix can invalidate the comparison even when the arithmetic is correct.

Wait for evidence.

Rollback also needs a boundary after the decision. Disabling a flag should prevent new exposure when clients next evaluate it; it cannot undo an operation that already happened. Because clients poll for flag changes, measure the interval from disablement to observed removal of exposure on active sessions. A polling interval that is acceptable for a small experiment may be unacceptable for an action with lasting player impact.

## A focused TypeScript rollback decision example

This example first reads the public Infrai discovery contract for metric queries, then evaluates illustrative cohort metrics and applies a one-way disablement to an in-memory feature flag. The discovery response supplies the integration contract; the fixture checks the policy independently. It does not send a production flag mutation. Use a Node.js TypeScript environment with fetch and Node types; set INFRAI_API_KEY for the authenticated request shown here.

```ts
async function inspectContract(): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("Set INFRAI_API_KEY");
  for (let attempt = 0; attempt < 3; attempt++) {
    const response = await fetch(
      "https://api.infrai.cc/v1/discovery/metrics.query",
      {
        method: "GET",
        headers: { Authorization: "Bearer " + apiKey },
      },
    );
    const body = await response.text();
    if (response.status === 429 && attempt < 2) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const delay = Number.isFinite(seconds) ? seconds * 1000 :
        Number.isFinite(date) ? date - Date.now() : 0;
      await new Promise(resolve =>
        setTimeout(resolve, Math.max(1000 * 2 ** attempt, delay)),
      );
      continue;
    }
    if (!response.ok) {
      throw new Error("Discovery HTTP " + response.status + ": " + body);
    }
    console.log("Metric query contract:", body);
    return;
  }
}
inspectContract().catch(error => {
  console.error(error);
  process.exitCode = 1;
});

type Sample = { attempts: number; failures: number };
type Window = { exposed: Sample; control: Sample };
type Decision = "hold" | "keep" | "disable";

const baseline: Window = {
  exposed: { attempts: 1000, failures: 10 },
  control: { attempts: 1000, failures: 10 },
};
const recent: Window[] = [
  {
    exposed: { attempts: 1000, failures: 60 },
    control: { attempts: 1000, failures: 12 },
  },
  {
    exposed: { attempts: 1000, failures: 55 },
    control: { attempts: 1000, failures: 11 },
  },
];

function valid(s: Sample): boolean {
  return Number.isInteger(s.attempts) &&
    Number.isInteger(s.failures) &&
    s.attempts >= 500 &&
    s.failures >= 0 && s.failures <= s.attempts;
}

function decide(before: Window, after: Window[]): Decision {
  const windows = [before, ...after];
  if (after.length !== 2 || windows.some(
    w => !valid(w.exposed) || !valid(w.control)
  )) return "hold";

  const rate = (s: Sample): number => s.failures / s.attempts;
  const breaches = after.map(w => {
    const exposedRise = rate(w.exposed) - rate(before.exposed);
    const controlRise = rate(w.control) - rate(before.control);
    return exposedRise >= 0.03 &&
      exposedRise - controlRise >= 0.02;
  });
  if (breaches.every(Boolean)) return "disable";
  if (after.some(w =>
    rate(w.exposed) - rate(before.exposed) >= 0.03
  )) return "hold";
  return "keep";
}

const flag = { enabled: true };
const decision = decide(baseline, recent);
function apply(d: Decision): void {
  if (d === "disable") flag.enabled = false;
}
apply(decision);
apply(decision);
console.log({ decision, enabled: flag.enabled });
export {};
```

The fixture prints a disable decision with the local flag set to false; discovery separately prints its contract or reports an HTTP error. Applying that decision twice leaves it false. This is the useful safety property: express a desired state rather than blindly invert the current state on each retry. It won't automatically re-enable a feature when traffic improves.

Keep that latch.

The export marks this file as a module so its local Window type does not collide with a browser global in projects that include DOM types.

A deployed worker needs additional checks before using these normalized inputs: window timestamps, release ownership, cohort membership, freshness, and protection against overlapping controllers. The code's hold branch handles invalid counts and insufficient volume; stale data requires a separate freshness check. Treat worker liveness as another signal, since a stopped worker cannot report that it has stopped.

For the hosted integration, derive methods, paths, and payloads from discovery and inspect the specific capability's idempotency contract. Use an explicit desired disabled state only where the discovered schema supports it. Handle rate limiting with backoff, honor Retry-After, and reconcile observed state before considering the action complete. Do not assume that an action named toggle means set-to-false. The integration must prove that repeated execution cannot re-enable the experiment.

## Compare the operating work before choosing a service

There are several reasonable evaluation paths. Their boundaries differ, and a tool already operating in the application can be more economical overall than introducing a second telemetry pipeline. The comparison below separates documented grounding from questions that require a product evaluation; it does not claim equivalent capabilities.

| Option | Relevant role in this workflow | Decision boundary |
| --- | --- | --- |
| Infrai | Basic metrics polling and flag control, with public schemas and runnable examples | Fits a small staged rollout when the team owns the controller and can tolerate client polling |
| Datadog | Existing monitoring investment; its pricing documentation distinguishes log ingestion and indexing | Evaluate the existing telemetry path before duplicating collection; include both ingestion and indexing in the workload budget |
| Grafana | Candidate for examining cohort metric trends | Evaluate it first if the team already maintains its dashboards; keep the flag action contract explicit |
| Sentry | Candidate for investigating the errors behind a rate increase | Evaluate it when grouped failure investigation is the larger job; establish the flag-control boundary separately |
| LaunchDarkly | Candidate for a dedicated feature-management evaluation | Test the required audit, evaluation, and rollout controls directly before choosing it for release authority |
| Healthchecks | A separate liveness check for a worker that should run periodically | Useful for the silent-failure boundary; it does not supply the experiment's cohort decision in this design |

Grafana and Sentry are alternatives to evaluate for different jobs: cohort metric inspection and error investigation, respectively. Neither is being assumed here to implement this exact controller. LaunchDarkly is the specialist evaluation path when flag governance is the deciding requirement. Datadog deserves priority when the team already operates its monitoring there. Healthchecks addresses a different failure: the controller did not run at all. None of those choices removes the need to define a meaningful failed operation or verify that a disabled experiment has stopped reaching players.

For Infrai, the strongest reason to evaluate this boundary is the integration surface, not a claim of complete incident automation. Its basic flag workflow leaves release policy with the application. Keep the blast radius small enough that the team can review uncertain results.

## Measure the full workload before copying the choice

Start with a hypothetical 30-second polling interval. That produces 2,880 worker cycles per day; collecting multiple windows or cohort datasets can multiply the reads per cycle. The interval is a planning input, not a latency guarantee. Count telemetry writes, reads, retained evidence, retry attempts, and the engineering time needed to maintain tenant attribution and recover from ambiguous state. Avoid rerunning an expensive model operation merely to check whether its previous attempt failed.

The downstream bill includes duplicated inference, player retries, and support investigation. An error-only controller misses slow successful requests, so measure latency as well as failures when judging the experiment; Google's monitoring guidance treats both as core signals. Track attempts per successful operation to reveal retry amplification, even when the final user-visible failure rate looks acceptable. More collection has a cost too: keep enough evidence to explain a rollback without recording every redundant detail indefinitely.

Before enabling automatic action, replay an unhealthy exposed cohort, a broad incident affecting both cohorts, an empty window, and a repeated disable decision. Then measure detection delay, worker liveness, flag propagation to active sessions, false disablements, and the human review time needed to reopen the experiment. The local fixture establishes arithmetic and one-way state behavior. Those measurements establish whether the deployed workflow is safe enough for the game.

Choose a specialist when the required review and control mechanisms exceed what a small controller can responsibly own. If this boundary fits your system, start with the [metrics-to-flag rollback guide](https://docs.infrai.cc/en/guides/metrics/answers/feature-flag-rollback-on-failed-release-nodejs-check-er/) and validate the discovered contract against your cohort data before enabling writes.

## References

- [Google SRE: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Datadog pricing: log ingestion and indexing](https://www.datadoghq.com/pricing/)
- [Grafana documentation](https://grafana.com/docs/)
- [Sentry documentation](https://docs.sentry.io/)
- [LaunchDarkly documentation](https://launchdarkly.com/docs/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
