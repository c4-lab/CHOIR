# Extracting the llm_trading_sim market as a world module

Version 0.1, October 2026. For Brett. Companion to the platform specification (v0.1), which defines the oracle, the clock modes, and the world-module contract this work targets.

## Goal

Lift the market mechanics out of Alejandro Lopez-Lira's `llm_trading_sim` (github.com/alejandroll10/llm_trading_sim) and package them as a world module for our platform. The module holds the order book, accounts, the dividend and fundamental-value process, and clearing. It contains no agents, no language-model calls, and no plotting. Our oracle drives it; our agent harness supplies the decisions.

The work succeeds when the extracted module, driven by a recorded sequence of orders, reproduces the original simulator's trades and prices round for round.

## Why this repository, and what it costs

The repository is a reasonable starting point for three reasons:

- **Small dependency footprint:** pure Python with numpy, pandas, and pydantic, and an OpenAI-compatible client that already points at local vLLM.
- **A tested market core:** a persistent order book with market and limit orders, partial fills, dividends, short selling, and leverage, plus a unit test suite, a health-check script, and a per-round simulation verifier.
- **Existing hooks and structure:** a discrete-round loop with before and after hooks, scenarios declared in YAML, and persona prompts already pinned by sha256 in the tests.

It also has three structural differences from our design:

- **Inverted control:** the simulation owns the round loop and calls the agents. In our platform the oracle owns the clock and the agents call in.
- **Batch matching:** matching is designed per round. Within a round, the order of submissions is randomized, then a staged procedure nets market orders against each other, executes them against the book, and matches crossing limit orders. This fits our sync mode directly and needs a decision for the async modes (see Async clearing).
- **Agent-held state:** prompt construction, memory notes, and the social feed sit on the agent side of the code, and position tracking may be entangled with agent objects. All of these need to move.

**License.** The repository has no LICENSE file, and `pyproject.toml` declares no license. Without one, default copyright applies, which is a problem for the release of the testbed and benchmark configurations that the proposal promises. Before writing code, email the author and ask whether he will add a permissive license (MIT or BSD). Work in a private fork until that is settled. If he declines, use the fallback described at the end.

## Stage 0: map the code

Clone the repository, install with `pip install -e .[dev]`, and run `pytest tests/ -q` and `python scripts/health_check.py --quick`. Run several of the deterministic scenarios, which need no API key.

Then map what `BaseSimulation.execute_round` in `src/base_sim.py` touches. The dev extras include call-graph tooling (`src/dependency_tree.py`, `src/analyze_execution.py`, and pyan3), so use it.

The paper names the services you are looking for: `MatchingEngine`, `OrderMatchingService`, `TradeExecutionService`, and `OrderStateManager`. The imports are flat and rooted at `src/` (for example `from market...`), so the market code most likely lives in a `market` package.

**Deliverable:** a one-page map that assigns every module to one of six categories:

- **world:** keep and extract.
- **agent:** port to the harness.
- **LLM:** discard.
- **logging:** replace with our event log.
- **analysis:** keep for reference.
- **glue:** discard.

Flag every place where world code imports agent code, or where agent objects hold cash, shares, or order commitments. Those crossings are the main surgery. Bring the map to a meeting before Stage 2.

## Stage 1: record golden traces

Before changing anything, record reference runs from the unmodified simulator.

1. Patch the original so that every random draw (dividends, within-round order shuffling, any agent randomness) comes from a single seeded generator passed in at construction. If the code uses Python's global `random` or numpy's global state, this patch is required for the comparison in Stage 3 to be possible. Commit it separately.
2. Using deterministic agents only, record for each round: every submitted order (agent id, side, quantity, type, limit price, replace decision), the shuffled processing order, every trade, the last price, best bid and ask, book depth, every account balance, and the fundamental value.
3. Record at least the basic, short-selling, leverage, and regime-shift scenarios, each under several seeds.

These traces are the reference. They let you change the code aggressively and detect any behavioral change.

## Stage 2: extract the market core

Create `worlds/market/` in our repository and move the market code into it. Rules:

- **No imports from agent, LLM, plotting, or sweep code.** If a market module needs something from those, the dependency is the thing to cut.
- **Accounts live in a world-owned ledger keyed by agent id.** Cash, shares, borrowed cash, borrowed shares, dividend account, and committed resources move from agent objects (if they live there) into the ledger.
- **The world owns all randomness** through the generator it receives in `init`.
- **State is data.** The world returns structured values. Prompt text is never generated inside the world.

## Stage 3: wrap the core in the world-module contract

Implement the contract from the platform specification:

- **`init(config, rng)`:** build the book, ledger, dividend process, and regime schedule from a scenario-style configuration. Keep their YAML parameter names where possible, so existing scenarios translate directly.
- **`channels()`:** declare the observation channels and their default policies. A natural first set:
  - `last_price`: last price, volume, and round.
  - `price_history`: the last k rounds of price and volume.
  - `depth`: the book to depth d. The original truncates depth per agent, so make d a per-agent channel parameter.
  - `position`: the agent's own ledger entry and its resting orders.
  - `dividends`: realized and expected dividends, per the information mode.
  - `fundamental`: available or unavailable per agent, which implements their `FUNDAMENTAL_INFO_MODE` as channel availability.
- **`tool(...)`:** none for now. The news feed comes later, from our side.
- **`submit(agent_id, action, t)`:** validate against the ledger, as the original does, and return an acknowledgment that states whether the order was accepted, resized, or rejected. Rejections are events for the log, not exceptions.
- **`advance(t)`:** run the matching sequence and pay dividends and interest. Return trades, fills, and account changes as events.
- **`default_action(agent_id)`:** no new orders, with resting orders left in place. Write this choice down, because it affects what a timeout means.
- **`charge(agent_id, amount)`:** debit the ledger for information costs, in a cash account kept separate from trading cash.
- **`state(t)`, `metrics(t)`, `ground_truth(t)`:** the market-level metrics their analysis already uses (price relative to fundamental, spread, depth, volume), plus the fundamental and dividend realizations as ground truth. Their per-round verifier becomes an internal assertion inside `advance`.

**Equivalence test.** Drive the wrapped world with the golden orders and the golden processing order, and compare trades, prices, and balances round for round. The results should be identical. Any difference is either a bug in the extraction or a hidden dependency on agent state, and you should be able to say which before moving on.

## Stage 4: port the agent side

These pieces move to the agent harness that the PhD student is building. Port them, but keep them out of the world module.

- **Deterministic agents:** each subclasses `BaseAgent` and implements `make_decision()`. Wrap each one as a `rule` policy that takes our observation payloads and returns our action schema. The registry in `src/agents/deterministic/deterministic_registry.py` lists them.
- **Persona prompts and templates:** the persona files in `src/agents/prompts/` and the user-prompt templates (market state, depth, position, history, fundamentals, trading options) become recipe prompt templates. The templates render our observation payloads into text, and the harness does the rendering. Carry over their sha256 pinning as our recipe hashes.
- **Decision schema:** their `TradeDecisionSchema` and `OrderSchema` (pydantic) can serve as the guided-decoding schema for vLLM. Two notes:
  - **Schema complexity:** their README warns that llama-3.1-8b-instruct fails their structured-output validation. Guided decoding constrains the format, but semantic errors such as impossible quantities will remain. Build a simplified pilot schema as well (one order per decision, plus the belief fields) and let the recipe choose.
  - **Belief fields:** keep `valuation` and `price_target` in every schema. They are an agent's stated belief about value and its expectation of next-round price, recorded every round. That is a ready-made semantic coordinate for the field, so stance does not have to be extracted from free text.
- **Memory and social features:** their `notes_to_self` memory becomes harness memory. Their shared social feed becomes a broadcast message channel in the oracle, which puts it under channel policy.

## Async clearing

The original matching procedure is a batch operation, and that defines how the market works in each clock mode:

- **sync:** call `advance` once per round, as the original does.
- **async modes:** orders arrive at different virtual times. The simplest step is a frequent batch auction: the world collects orders that arrive during a fixed virtual interval and clears them with the existing engine at the end of the interval. This reuses the engine unchanged, and the interval becomes a configuration parameter (and a market-design lever in its own right).

Continuous matching on every arrival is a later extension. In that mode the market-to-market netting stage has no natural meaning, so it would be a change to the engine, not a wrapper.

## Warm-up: an El Farol world

Before Stage 2, write an El Farol world against the same contract: attendance, capacity, payoffs, attendance history as a channel, and a news channel driven by a latent variable. It takes a day or two. It teaches the contract before you do the harder extraction. It also gives the PhD student a real world module to test the oracle against while the market work is underway, and it is the pilot's first domain.

## Fallback

If the license cannot be resolved, or if the Stage 0 map shows that the market core cannot be separated without rewriting most of the matching code, change sources. The fallback has two levels, in this order.

1. **ABIDES components inside our world module.** ABIDES-Markets is BSD-3 licensed and contains a limit order book and fundamental-value processes as separate classes. Using only those classes inside `worlds/market/` keeps our oracle, clock, and harness intact. Check that the order book class can be used without the ABIDES kernel, since that decides whether this option holds.
2. **The full ABIDES kernel as the world.** This is the most work and the poorest fit. ABIDES has its own discrete-event kernel and its own agents. Every external language-model agent would need a proxy agent inside ABIDES that relays messages to our oracle, and the two clocks would have to be reconciled. Consider this only if the order book cannot be separated from the kernel.

Whichever source is used, the contract and the equivalence-test method stay the same.

## Checkpoints

1. Stage 0 map reviewed.
2. El Farol world passing the platform's rule-agent acceptance tests.
3. Golden traces recorded.
4. Equivalence test passing on all recorded scenarios.
5. Deterministic agents and one persona running through the harness in sync mode.
6. Batch-auction clearing running in async-virtual mode.