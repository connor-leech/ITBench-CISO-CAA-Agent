# CISO Agent

CISO (Chief Information Security Officer) agents automate compliance assessments. They generate policies (e.g., Kyverno, OPA Rego) from natural language prompts, automate evidence collection, integrate with GitOps workflows, and deploy policies to validate compliance. These capabilities streamline compliance processes and enhance operational efficiency. The agents are built using the open-source frameworks [CrewAI](https://github.com/crewAIInc/crewAI) and [LangGraph](https://github.com/langchain-ai/langgraph).

## Prerequisites

- Access to an OpenAI-compatible LLM service  
  (tested with IBM watsonx.ai, OpenAI, and Azure OpenAI)
- A deployed [ITBench scenario](https://github.com/IBM/ITBench-Scenarios)  
  (requires 1 Kubernetes cluster and/or 1 RHEL host)
- `python` (tested with 3.11)
- `docker` or `podman` (tested with Docker 26.1.0, Podman 5.1.2)

## Getting started

### 1. Deploy a scenario

Follow the [CISO scenario guide](https://github.com/IBM/ITBench-Scenarios/tree/main/ciso#readme):

```bash
make deploy_bundle
make inject_fault
```

Then get the scenario goal for the agent:

```bash
make get 2>/dev/null | jq -r .goal_template
```

Example goal output:

```text
I would like to check if the following condition is satisfied, given a Kubernetes cluster with `kubeconfig.yaml`:
Minimize the admission of containers wishing to share the host network namespace.

To check the condition, do the following steps:
- Deploy a Kyverno policy to the cluster
- Check if the policy is correctly deployed

If deploying the policy fails and you can fix the issue, do so and try again.

The cluster's kubeconfig is at `{{ kubeconfig }}`.
```

Save this goal text for later.

### 2. Clone this repository

```bash
git clone https://github.com/IBM/it-bench-ciso-caa-agent.git
cd ciso-agent
```

### 3. Build the agent container

```bash
docker build -f Dockerfile -t ciso-agent:latest .
# or, if using podman:
podman build -f Dockerfile -t ciso-agent:latest .
```

### 4. Configure `.env` with LLM credentials

Create a `.env` file using your LLM service credentials:

- **IBM watsonx.ai**
```env
LLM_BASE_URL=<ENDPOINT_URL>
LLM_API_KEY=<YOUR_API_KEY>
LLM_MODEL_NAME=<MODEL_NAME>
WATSONX_PROJECT_ID=<YOUR_PROJECT_ID>
```

- **OpenAI**
```env
LLM_API_KEY=<YOUR_API_KEY>
LLM_MODEL_NAME=<MODEL_NAME>
```

- **Azure OpenAI**
```env
LLM_BASE_URL=<ENDPOINT_URL>
LLM_API_KEY=<YOUR_API_KEY>
LLM_MODEL_NAME=<MODEL_NAME>
LLM_PARAMS='{"api-version": "<API_VERSION>"}'
```

### 5. Run the agent

Prepare:

- `<WORKDIR>`: Contains scenario files (`kubeconfig.yaml`, etc.)
- `<DOT_ENV>`: Path to your `.env`
- `<GOAL_TEXT>`: Scenario goal (replace `{{ kubeconfig }}` with `/tmp/agent/kubeconfig.yaml`)

Then run:

```bash
docker run --rm -ti \
  -v <WORKDIR>:/tmp/agent \
  -v <DOT_ENV>:/etc/ciso-agent/.env \
  ciso-agent:latest \
  python src/ciso_agent/main.py \
  --goal '<GOAL_TEXT>' \
  --auto-approve
```

Replace `docker` with `podman` if needed.

The agent logs will appear similar to:

![Start](img/agent_log_example_beginning.png)  
(Agent starting)

![Result](img/agent_log_example_result.png)  
(Agent finished)

The run typically takes under 5 minutes.

### 6. Evaluation

After the agent finishes, evaluate results using the scenario’s instructions [here](https://github.com/IBM/ITBench-Scenarios/tree/main/ciso#readme).
