# Managed and shared core tools

Core resource and execution tools only. Read the live schema before calling; required and optional top-level arguments remain unchanged. Nested configuration guidance is in [Managed workflows](workflows.md) and [Forward workflows](forward-workflows.md).

R = read-only; W = mutation; D = delete/detach (inspect its scope). Managed lists return top-level `items`; Forward lists normally return `item.data`. Forward detail tools use their declared resource-specific IDs: `template_id`, `session_id`, `schedule_id`, or `environment_id`; check each live schema.

## Models and Agents

| Mode | Tool | Use | Arguments |
| --- | --- | --- | --- |
| R | `list_models` | List the models available to the caller for Qoder Cloud Agents, including reasoning efforts and context window options. | **Required:** none; **optional:** none |
| W | `create_agent` | Create a Qoder Cloud Agents agent with its model, system prompt, built-in or Browser Use tools, MCP servers and existing Skills. | **Required:** `name`, `model`; **optional:** `description`, `mcp_servers`, `metadata`, `multiagent`, `skills`, `system`, `tools` |
| R | `get_agent` | Retrieve one Qoder Cloud Agents agent, optionally at a specific version. | **Required:** `agent_id`; **optional:** `version` |
| R | `list_agents` | List Qoder Cloud Agents agents. | **Required:** none; **optional:** `created_at_gte`, `created_at_lte`, `include_archived`, `limit`, `page` |
| W | `update_agent` | Update a Qoder Cloud Agents agent using its current version. Tool, MCP-server and Skill arrays replace their stored arrays. | **Required:** `agent_id`, `version`; **optional:** `description`, `mcp_servers`, `metadata`, `model`, `name`, `skills`, `system`, `tools` |
| W | `archive_agent` | Archive a Qoder Cloud Agents agent so it stops appearing in default listings. | **Required:** `agent_id`; **optional:** none |

## Environments

| Mode | Tool | Use | Arguments |
| --- | --- | --- | --- |
| W | `create_environment` | Create a Qoder Cloud Agents environment describing the packages and setup script that agent sessions run inside. | **Required:** `name`; **optional:** `config`, `description`, `metadata` |
| R | `get_environment` | Retrieve one Qoder Cloud Agents environment. | **Required:** `environment_id`; **optional:** none |
| R | `list_environments` | List Qoder Cloud Agents environments. | **Required:** none; **optional:** `created_at_gte`, `created_at_lte`, `include_archived`, `limit`, `page` |
| W | `update_environment` | Update a Qoder Cloud Agents environment. Only the supplied fields change, and config replaces the stored configuration wholesale. | **Required:** `environment_id`; **optional:** `config`, `description`, `metadata`, `name` |
| D | `delete_environment` | Permanently delete a Qoder Cloud Agents environment. | **Required:** `environment_id`; **optional:** none |
| W | `archive_environment` | Archive a Qoder Cloud Agents environment so it stops appearing in default listings. | **Required:** `environment_id`; **optional:** none |

## Skills

| Mode | Tool | Use | Arguments |
| --- | --- | --- | --- |
| R | `list_skills` | List Qoder Cloud Agents Skills available to the caller for discovery and Agent binding. | **Required:** none; **optional:** `display_title`, `limit`, `page`, `source` |
| R | `get_skill` | Retrieve one Qoder Cloud Agents Skill shell, including its latest version pointer. | **Required:** `skill_id`; **optional:** none |

## Files

| Mode | Tool | Use | Arguments |
| --- | --- | --- | --- |
| R | `list_files` | List Qoder Cloud Agents files, such as artifacts delivered by sessions. | **Required:** none; **optional:** `after_id`, `before_id`, `limit`, `name`, `page`, `scope_id` |
| R | `get_file` | Retrieve the metadata of one Qoder Cloud Agents file. | **Required:** `file_id`; **optional:** none |

## Existing Vaults

| Mode | Tool | Use | Arguments |
| --- | --- | --- | --- |
| R | `get_vault` | Retrieve one Qoder Cloud Agents vault. | **Required:** `vault_id`; **optional:** none |
| R | `list_vaults` | List Qoder Cloud Agents vaults. | **Required:** none; **optional:** `after_id`, `before_id`, `include_archived`, `limit`, `name`, `page` |

## Sessions

| Mode | Tool | Use | Arguments |
| --- | --- | --- | --- |
| W | `create_session` | Create a Qoder Cloud Agents Session with an Agent, Environment and optional resource bindings. Send the first message separately with send_session_events; no events/initial_events input is exposed here. | **Required:** `agent`, `environment_id`; **optional:** `environment_variables`, `metadata`, `resources`, `title`, `vault_ids` |
| R | `get_session` | Retrieve one Qoder Cloud Agents session, including its status, resources and usage. | **Required:** `session_id`; **optional:** none |
| R | `list_sessions` | List compact Qoder Cloud Agents session summaries. Use get_session for the complete Agent snapshot, resources and outcome details. | **Required:** none; **optional:** `agent_id`, `agent_version`, `created_at_gt`, `created_at_gte`, `created_at_lt`, `created_at_lte`, `deployment_id`, `include_archived`, `limit`, `memory_store_id`, `order`, `page`, `statuses` |
| W | `update_session` | Update mutable Qoder Cloud Agents Session fields or replace the Session Agent snapshot's tools and MCP servers. Browser Use is accepted in agent.tools. | **Required:** `session_id`; **optional:** `agent`, `metadata`, `title` |
| D | `delete_session` | Permanently delete a Qoder Cloud Agents session and its recorded events. | **Required:** `session_id`; **optional:** none |
| W | `archive_session` | Archive a Qoder Cloud Agents session so it stops appearing in default listings. | **Required:** `session_id`; **optional:** none |

## Session Resources and Runtime

| Mode | Tool | Use | Arguments |
| --- | --- | --- | --- |
| W | `add_session_resource` | Mount an existing Qoder Cloud Agents file into a running session. | **Required:** `session_id`, `type`, `file_id`; **optional:** `mount_path` |
| R | `get_session_resource` | Retrieve one resource mounted into a Qoder Cloud Agents session. | **Required:** `session_id`, `resource_id`; **optional:** none |
| R | `list_session_resources` | List the resources mounted into a Qoder Cloud Agents session. | **Required:** `session_id`; **optional:** `after_id`, `before_id`, `limit`, `page` |
| D | `delete_session_resource` | Unmount a resource from a Qoder Cloud Agents session. | **Required:** `session_id`, `resource_id`; **optional:** none |
| W | `send_session_events` | Send events to a running Qoder Cloud Agents session: user messages, interrupts, tool confirmations, tool results and outcome definitions. | **Required:** `session_id`, `events`; **optional:** none |
| R | `list_session_events` | List complete Qoder Cloud Agents session events, including type-specific payloads that may contain large tool or fetched-page results. Filter by types and use small limits; use list_session_event_summaries to discover event types without payloads. | **Required:** `session_id`; **optional:** `after_id`, `before_id`, `created_at_gt`, `created_at_gte`, `created_at_lt`, `created_at_lte`, `limit`, `order`, `page`, `types` |
| R | `list_session_event_summaries` | List lightweight Qoder Cloud Agents session event identities and timestamps without type-specific payloads. Use this first to discover event types or poll progress, then call list_session_events with a narrow types filter when payload content is needed. | **Required:** `session_id`; **optional:** `after_id`, `before_id`, `created_at_gt`, `created_at_gte`, `created_at_lt`, `created_at_lte`, `limit`, `order`, `page`, `types` |
| W | `cancel_session` | Cancel a Qoder Cloud Agents session, stopping the agent and releasing its sandbox. | **Required:** `session_id`; **optional:** none |

## Deployments and Runs

| Mode | Tool | Use | Arguments |
| --- | --- | --- | --- |
| W | `create_deployment` | Create a Qoder Cloud Agents deployment that runs an agent on a cron schedule or on demand. | **Required:** `name`, `agent`, `environment_id`, `initial_events`; **optional:** `description`, `environment_variables`, `metadata`, `resources`, `schedule`, `vault_ids` |
| R | `get_deployment` | Retrieve one Qoder Cloud Agents deployment, including its schedule and upcoming run times. | **Required:** `deployment_id`; **optional:** none |
| R | `list_deployments` | List Qoder Cloud Agents deployments. | **Required:** none; **optional:** `after_id`, `agent_id`, `before_id`, `created_at_gte`, `created_at_lte`, `include_archived`, `limit`, `page`, `status` |
| W | `update_deployment` | Update a Qoder Cloud Agents deployment. Only the supplied fields change. | **Required:** `deployment_id`; **optional:** `agent`, `description`, `environment_id`, `environment_variables`, `initial_events`, `metadata`, `name`, `resources`, `schedule`, `vault_ids` |
| W | `archive_deployment` | Archive a Qoder Cloud Agents deployment so it stops running and stops appearing in default listings. | **Required:** `deployment_id`; **optional:** none |
| W | `pause_deployment` | Pause a Qoder Cloud Agents deployment so its schedule stops firing. Pass the deployment ID as deployment_id; jobId is not accepted. | **Required:** `deployment_id`; **optional:** none |
| W | `unpause_deployment` | Resume a paused Qoder Cloud Agents deployment so its schedule fires again. Pass the deployment ID as deployment_id; jobId is not accepted. | **Required:** `deployment_id`; **optional:** none |
| W | `run_deployment` | Trigger one Qoder Cloud Agents deployment run immediately, outside its schedule. Pass the deployment ID as deployment_id; jobId is not accepted. | **Required:** `deployment_id`; **optional:** none |
| R | `list_deployment_runs` | List Qoder Cloud Agents deployment runs across every deployment. | **Required:** none; **optional:** `after_id`, `before_id`, `created_at_gt`, `created_at_gte`, `created_at_lt`, `created_at_lte`, `deployment_id`, `has_error`, `limit`, `page`, `trigger_type` |
| R | `get_deployment_run` | Retrieve one globally addressable Qoder Cloud Agents deployment run. Pass only run_id; do not also send deployment_id. | **Required:** `run_id`; **optional:** none |
