# PR: Settle the RequiredResource API shape for scorer-declared permissions

**Jira:** [RHOAIENG-56609](https://redhat.atlassian.net/browse/RHOAIENG-56609)
**Branch:** `jobs-execution-rfc2`
**RFC:** [0002-job-executor-plugins](https://github.com/mlflow/rfcs/blob/main/rfcs/0002-job-executor-plugins/0002-job-executor-plugins.md)
**RFC discussion:** [PR #2 comment](https://github.com/mlflow/rfcs/pull/2#discussion_r3012870624)

---

## What this PR does

Introduces the `RequiredResource` API shape that lets scorers (and eventually other job types) declare what MLflow-managed resources they need at runtime (e.g., a custom scorer running in a remote K8s pod needs access to a specific gateway endpoint). This is necessary for the RFC's scoped token minting — the framework reads these declarations to grant only the permissions each job needs.

## API usage

### Declaring resources on a custom scorer

```python
from mlflow.genai.scorers import scorer, RequiredResource


@scorer(
    required_resources=(
        RequiredResource(type="gateway_endpoint", name="my-llm-endpoint"),
        RequiredResource(type="prompt", name="prompts:/tool-grounded/1"),
    ),
)
def tool_grounded(outputs, trace):
    client = OpenAI(base_url="<gateway-url>/gateway/my-llm-endpoint")
    prompt = mlflow.genai.load_prompt("prompts:/tool-grounded/1")
    text = prompt.format(output=outputs, tools=trace.search_traces(span_type="TOOL"))
    response = client.responses.create(text)
    return parse_result(response)
```

When the scorer is registered, `required_resources` is serialized into the `serialized_scorer` TEXT column of the `scorer_versions` table:

```json
{
  "name": "tool_grounded",
  "call_source": "...",
  "required_resources": [
    { "type": "gateway_endpoint", "name": "my-llm-endpoint" },
    { "type": "prompt", "name": "prompts:/tool-grounded/1" }
  ]
}
```

The job framework can read `required_resources` with a simple `json.loads()` without going through `Scorer.model_validate()`, which would run `exec()` on custom scorer code. The RFC calls this [shallow parsing](https://github.com/mlflow/rfcs/blob/main/rfcs/0002-job-executor-plugins/0002-job-executor-plugins.md#permission-extraction).

Built-in and `make_judge` scorers don't need `required_resources` — the framework can [infer their gateway endpoint](https://github.com/mlflow/rfcs/blob/main/rfcs/0002-job-executor-plugins/0002-job-executor-plugins.md#permission-extraction) from the structured `model` field (e.g., `model="gateway:/my-endpoint"`). `required_resources` is for custom `@scorer` code where resource usage can't be statically inspected.

**A note on `name` for prompts:** The `name` field accepts either a prompt name (`"tool-grounded"`) or a prompt URI with a version (`"prompts:/tool-grounded/1"`). Since MLflow RBAC grants prompt permissions per name, not per version, the downstream permission extraction will grant `READ` on the prompt name regardless. At runtime, the scorer code can load any version of that prompt.

## Reasoning for above API shape

Two possible implementation paths were discussed in the RFC review:

**B-Step62's position** ([comment](https://github.com/mlflow/rfcs/pull/2#discussion_r3012870624)): Reuse the existing `mlflow.models.resources.Resource` ABC to avoid having "two permission resource objects in the platform."

**mprahl's position** ([reply](https://github.com/mlflow/rfcs/pull/2#discussion_r3012870624)): Create a new `RequiredResource` dataclass — the shapes are different enough that reuse isn't viable.

After analyzing both, we went with a new type. The existing `Resource` class ([`mlflow/models/resources.py`](https://mlflow.org/docs/latest/api_reference/_modules/mlflow/models/resources.html)) declares infrastructure dependencies that a registered model's program needs when deployed on Databricks via [`log_model(resources=[...])`](https://learn.microsoft.com/en-us/azure/databricks/generative-ai/agent-framework/log-agent). It serves a fundamentally different purpose:

|                              | `Resource` (model serving)                                                                                                                                                                                                                   | `RequiredResource` (job execution)                                                                                                                                                                            |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Purpose**                  | Declare platform-specific infrastructure a model needs for [deployment](https://learn.microsoft.com/en-us/azure/databricks/generative-ai/agent-framework/agent-authentication-model-serving#log-resources-automatic) (Databricks-only today) | Declare MLflow-managed resources a job needs to execute, independent of where it runs (local, K8s, Docker, or any executor plugin)                                                                            |
| **Who reads it**             | [Databricks Model Serving](https://www.databricks.com/product/model-serving) reads the `resources` list at deploy time and provisions credentials for the declared resources                                                                 | MLflow's job framework reads a registered scorer's `required_resources` at job submission time and mints a short-lived token granting access to only the declared resources                                   |
| **Who checks access**        | Databricks platform — outside MLflow's control                                                                                                                                                                                               | MLflow itself — [API server middleware](https://github.com/mlflow/rfcs/blob/main/rfcs/0002-job-executor-plugins/0002-job-executor-plugins.md#api-server-enforcement) validates the job token on every request |
| **Platform binding**         | Tied to Databricks — the `Resource` ABC requires a `target_uri` property (always `"databricks"` in practice)                                                                                                                                 | Platform-independent — no `target_uri` concept needed                                                                                                                                                         |
| **Supported resource types** | 9 Databricks infrastructure types (UC_CONNECTION, SERVING_ENDPOINT, VECTOR_SEARCH_INDEX, ...)                                                                                                                                                | MLflow RBAC resource types (gateway_endpoint, prompt) — extensible to more types as needed                                                                                                                    |

Forcing gateway endpoints and prompts into the `Resource` hierarchy would mean:

- Confusing [`log_model(resources=[...])`](https://mlflow.org/docs/latest/api_reference/_modules/mlflow/models/resources.html) users — scorer resources would be valid `Resource` subclasses, so type checkers would allow passing them to `log_model` where they have no meaning
- Mixing Databricks infrastructure types designed for a registered model's program served on the Databricks platform with MLflow-internal resource types in the same `ResourceType` enum
- Requiring a `target_uri` value for MLflow-internal resources when the concept doesn't apply

### How we build on the RFC's `RequiredResource` proposal

We adopt the RFC's `RequiredResource` dataclass approach, which already uses `Literal["gateway_endpoint", "prompt"]`. We add the following refinements:

1. **Runtime validation** — unknown types raise `ValueError` at scorer registration time, catching mistakes even in code paths that type checkers don't reach.
2. **`name` instead of `identifier`** — consistent with how the rest of the MLflow codebase refers to resources (e.g. `DatabricksServingEndpoint`).
3. **`type` values align with existing MLflow RBAC constants** — `type="gateway_endpoint"` matches `RESOURCE_TYPE_GATEWAY_ENDPOINT` in `mlflow/server/auth/permissions.py`. Same for `"prompt"` → `RESOURCE_TYPE_PROMPT`. We can't import these constants directly due to circular dependencies (`mlflow.entities` → `mlflow.server` → `mlflow.genai` → `mlflow.entities`), so the strings are hardcoded with a comment pointing to the authoritative source and a test (`test_resource_types_match_rbac`) that enforces the match.
4. **Placed in `mlflow/entities/`** — `mlflow/entities/_required_resource.py` is available to any subsystem (scorers, prompt optimization, future job types). Re-exported from `mlflow.genai.scorers` for the user-facing API.

## How downstream tickets will use this

### RHOAIENG-56613 — Remote-job permission extraction and scoped token enforcement

This is the primary consumer. The [permission extraction](https://github.com/mlflow/rfcs/blob/main/rfcs/0002-job-executor-plugins/0002-job-executor-plugins.md#permission-extraction) logic will:

1. Read `required_resources` from the serialized scorer JSON
2. Union declared resources with inferred resources (e.g., from the `model` field on built-in or `make_judge` scorers in the same job — a single job can score new traces across an experiment that has multiple scorers of different types)
3. Map each resource to an RBAC permission:
   - `type="gateway_endpoint"` → `USE` permission
   - `type="prompt"` → `READ` permission
4. Build `scoped_permissions` dict on the job row
5. Mint a token scoped to exactly those permissions

## What this PR does NOT do

- **No permission enforcement** — that's RHOAIENG-56613
- **No token minting** — that's RHOAIENG-56613
- **No routing changes** — that's RHOAIENG-56598
- **No DB schema or protobuf changes** — `required_resources` travels inside the existing `serialized_scorer` JSON blob
- **No `exec()` changes** — custom scorers are still blocked in OSS; this PR only adds the resource declaration mechanism

## Test results

- 17 new tests: all pass
- 66 existing scorer tests: all pass (0 regressions)

## References

### RFC and discussion

- [RFC 0002 — Job Executor Plugins](https://github.com/mlflow/rfcs/blob/main/rfcs/0002-job-executor-plugins/0002-job-executor-plugins.md) — full RFC text
- [RFC PR #2 — review discussion on RequiredResource vs Resource](https://github.com/mlflow/rfcs/pull/2#discussion_r3012870624) — the B-Step62 / mprahl debate this PR resolves
- [RFC: Permission extraction](https://github.com/mlflow/rfcs/blob/main/rfcs/0002-job-executor-plugins/0002-job-executor-plugins.md#permission-extraction) — how `required_resources` feeds into scoped token minting
- [RFC: API server enforcement](https://github.com/mlflow/rfcs/blob/main/rfcs/0002-job-executor-plugins/0002-job-executor-plugins.md#api-server-enforcement) — endpoint-to-permission mapping table

### Existing `Resource` primitive (Databricks model serving)

- [`mlflow.models.resources` API reference](https://mlflow.org/docs/latest/api_reference/_modules/mlflow/models/resources.html) — the existing `Resource` ABC and Databricks subclasses
- [Databricks: Automatic authentication passthrough](https://learn.microsoft.com/en-us/azure/databricks/generative-ai/agent-framework/agent-authentication-model-serving#log-resources-automatic) — how Databricks consumes `Resource` declarations
- [Databricks: Log and register AI agents](https://learn.microsoft.com/en-us/azure/databricks/generative-ai/agent-framework/log-agent) — `log_model(resources=[...])` usage examples

### MLflow RBAC

- [MLflow RBAC documentation](https://mlflow.org/docs/latest/self-hosting/security/role-based-access-control/) — resource types (`gateway_endpoint`, `prompt`, `experiment`, etc.) and permission levels (`READ`, `USE`, `EDIT`, `MANAGE`)

### MLflow scorers

- [Custom scorers](https://mlflow.org/docs/latest/genai/eval-monitor/scorers/custom/) — `@scorer` decorator documentation
- [LLM judge workflow](https://mlflow.org/docs/latest/genai/eval-monitor/scorers/llm-judge/workflow) — `make_judge` and built-in scorers
- [Automatic evaluations (online scoring)](https://mlflow.org/docs/latest/genai/eval-monitor/automatic-evaluations/) — server-side periodic scoring

### Jira

- [RHOAIENG-56609](https://redhat.atlassian.net/browse/RHOAIENG-56609) — this ticket (settle RequiredResource API shape)
- [RHOAIENG-56594](https://redhat.atlassian.net/browse/RHOAIENG-56594) — parent epic (job executor plugin framework)
- [RHOAIENG-56613](https://redhat.atlassian.net/browse/RHOAIENG-56613) — downstream: permission extraction and scoped token enforcement
- [RHOAIENG-56598](https://redhat.atlassian.net/browse/RHOAIENG-56598) — downstream: per-job executor routing
