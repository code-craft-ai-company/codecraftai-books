# Foreword

There is a standard story about CI pipelines that goes like this: in the
beginning a team writes a fifty-line YAML file in their CI vendor's
syntax.  The file describes three steps — build, test, deploy — and it
fits on one screen.  Six months later it is six hundred lines.  Forty
of those lines are environment-variable exports, half of which have
unclear ownership.  Twelve of them are `if-else` branches keyed off
`$CI_PIPELINE_SOURCE`.  Three are commented-out blocks marked
`TODO: clean this up`.  The team avoids touching the file because
nobody can predict what will break.  When a pipeline fails, the
standard response is to retry it; the standard response when it fails
again is to find the engineer who last touched it.

This book is not about that file.  This book is about the discipline
that keeps that file from happening in the first place.

The thesis is that integration-level automation — the layer above
individual jobs and below the CI vendor — has six structured concerns:
**Specifications**, **Landscapes**, **Topologies**, **Runtimes**,
**Credentials**, and **Data Profiles**.  Each is a typed configuration.
Each lives in a file (or a database) with a schema.  Each has a name
that other configurations and orchestrations refer to.  None of them
is an environment variable.

When those six concerns have first-class typed expression, the
hundred-line vendor YAML collapses to thirty.  Most of the thirty is
the workflow itself, expressed in a tool-neutral form that can run on
GitLab CI, GitHub Actions, Jenkins, or your laptop.  The "it works in
CI but not locally" problem disappears, because there is nothing in CI
that wasn't on your laptop — there are only different *profiles* that
select different *named* configurations of the same shape.

A word the reader will meet repeatedly: a **Pod** is the runtime
realisation of a Landscape and a Topology — the actual collection of
interoperating resources a job runs against.  Pods are the unit of
horizontal CI parallelism (Volume 2 chapter 3, Volume 6 chapter 9):
within one Pod, a job runs sequentially because the Pod is a shared
environment; across independent Pods, jobs run in parallel because
Pods are independent.  When the rest of this volume talks about
"more horizontal scale," the answer is *more Pods* — provisioned
through more Specifications — not more parallel tests inside one
Pod.

The portfolio of `codecraftai-*` packages implements this discipline.
The configuration loader supports five of the six categories today
(the sixth is in flight).  The orchestration layer is `codecraftai-taskplus`.
The job layer below it is `codecraftai-testbase` and its distributed
extension `codecraftai-testplus`.  The credentials story is
`codecraftai-credentials`.  The book draws its examples from these
packages because the discipline is implemented there, not because the
discipline depends on them — the same patterns translate to any
toolkit a reader might be using.

A note on voice.  The series is narrated by CodeCraftX, the persona
that opens every `codecraftai-*` repository's `AGENTS.md` and that
embodies the platform's rules: debuggability paramount, explicit over
implicit, errors should never pass silently, properties as simple data
accessors, every function ending in an explicit `return`.  Those rules
are introduced in Volume 0 of this series; they are referenced here.
A reader who has not yet read Volume 0 will still follow the argument
of this volume, but the rationale for some of the apparently
unconventional choices — the rejection of decorator-based retry, for
example, or the insistence that `SemanticError` always bubble — lives
there.

A note on order.  The volume is structured into six parts.  Parts I
through IV present the discipline before the implementation.  Part V
is where the implementation lands — the `taskplus` task model,
workflow YAML, and FlowEngine.  Part VI wires everything to a real CI
vendor and walks through a worked port of an unmaintainable
fifty-step YAML to a thirty-line workflow plus five named
configurations.  A reader who is impatient and wants to see the code
first can read Part V before Parts II–IV; the discipline-first
ordering is the recommendation, not a requirement.

Finally: this is not the only viable approach to integration
automation.  Chapter 27 is honest about where the discipline does not
fit — heavy parallel fan-out, dynamic workflow generation from
streaming events, sub-second event-driven flows.  The book is opinionated
because the alternative is bland; the chapter where the opinions are
qualified comes at the end, where it should.
