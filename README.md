# herdr agent priority

Rank the agents that are waiting on you from 1 to 5, jump to the top of that
ranking, snooze the ones waiting on someone else, and pick from a ranked popup.
A plugin for [herdr](https://herdr.dev) ≥ 0.8.2 (tested on 0.8.2), Linux.

With thirty agents the sidebar tells you *that* several want you, never *which*
to do first, so the day goes to whoever finished last. A number you set once,
when the session starts, lets the work be chosen instead of serviced. Priority
1 work never getting reached is the intent — it is the cue to close it.

## Install

```sh
herdr plugin install Mindful-Stack/herdr-agent-priority
```

Then add the keys below to `~/.config/herdr/config.toml` (anywhere after your
`[keys]` table), add the two tokens to your sidebar rows, and run
`herdr server reload-config`.

<!-- keys-snippet -->
```toml
[[keys.command]]
key = "alt+p"
type = "plugin_action"
command = "agent-priority.up"
description = "Raise agent priority"

[[keys.command]]
key = "alt+shift+p"
type = "plugin_action"
command = "agent-priority.down"
description = "Lower agent priority"

[[keys.command]]
key = "alt+z"
type = "plugin_action"
command = "agent-priority.snooze"
description = "Snooze agent (5m/30m/2h/4h/off)"

[[keys.command]]
key = "alt+g"
type = "plugin_action"
command = "agent-priority.goto"
description = "Go to the top-ranked waiting agent"

[[keys.command]]
key = "prefix+p"
type = "plugin_action"
command = "agent-priority.open"
description = "Ranked agent picker"
```

Sidebar: add `"$pri"` (the rank label) and `{ token = "$snz", dim = true }`
(the snooze label) to the agent rows you already have, for example

```toml
[ui.sidebar.agents]
rows = [["state_icon", "$pri", { token = "$snz", dim = true }, "workspace", "tab"], ["agent"]]
```

Merge these into your existing `[ui.sidebar.agents]` rather than pasting a
second one — TOML rejects a table defined twice. The sidebar *order* needs no
config: the plugin sets herdr's agent view to sort by rank. (`ui.agent_panel_sort
= "priority"` is only a fallback if that view is ever cleared.)

## Keys

| Key | Does |
| --- | --- |
| `alt+p` | Raise the focused agent's rank one step (stops at 5). |
| `alt+shift+p` | Lower it one step (stops at 1). |
| `alt+z` | Snooze it: 5m, 30m, 2h, 4h, off — one step per press. |
| `alt+g` | Go to the top of the ranking, cycling within the top band. |
| `prefix+p` | Open the ranked picker. |

`alt+shift+p` needs the kitty keyboard protocol between your terminal and
herdr: in the legacy encoding it is `ESC P`, which starts a DCS string, and
herdr waits for a terminator that never comes. foot, kitty, ghostty and
wezterm negotiate it.

Inside the picker: type to filter (the query survives every action), `enter`
jumps, `alt+p`/`alt+shift+p` nudge, `alt+z` snoozes, `alt+1`–`alt+5` rank
outright (`alt+3` = unranked), `ctrl+backspace` deletes a word, `esc` closes.

`alt+1`–`alt+5` also move the cursor to the row below after ranking
(set-and-advance), so walking down a list and ranking as you go keeps landing
on the row that was visually next, instead of the re-sorted list carrying the
cursor away with the row you just ranked. `alt+p`, `alt+shift+p` and `alt+z` do
not advance, since a repeated press of those must keep hitting the same row.
The list wraps (the picker sets `--cycle`), so ranking past the last row moves
to the first rather than stopping.

`alt+0` clears the ranks of every row the current filter matches — type a
workspace, a tab, or part of a title first, then press it, rather than
`alt+3`-ing each row by hand.

## The scale

Five is most important, one least, and **three means unranked**: landing back
on three removes the label rather than writing `P3`, because unset already
sorts as three and thirty meaningless labels are noise.

## alt+g

It goes to the **top band** — every waiting agent sharing the highest rank —
and cycles within it. It never drops you onto a 4 because you pressed again:
pressing past what you called most important is how a ranking rots into a
to-do list. Waiting means blocked, done or idle; working and snoozed agents
are skipped. When the band is just the agent you are on, it says so and names
your demote and picker keys.

## Snooze

For "waiting on someone else". A snoozed agent leaves the alt+g queue until
its time is up, then simply counts again. The sidebar shows the time left,
`zz 24m`, refreshed once a minute by one small background process that exits
when nothing is snoozed; the picker shows the wake time.

## Requirements

`jq`, `fzf` (recent: the picker uses `--track`, `--id-nth` and
`--with-shell`), `python3` ≥ 3.11, util-linux `flock` and `setsid`.
`notify-send` is optional (rank and snooze toasts).

## State

Everything lives in herdr's state directory for this plugin,
`${XDG_STATE_HOME:-~/.local/state}/herdr/plugins/agent-priority/`:

| File | Holds |
| --- | --- |
| `agent-priority` | The ranks — the source of truth. |
| `agent-snooze` | Active snoozes: pane, wake time, step. |
| `agent-priority.lock` | Serialises every store write and label write. |
| `agent-snooze.ticker.lock` | Held by the one label ticker. |
| `agent-priority.log` | A line per invocation. |
| `agent-priority.selection`, `agent-priority.notify-id` | Picker hand-off, toast id. |

Running the script by hand from a plain shell, point it at that directory, or
it uses a separate fallback (`…/state/herdr-agent-priority/`):

```sh
HERDR_PLUGIN_STATE_DIR=~/.local/state/herdr/plugins/agent-priority \
  "$(herdr plugin list | sed -n 's/.*agent-priority.*\[[a-z]*:\(.*\)\]/\1/p')/agent-priority.sh" list
```

## When a key looks dead

Read `agent-priority.log` in the state directory first, then
`herdr plugin log list --plugin agent-priority`. Keybind commands run with no
terminal attached, so a failure is otherwise silent.

## Development

```sh
herdr plugin link /path/to/checkout
sh tests/test.sh
```

The offline suite drives the real script against stub `herdr`, `setsid`,
`fzf`, `sleep` and `notify-send` on `PATH`, in a temp `HOME`/`XDG_STATE_HOME`
with a socket path that does not exist, so it never touches a live session.

## License

MIT — see [LICENSE](LICENSE).
