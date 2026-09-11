## The Problem with Simple Threshold Alert
Alerting when a mertic drops below a certain level is very noisy. A quick 2-min drop can cause a lot of alerts and wake up the on-call engineers for essentially nothing.

## The SLO Approach
Instead of watching the metrics, we define an **error budget**. It is the acceptable amount of failure over a period (e.g. 5% over 30 days). 
Then we define alerts based on how fast we're consuming that budget.

## Burn Rate
$$\text{Burn Rate} = \frac{\text{Current Error Rate}}{ \text{Error Budget}} $$
- `1x` = Spending budget at a normal pace, exhausted at 30 days.
- `10x` = Spending the budget very fast, exhausting in 3 days.
- `0.5x` = Healthy, spending half the budget

## Multiple Windows
Each alert uses multiple windows to detect spikes and long-lasting errors.

---

## SLO Tuning & Alerting (Quick Heuristics)

**The 30%–60% Budget Rule**

* **Sweet spot:** Aim to end the month with 30% to 60% remaining error budget.
* **100% left?** SLO is too lenient, or we're moving too slowly.
* **0% left regularly?** SLO is too strict, causing useless alert noise.

**Reading Budget Graphs**

* **The Cliff (Fast Drop):** Real incident, broken dependency, or failed deploy. *Action: Trigger a page.*
* **The Slope (Slow Drop):** Baseline system noise (network blips, client timeouts). *Action: Lower the SLO target to match reality or ignore non-critical errors.*
* **The Jump (Fast Rise):** Normal rolling-window behavior. An old incident from 30 days ago just fell off the back of the window.
* **Steady Rise:** Service is running cleanly. Good time to ship features.

**Golden Rule:** Always set the SLO to match what the service *actually* achieves today + a tiny buffer, not an arbitrary "three nines" goal.