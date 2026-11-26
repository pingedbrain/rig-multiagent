> **Moved.** `rig-multiagent` now lives in [pingedbrain/rig-contrib](https://github.com/pingedbrain/rig-contrib/tree/main/crates/rig-multiagent), together with the other Rig companion crates. This repository is archived.

# rig-multiagent

Multi-agent patterns for [Rig](https://github.com/0xPlaygrounds/rig): a **Swarm** of
agents that hand work to each other, and a **Graph** of agents, swarms, graphs and plain
functions with conditional edges and parallel branches.

Rig composes agents by making one agent a tool of another. This crate adds the two
patterns where no single agent is in charge.

```rust
use rig_multiagent::{Graph, GraphState, Swarm};

// Agents that pass the work on: each gets a `handoff_to_agent` tool.
let team = Swarm::builder()
    .agent(explorer)       // named Rig agents; descriptions tell the others what each is for
    .agent(writer)
    .max_handoffs(6)
    .build()?;
let run = team.run("What does this project do?").await?;
println!("{} answered: {}", run.agent, run.answer);

// Steps with dependencies: strengths and risks run in parallel, report joins them.
let review = Graph::builder()
    .node("explore", explorer)
    .node("strengths", strengths)
    .node("risks", risks)
    .node("report", report)
    .edge("explore", "strengths")
    .edge("explore", "risks")
    .edge("strengths", "report")
    .edge("risks", "report")
    .build()?;
println!("{}", review.run("Review src/tools.rs").await?.answer());
```

## Swarm

The run starts with the entry agent (the first one added, or `entry(name)`). Each turn,
the agent sees the user request, the handoff message it was given, who worked on the
task before, what they shared, and the other agents with their descriptions. It either
answers, and that answer is the swarm's, or calls `handoff_to_agent(agent_name, message,
context?)`. A handoff ends the agent's turn right there (a Rig hook stops the run), so it
does not keep working on what the next agent now owns, nor spend tokens on it.

Every turn starts fresh; what one agent tells the next travels in the handoff message
and the shared context, which every later agent sees. A handoff to an unknown agent, or
to itself, is a tool error that names the valid agents, so the model can correct itself.

| Limit | Default | Beyond it |
|---|---|---|
| `max_handoffs` | 20 | `SwarmFailure::MaxHandoffs` |
| `agent_timeout` | 5 min | `SwarmFailure::Timeout` |
| `execution_timeout` | 15 min | `SwarmFailure::Timeout` |
| `detect_repetition(window, min_unique)` | off | `SwarmFailure::Repetitive`: the last `window` turns went to fewer than `min_unique` agents |

A `SwarmRun` has the answer, the agent that gave it, every turn and handoff in order,
the token usage of every model call (stopped turns included) and the elapsed time. A
`SwarmError` carries the same trail up to the failure.

The swarm adds `handoff_to_agent` to each agent's tool set, which clones of the agent
share.

## Graph

A run starts with the entry nodes (by default, those without incoming edges), batch by
batch: the nodes of a batch run in parallel; each node that finished follows its
outgoing edges whose condition holds (`edge_if(from, to, |state| ...)`), and the nodes
they reach make the next batch, each once. A node gets the task if nothing led to it, or
else the task plus the outputs of the finished nodes with an edge to it that holds:

```text
Original task: Review src/tools.rs

Inputs from previous nodes:

From strengths:
...

From risks:
...
```

A node is anything implementing `Node`: a Rig `Agent`, a `Swarm`, another `Graph`, or an
async function through `node_fn`, for deterministic steps between agents (parsing,
validation, a lookup) that need no model.

Cycles are allowed, for loops like a reviewer sending a draft back to its writer until
it approves; a node that runs again sees the newest outputs. A graph with a cycle must
set `max_node_executions`, or it does not build.

A failed node does not follow its edges, so its branch stops while the others carry on;
the run then returns a `GraphError` with every failure and everything the rest produced.

| Limit | Default | Beyond it |
|---|---|---|
| `max_node_executions` | none (required with cycles) | `GraphFailure::MaxNodeExecutions` |
| `node_timeout` | 5 min | `GraphFailure::NodeTimeout` |
| `execution_timeout` | 15 min | `GraphFailure::Timeout` |

`GraphRun::answer()` is the output of the last node to finish (or of each node in the
last batch, under its name); `output(id)` reads any node's.

## Streaming

`stream(task)` runs a swarm or graph while reporting what happens, so a UI or CLI shows
progress instead of waiting a minute for the answer:

```rust
use futures::StreamExt;
use rig_multiagent::SwarmEvent;

let mut events = team.stream("What does this project do?");
while let Some(event) = events.next().await {
    match event {
        SwarmEvent::AgentStarted { agent } => println!("\n[{agent}]"),
        SwarmEvent::Text { text, .. } => print!("{text}"),        // as the model writes it
        SwarmEvent::Handoff(handoff) => println!("\n-> {}", handoff.to),
        SwarmEvent::Finished(outcome) => println!("\n{:?}", (*outcome).map(|run| run.answer)),
        _ => {}
    }
}
```

| `SwarmEvent` | `GraphEvent` |
|---|---|
| `AgentStarted { agent }` | `NodeStarted { node }` |
| `Text { agent, text }` | `Text { node, text }` |
| `Handoff(Handoff)` | `NodeFinished { node, output }`, `NodeFailed { node, error }` |
| `Finished(Box<Result<SwarmRun, SwarmError>>)`, last | `Finished(Box<Result<GraphRun, GraphError>>)`, last |

Agents run in Rig's streaming mode, so their text arrives as the model writes it; a
swarm or graph used as a graph node streams its agents' text under that node. A custom
`Node` streams through `NodeIo::text`; one that does not reports only its output.
The run advances only while the stream is polled, and dropping the stream stops it.

## A person in the loop

A streamed run can stop to ask a person: to approve a step, pick an option, or supply a
fact. The question arrives as a `Question` event; the part of the run that asked waits
until you answer it, and time spent waiting does not count against the run's timeouts.

```rust
let mut events = graph.stream("Ship the release");
while let Some(event) = events.next().await {
    match event {
        GraphEvent::Question(question) => {
            println!("{} asks: {}\n{}", question.from(), question.question(), question.context());
            question.answer(read_line()); // dropping it unanswered fails the asker
        }
        GraphEvent::Finished(outcome) => { /* ... */ }
        _ => {}
    }
}
```

- **In a graph**, `human(question)` is a node that asks, with its input as the
  question's context, and outputs the answer. Conditional edges act on it; to gate work,
  put the condition on the edges into the gated node, from the human node and from the
  work it needs:

  ```rust
  let approved = |state: &GraphState| state.output("approve") == Some("yes");
  Graph::builder()
      .node("build", builder)
      .node("approve", human("Deploy this release? (yes/no)"))
      .node("deploy", deployer)
      .edge("build", "approve")
      .edge_if("approve", "deploy", approved)
      .edge_if("build", "deploy", approved) // deploy also gets the build's output
  ```

- **In a swarm**, `allow_questions()` gives every agent an `ask_human(question)` tool.
  The agent waits for the answer within its turn and carries on with it. In a plain
  `run()` nobody can answer, and the tool tells the agent to decide on its own.
- A custom `Node` asks through `NodeIo::ask`.
- Questions from a swarm or graph nested in a graph come out of the outer run, named by
  where they came from (`team/planner`).

A run waits in memory for its answers; resuming a run after a restart is not supported.

## Resuming runs

Give a swarm or graph a [rig-memory](https://crates.io/crates/rig-memory) store, and runs
started with `run_as(run_id, task)` or `stream_as(run_id, task)` save their progress as
they go. Calling it again with the same `run_id`, in the same process or after a
restart, resumes the run where it stopped, or starts it if it finished or never ran.

```rust
let graph = Graph::builder()
    .checkpoints(rig_memory::DirStore::open("runs")?) // or MemoryStore, rig-sqlite, rig-postgres...
    // ...
    .build()?;
let run = graph.run_as("release-1.2", "Ship 1.2").await?;
```

- A swarm saves before every turn; a resumed run starts with the turn it was on,
  keeping the handoffs, shared context and usage before it.
- A graph saves as each node finishes and at the end; a resumed run runs only the
  nodes that had not finished (a parallel batch cut short keeps its finished nodes), and
  a run that ended with failed nodes runs those nodes again (a transient provider error,
  say) without redoing the rest.
- **Tools run once.** Every agent run (a swarm turn, an agent node) keeps a journal of
  its successful tool results. When a cut-short turn or node runs again and its model asks
  for the same tool with the same arguments, it gets the recorded result and the tool does
  not run: the mail is not sent twice. A call with other arguments runs for real; a call
  that failed runs again; the swarm's handoff always runs.
- **Answers are kept.** A person's answer to `ask_human` is in the journal too, so a
  resumed turn does not ask again. A question still waiting survives a restart: the
  resumed run asks it again, so the answer can come from another process, hours later.
- **Nested runs resume inside.** A swarm or graph node keeps its own progress, in its own
  store or else its parent's, under an id derived from the parent run and the node; when
  the node runs again, it resumes where it was rather than starting over.
- The time a run had counted carries over, so its timeouts still hold.
- A streamed resumed run starts with a `Resumed` event.

What still repeats: the model calls of a turn or node that was cut short (their
replies can differ, which is why a different tool call runs), and a tool call that was
running at the moment the process died, since its result never reached the journal
(at-least-once; give such tools an idempotency key of their own if that matters). A
run id is 1-128 of `[A-Za-z0-9_-]`; a run is one JSON file,
`rig-multiagent/{swarm,graph}/<run_id>.json`, plus one per journal and nested run,
each written atomically.

## Prompt caching

Each conversation of a run, a swarm agent's turn or an agent node's run, sends one
prompt-cache key (`GenerationOptions::cache_key`) with all its model calls, so gateways
that route by it (OpenCode Go, OpenAI's prompt cache) keep its calls together and reuse
their cached prefix; without one, Rig sends a fresh id with every call. A checkpointed
run's keys are stable, so a resumed turn or node continues its cache. An agent that sets
its own key keeps it.

## Tracing

Runs are `swarm.run` / `graph.run` spans, with a `swarm.agent` or `graph.node` span per
turn; Rig's own spans for model calls and tools nest under them.

## Compared with Strands Agents

Modeled on the [Strands](https://strandsagents.com) `Swarm` and `Graph`, with the same
semantics where they overlap: the handoff tool and the context it builds, the limits,
the repetition check, batch execution, conditional edges, input from dependencies,
pruning of failed branches, and nesting. Differences: a handoff ends the agent's turn
at once; a cyclic graph needs an execution bound; edge conditions are plain closures
over the run state; a person is asked through a stream event rather than an interrupt
that ends the run; checkpoints go to any rig-memory store, Strands' being its session
managers.

## Testing

`cargo test` runs swarms and graphs on Rig's scripted mock models and plain functions,
with no keys or network: handoffs, shared context, refused handoffs, every limit,
parallel branches (timed), joins, conditional edges, review loops, failures, timeouts,
nesting and usage accounting; streamed runs on scripted streaming models, with event
order, nested streaming and cancellation by dropping the stream; approval gates, agents
asking, dismissed questions, and timeouts that stop while a person thinks; resuming a
swarm after a failed turn and after a restart, rerunning only a graph's failed node, a
question answered by a later process, finished runs starting over, and bad run ids.
