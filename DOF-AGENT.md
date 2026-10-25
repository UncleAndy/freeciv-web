# DOF agent: the built-in freeciv AI with the DOF principle

This image source carries a modified *built-in* AI: the seats that the game
engine plays itself (the seats no human or external client controls) act by the
DOF principle — degrees of freedom instead of the classic "grab what pays off"
heuristics. The modification lives in the Freeciv C server, not in the Python
client and not in any external bot, so it works in single-player, in headless
tournaments and in multiplayer alike.

Principle: <https://github.com/UncleAndy/dof-core> (DOF-Core).
Implementation and measurements in the game: `deploy/freeciv-dof` of the
DOF-MODEL-CIV project.

## What the agent does differently

| rule | classic AI | DOF seat |
|---|---|---|
| war without a reason | declares war for profit, for plunder, "no mercy" on the weak | never: war desire is zeroed, the plunder war and the self-targeting are removed, war for an ally and war to stop a spaceship are refused |
| who may be attacked | anyone who looks weak | only a *source of aggression* — a player observed attacking a city or a unit |
| espionage as an attack | bribes cities, sabotages, steals tech, poisons, delivers a nuclear strike by spy | refused unless the target is a source of aggression; embassies and peaceful presence stay free |
| response to an attack | ordinary war planning by expected profit | war of response: the offender is attacked without a profit calculation |
| solidarity | a war next door is somebody else's business | a seat that knows both sides (it has met the attacker and the victim) treats the attacker as a source even when it was not the victim, and joins against it |
| ending a war | tie when profitable | a ceasefire is offered once the source flag expires ("We have no quarrel left with you") |

A stranger is not a target: an unmet player is *unknown*, not hostile. Unknown
is not zero — the agent does not treat the absence of information as the absence
of options.

## Code

| file | role |
|---|---|
| `ai/default/dofpolicy.c`, `.h` | ledger of aggression sources, solidarity view (who attacked somebody this seat knows), rule journal, per-seat policy switch, counters |
| `ai/default/daidiplomacy.c` | war desire, declaration guard, war of response, ceasefire offer |
| `ai/default/daiactions.c` | refusal of coercion (bribery, sabotage, tech theft, poison, spy nuke) |
| `ai/default/daiunit.h` | a stranger is not a `POTENTIALLY_HOSTILE_PLAYER` |
| `ai/default/daisettler.c` | NULL-guard in `find_best_city_placement` (a crash on long headless games) |
| `ai/classic/classicai.c` | `/aicmd <player> dof on\|off\|status` console command |
| `server/aiiface.c` | per-turn hook: read the policy, note the incidents |
| `server/srv_main.c` | turn advance without a connected client (headless games) |
| `meson.build` | the new sources in the server build |

The whole change is one patch: `freeciv/patches/dof-principle.patch`, applied by
`freeciv/apply_patches.sh` together with the other freeciv-web patches.

## Policy switch

The policy is a file read at the start of every AI turn, so it can be changed
without a restart (`DOF_POLICY_FILE` in the environment overrides the path):

```
/docker/freeciv/dof_policy.conf      # shipped default: all
```

```
none            # classic AI everywhere (control arm)
all             # DOF everywhere
even | odd      # DOF on AI seats with even / odd player index
0,2,4           # explicit seat list
```

`even` and `odd` are how the two arms are mixed inside one game: half the seats
are DOF, half stay classic, on the same map and the same seed.

## Journal

The agent writes its decisions to the server output, one line per event:

```
[DOF] policy=all turn=1
[DOF] roster ai_seats=[Tuldila:idx=0:dof=1,Pakal:idx=1:dof=1,...]
[DOF] contact turn=6 me=Pakal other=Burnaburish II status=MET
[DOF] declare turn=58 declarer=... target=... declarer_dof=1 reason=3
[DOF] news turn=61 violator=... victim=... violator_dof=1 victim_dof=0
[DOF] rule turn=142 me=... other=... what=war_on_source_third
[DOF] rule turn=148 me=... other=... what=ceasefire_offer
[DOF] rule turn=151 me=... other=... what=coercion_refused
[DOF] blocked turn=64 me=... other=... what=declare_war
```

`war_on_source` marks a war of self-defence, `war_on_source_third` a war of
solidarity. The same lines are duplicated to stderr, because the `--log` file is
not always written in this container layout. Counters of the run: refused
coercion, peace offers, wars started on an observed attack, suppressed
war-desire pairs.

## Building

Full build — the patch is part of `PATCHLIST`, so it is applied to the engine
source fetched at build time:

```
docker build -t civrealm/freeciv-web:1.5 .
```

Fast path — reuse the build directory of the base image and recompile only the
engine (a couple of minutes instead of a full rebuild):

```
docker build -f Dockerfile.dof -t civrealm/freeciv-web:1.5 .
```

## Measured effect

Headless tournament series, 8 AI seats, 200 turns, same seeds per arm
(DOF-MODEL-CIV, `deploy/freeciv-dof/logs/series`):

| arm | war declarations | of them unfounded | techs per seat | science, bulbs/turn |
|---|---|---|---|---|
| classic | 138 | 108 (96%) | 40.8 | 144 |
| DOF | 0 | 0 | 47.2 | 395 |
| half DOF, half classic | 44 (3 by DOF seats) | — | 39.8 | 134 |

The DOF seats neither start wars nor coerce anyone; the resources that classic
seats spend on wars go into growth and science instead.

Solidarity series (6 games per arm, 200 turns, size 3, same seeds), mixed arm
split by seat:

| seats | war declarations | of them unfounded | score | science, bulbs/turn |
|---|---|---|---|---|
| DOF seats, mixed game | 12 | 0 | 175.8 | 156 |
| classic seats, same games | 49 | 41 (84%) | 157.5 | 98 |
| classic control | 138 | 130 (94%) | 172.1 | 144 |

All 12 DOF declarations came after an attack by the target and all 12 were
journaled as `war_on_source_third`: these are wars for third parties. The
control arm reproduced the earlier series exactly, so the solidarity change
does not touch a classic seat. Price of solidarity: about 12% of the science
(177 → 156 bulbs) at an unchanged score.
