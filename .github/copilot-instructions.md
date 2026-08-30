# Copilot Instructions

## Workspace Overview
This repository is an Ansible monorepo to create a test environment for the
MyAIOps project. It's roles include creating a K8s cluster running on EC2
instances and installing fundamental K8s components into the cluster.

Project structure:

```
.
├── ansible.cfg                    # Ansible runtime config (inventory path, roles_path, etc.)
├── inventory.yaml                 # Hardcoded inventory with only the local control node
├── playbooks/
│   ├── deploy.yml              # Deploy or rerun insfrastructure provisioning and cluster setup
│   ├── destroy.yml                 # Destroy the cluster and all EC2 instances
├── roles/
│   ├── ec2_provision/             # Launches/tags EC2 instances via AWS API
│       ├── tasks
│       ├── templates
│       ├── vars
│       ├── defaults
│       ├── files
│   ├── container_runtime/         # Installs and configures containerd  
│   ├── kubeadm_init/              # Initializes the control-plane node
│   ├── kubeadm_join/              # Joins worker nodes to the cluster
│   ├── cni/                       # Installs the cluster networking plugin
│   └── k8s_components/            # Installs fundamental in-cluster components
├── requirements.yml               # Galaxy collection/role dependencies
├── requirements.txt               # Python dependencies
└── README.md                      # Setup, usage, and Jenkins job integration notes
```

## General Ansible Coding Rules
- Every role should have these files if needed: `tasks/main.yml`, `defaults/main.yml`, `handlers/main.yml`. Use `vars/` only for constants that should never be overridden.
- Every role should have a tag in playbook deploy.yaml to allow rerunning only that role (e.g., `--tags kubeadm_init`).
- Always define variables in `defaults/main.yml` with sane defaults — never hardcode values (instance type, region, AMI, k8s version, pod CIDR) directly in tasks.
- Use `ansible.cfg` to pin: `inventory`, `roles_path`, `host_key_checking = False` (justify in comments), `retry_files_enabled = False`, `stdout_callback = yaml`.
- Always pin versions in Ansible collections in `requirements.yml` and python dependencies in `requirements.txt`. Never use `latest` for cluster-critical packages, but prefer to pin the latest stable release version.
- All tasks must have a `name:` that describes intent in plain English.
- Use FQCN (fully qualified collection names) for every module, e.g. `ansible.builtin.copy`, `amazon.aws.ec2_instance`, `community.kubernetes.k8s`.
- Use `become: true` only at the task or block level where privilege escalation is actually needed — never blanket at play level unless every task requires it.
- Idempotency is mandatory: every task must be safely re-runnable. Prefer built-in modules over `command`/`shell`. When `shell`/`command` is unavoidable, add `changed_when` / `creates` / `removes` to make it idempotent, and comment why.
- Use `handlers` for service restarts (kubelet, containerd) — never restart services inline mid-task.
- The playbook always executes from the local control node. AWS API calls (via `amazon.aws.ec2_instance`) run locally to provision instances; all subsequent configuration tasks then connect *out* to those instances over SSH — the control node itself is never a managed/target node.
- The inventory is hardcoded to include only the local control node.
- Use `block/rescue/always` for operations that can fail (e.g., kubeadm init) so failures produce clear diagnostics instead of a silent halt.
- Ansible variable naming convention:
  + Public role variables:
    + Concept: Variables that are exposed as public input or output variables of a role, or those defined in `defaults/main.yml` and `vars/main.yml`.
    + Naming format: `<role-name>__<variable-name>`
  + Private role variables:
    + Concept: Variables that are registered or set via `set_fact` in tasks at runtime and are intended to be used privately within the role itself.
    + Naming format: `_<role-name>__<variable-name>`
  + Loop variables:
    + Concept: Variables that are used as loop iterators and are not meant to be used outside of the loop.
    + Naming format: `__<role-name>__<variable-name>`
- Do not use emojis or icon characters in comments, docstrings, or commit messages.
- Keep comments short and only add them when the code is not self-explanatory; explain why, not what.
- As for file configuration, prefer to use templates over cloning a default config file and editing it in place. Use ansible.builtin.template to render config files from Jinja2 templates, and use ansible.builtin.copy only for static files that do not require any variable substitution.
- Use full path when defining ansible module arguments (e.g., `ansible.builtin.copy`, `ansible.builtin.template`, `ansible.builtin.command`, etc.) to avoid ambiguity and ensure compatibility across different Ansible versions and collections.

## Secrets & Security
- Never hardcode AWS credentials, tokens, or the kubeadm join token in playbooks.
- Sensitive variables (SSH private key paths, registry credentials, tokens) are injected at runtime via Jenkins (as environment variables or `--extra-vars`).
- Generate the kubeadm join token dynamically and pass it via `set_fact` / `add_host` from control-plane output to worker plays — do not hardcode.
- Security groups: expose only required ports (22, 6443, 2379-2380,10250-10252, 30000-32767 NodePort range if needed) — never `0.0.0.0/0` on all ports.

## Testing Requirements
- All playbooks/roles must pass `ansible-lint` (production profile) and `yamllint` before being considered complete:
  + Ansible-lint checking command: `ansible-lint .`
  + Yamllint checking command: `yamllint .`
- DO NOT write ansible unit tests (or any other types of test files) and run ansible playbooks except being asked in prompt.
- Use venv python path `~/.venv/bin/activate` to run python code if required. Do not use the system python path.

## LLM token usage rules
- Save and use input and output LLM token efficiently as much as possible.