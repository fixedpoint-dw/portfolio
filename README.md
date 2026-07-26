# Enabling GitHub Pages:

In your repo, click Settings
In the left sidebar, click Pages
Under "Build and deployment" → Source, choose Deploy from a branch
Set the branch to main and the folder to / (root), then click Save

# Frontier — a classroom portfolio game

A real-time, phone-first trading game for teaching Markowitz portfolio choice. Students each start with $100 in a money-market account, trade five stocks whose prices follow correlated geometric Brownian motion, survive surprise liquidity calls (and, if you press the button, a crash), and get ranked by a risk-adjusted score — not raw wealth. Companion piece to the Bubble Lab game: same stack, same classroom flow, and it deliberately calls back to the Kelly game's volatility-drag lesson.

Everything is one file: `index.html`. No build step, no server. Prices are computed deterministically from a shared random seed, so every phone shows the identical market and your device does not need to stay online to keep the game running.

## Deploy in two minutes

1. Put `index.html` in a GitHub repository (its own repo, or a subfolder of an existing Pages site) and enable GitHub Pages: Settings → Pages → deploy from branch.
2. The file already contains the Firebase config for the `bubblelab0715` project you used for Bubble Lab, and this game writes under its own `pf/` path, so the two games coexist without touching each other. If your Realtime Database rules are currently wide open, you're done. If you want to scope them, use:

```json
{
  "rules": {
    "rooms": { ".read": true, ".write": true },
    "pf":    { ".read": true, ".write": true }
  }
}
```

(`rooms` is Bubble Lab's path, `pf` is this game's. Open rules are fine for a classroom toy — there's no personal data — but don't reuse this database for anything real.)

3. **Change the admin password.** Near the top of the script in `index.html`:

```js
const ADMIN_PASSWORD = "efficientfrontier";           // change me
```

To use a fresh Firebase project instead: console.firebase.google.com → create project → Build → Realtime Database → create in test mode → Project settings → Web app → copy the `firebaseConfig` object over the one at the top of the script.

Multiple sections? Append `?room=mba2026` (any name) to the URL — each room is a fully independent game. The default room is `classroom`.

## Running a class

Students open the page and enter a name — that's their whole onboarding. You open the same page, tap "I'm the instructor," and enter the password.

In the lobby you set the game up. The defaults are tuned (12 minutes, 3 s per tick, one tick = one week, so the class lives through about 4.6 market-years; 2 random liquidity calls; 0 random crashes; scoring by risk-adjusted score with γ = 3). Press **Start** and the market opens on every phone simultaneously.

While it runs you can: **Pause** (freezes prices and trading everywhere — good for mid-game discussion), send a **banner announcement**, fire a **liquidity call** or a **crash** manually, switch the **ranking metric live** (the leaderboard re-sorts on every screen — great for the "look who's first now" moment), and watch the live standings, the risk-return map, and the market chart. Firing the crash manually, at a moment of your choosing, is much better theater than leaving it to chance — the class watches in real time who was diversified and who was not.

**End game** (or the clock running out) sends everyone to the results screen: final table with return, volatility, Sharpe, max drawdown and score; the risk-return map with the capital-market line and where every single asset actually landed; a "theory corner" that reveals the true parameters, each asset's geometric drift, the maximum attainable Sharpe ratio and the tangency portfolio; and six discussion questions. "New game, same class" keeps the roster and starts fresh with a new seed.

Students can join late (they start with fresh cash at the current tick), and there is deliberately no restart button for them: one life per game, everyone watches the same standings.

## Why the scoring works (the design answer to "lucky gamblers win")

Ranking by wealth alone makes all-in-on-the-craziest-stock the rational classroom strategy. So the default ranking is a certainty-equivalent score:

**score = annualized log growth − (γ/2) × annualized variance**, with γ = 3 by default.

Sitting entirely in T-bills scores exactly the risk-free rate; taking volatility only pays if it buys enough growth. Two other mechanisms back this up. The **liquidity call** makes cash valuable: everyone must show 20% of wealth in cash on demand; if you're covered nothing happens, but any shortfall is fire-sold at a 25% haircut and the seller is left stranded in cash. (It's a stress test, not a tax — a player who keeps a cash buffer loses nothing, which keeps the game positive-sum and keeps the risk metrics clean for prudent players.) And one asset, MoonShot Ventures, is a trap: the highest expected return on the menu (22%) with 75% volatility, i.e. a *negative* geometric drift of −6%/yr — the Kelly lesson wearing a different hat.

Simulating six archetypal strategies across 200 random 12-minute games with the shipping defaults:

| strategy | wins by wealth | wins by score | avg return | avg vol | avg score |
|---|---|---|---|---|---|
| Gambler (all-in MoonShot) | 28% | **0%** | +193% | 63% | −64% |
| TechBro (all-in TechNova) | 32% | 10% | +63% | 28% | −5% |
| Equal-weight, 90% risky | 1% | 2% | +66% | 19% | 0% |
| Markowitz ~tangency, 70% risky | 16% | **40%** | +41% | 11% | +5.2% |
| Cautious tangency, 40% risky | 2% | 14% | +32% | 6% | +5.2% |
| All cash | 23% | 34% | +20% | 0% | +4.0% |

The gamblers frequently top the *wealth* table — which is exactly the contrast you want on screen — and essentially never win the *score*. The modal score winner is the diversified, moderately-levered tangency-style portfolio, i.e. the thing the course teaches. (All-cash still wins its share of score races purely because 4.6 years is a small sample — realized means are noisy; a nice discussion point in itself. Longer games shrink it.)

If you'd rather run a pure sandbox, set both shock counts to 0 and switch the metric to wealth or Sharpe; everything is a lobby setting, and the metric can also be changed after the fact on the results screen to replay the ranking debate.

## The market, briefly

Five stocks (BlueChip 8%/16%, TechNova 14%/32%, Utilities 6%/11%, Gold 5%/17%, MoonShot 22%/75%) plus T-bills at 4%. Log returns are jointly normal with a realistic correlation matrix (equities correlated 0.15–0.60, gold slightly negative against everything); presets for zero and crisis-level correlations are in the lobby, and you can edit each asset's mean and vol. With the default menu the maximum Sharpe ratio is 0.35 and the tangency portfolio holds all five assets — including a 23% weight on gold despite its feeble standalone Sharpe of 0.06, which is discussion question 3. A manual crash gaps every stock down in proportion to its risk while gold rises (crash beta −0.3).

Trades execute at the current tick's price with a 10 bp fee and never move prices. Under the hood every client recomputes every player's full wealth path from the trade log and the seeded price path, clamping any infeasible trade — so a tampered client can't create money that other phones would honor.

One accounting footnote: because the score penalizes variance of *log* growth (which already embeds −σ²/2 of drag), the effective arithmetic risk aversion is about γ + 1. With γ = 3 the optimal risky share for the default menu is roughly 60–70%. Nobody needs to know this to play; it's here for your own calibration.

## Teaching beats that come up reliably

The wealth-vs-score ranking swap (ask the class to predict the score ranking before you switch the toggle). MoonShot's arithmetic-vs-geometric gap in the theory corner, and its link back to Kelly. Why gold earned a big weight — covariance, not expected return, is what's priced. Who had cash when the liquidity call hit, and what "lazy money" was actually doing. Each dot's distance below the gold line as the price of under-diversification. And the γ-doubling question: the risky *mix* shouldn't change at all, only the cash split — two-fund separation, live on their own leaderboard.
