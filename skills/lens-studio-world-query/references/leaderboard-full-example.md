# Leaderboard — Full Example Reference

Sourced from `Essentials/Assets/Leaderboard/LeaderboardExample.ts`.

## Create + populate + display a leaderboard

```typescript
@component
export class LeaderboardExample extends BaseScriptComponent {
  @input leaderboardModule: LeaderboardModule
  @input textLogs: Text

  private leaderboard: any
  private readonly BOARD_NAME = 'MY_LEADERBOARD'

  onAwake(): void {
    if (!this.leaderboardModule) {
      this.leaderboardModule = require('LensStudio:LeaderboardModule')
    }
    this.createLeaderboard()
  }

  // ── Step 1: Create or retrieve ──────────────────────────────
  private createLeaderboard(): void {
    const options = Leaderboard.CreateOptions.create()
    options.name = this.BOARD_NAME
    options.ttlSeconds = 86400                            // 24 hours
    options.orderingType = Leaderboard.OrderingType.Descending

    this.leaderboardModule.getLeaderboard(
      options,
      (lb) => { this.leaderboard = lb; this.readBoard() },
      (status) => print('Create failed: ' + status)
    )
  }

  // ── Step 2: Read current entries ────────────────────────────
  private readBoard(): void {
    const opts = Leaderboard.RetrievalOptions.create()
    opts.usersLimit = 10
    opts.usersType = Leaderboard.UsersType.Global   // or Friends, User

    this.leaderboard.getLeaderboardInfo(
      opts,
      (others, me) => {
        if (me) {
          this.log(`My score: ${me.score}`)
        }
        others.forEach((r, i) => {
          if (r?.snapchatUser) {
            this.log(`#${i+1}: ${r.snapchatUser.displayName ?? 'Unknown'} — ${r.score}`)
          }
        })
        this.submitMyScore()
      },
      (status) => { print('Read failed: ' + status); this.submitMyScore() }
    )
  }

  // ── Step 3: Submit ──────────────────────────────────────────
  private submitMyScore(): void {
    const score = Math.floor(Math.random() * 1000)

    this.leaderboard.submitScore(
      score,
      (userInfo) => {
        this.log(`Submitted ${score}`)
        if (!isNull(userInfo)) {
          this.log(`Confirmed: ${userInfo.snapchatUser.displayName} = ${userInfo.score}`)
        }
      },
      (status) => print('Submit failed: ' + status)
    )
  }

  // ── Helper: DelayedCallbackEvent timer ──────────────────────
  scheduleAfterDelay(cb: () => void, seconds: number): void {
    const ev = this.createEvent('DelayedCallbackEvent')
    ev.bind(cb)
    ev.reset(seconds)
  }

  private log(msg: string): void {
    print('[LB] ' + msg)
    if (this.textLogs) {
      this.textLogs.text = (this.textLogs.text || '') + '\n' + msg
    }
  }
}
```

## Key gotchas

- `Leaderboard.UsersType.Global` vs `Friends` vs `User` — `Friends` requires Snapchat login.
- `ttlSeconds = 0` → board never expires (fills quota). Use e.g. `86400 * 7` for weekly.
- `Descending` = higher score is rank 1. Use `Ascending` for best-time or golf-style scores.
- `isNull(userInfo)` — use Lens Studio's `isNull()` helper, not `=== null`, for JS-bridged objects.

