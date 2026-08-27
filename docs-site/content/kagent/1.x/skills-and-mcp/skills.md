---
title: Skills
description: Package instructions and supporting files as skills, and attach them to an AgentTemplate.
weight: 20
author: kagent.dev
---

A **skill** packages a piece of know-how that an agent can pick up: a set of instructions, together with whatever scripts or reference files those instructions depend on. An {{< gloss "AgentTemplate" >}}AgentTemplate{{< /gloss >}} attaches skills by naming where each one comes from, and kagent fetches them and places them where the agent runtime can find them.

## About

A skill is a directory whose root holds a `SKILL.md` file. That file carries front matter naming the skill and describing what it is for, followed by the instructions themselves.

The description is what makes a skill usable. When an agent starts, the runtime reads the front matter of every skill that is attached to it, and offers those skills to the model as tools that it can call. The model chooses a skill from its description, in the same way that it chooses any other tool. This means that a description that states plainly when to use the skill matters more than the length of the instructions behind it.

A skill can be published on its own, or bundled with other skills in a **plugin package**. A plugin package follows the Agent Plugins 1.0.0 format. Its `plugin.json` manifest names the package, and its skills live in a `skills` directory, one subdirectory per skill. A package may also declare Model Context Protocol (MCP) servers, which kagent adds to the agent's tools alongside the skills that you enable from that package.

## Attach skills to an AgentTemplate

An AgentTemplate attaches skills in two ways, and it can use both at once. Use `spec.skills` for a standalone skill, and `spec.plugins` for a plugin package.

```yaml
apiVersion: kagent.dev/v1alpha3
kind: AgentTemplate
metadata:
  name: incident-responder
  namespace: kagent
spec:
  modelConfig:
    name: default-model-config
  systemPrompt: You help engineers work through production incidents.
  skills:
    - name: incident-triage
      source:
        oci: registry.example.com/skills/incident-triage@sha256:<digest>
  plugins:
    - source:
        git:
          url: https://github.com/example/agent-plugins
          commit: <full-commit-id>
        path: bundles/observability
      skills:
        - log-search
        - runbook-lookup
```

| Field | Description |
| ----- | ----------- |
| `skills[].name` | The name that the skill is mounted under, and the name that the model sees. |
| `skills[].source` | Where to fetch this one skill from. The source root must hold a `SKILL.md` file. |
| `plugins[].source` | Where to fetch the plugin package from. The source root must hold a `plugin.json` manifest. |
| `plugins[].skills` | The names of the skills inside that package to enable. Omit or leave empty to enable none. |
| `source.path` | Selects a directory inside the artifact, when the content is not at its root. The path must be relative, and it cannot climb out of the artifact with `..` segments. |

### Plugin allowlist

Attaching a plugin package does not enable any of the skills that it carries. Only the names listed in `plugins[].skills` are enabled.

> [!IMPORTANT]
> An empty skills list enables nothing. Adding a plugin package and omitting its `skills` list gives the agent no skills from that package, which is the safe default rather than an error. Listing skills explicitly also means that a package gaining new skills in a later version does not silently grant them to your agent.

## Every source is immutable

A skill changes what an agent does, so kagent only accepts artifact references that cannot shift underneath a running agent. Each source names exactly one of three kinds of artifact, and each one has to be pinned.

- **`oci`**: An image reference pinned to a digest, in the form `<repository>@sha256:<digest>`. A tag alone is rejected, because a tag can be moved to different content later.
- **`git`**: A repository URL together with a full commit identifier. An abbreviated commit, a branch, or a tag is rejected.
- **`bucket.s3`**: An endpoint, bucket, and key, together with the `versionId` of that exact object version. A region is included where the service requires one for request signing.

Pinning has a practical consequence worth planning for. Publishing a new version of a skill means updating the AgentTemplate to name the new digest, commit, or object version, which compiles a new revision. Agents that are already running keep using the skill content that they started with.

## Naming rules

Skill names are checked before anything is fetched, and two rules apply across every skill on an AgentTemplate.

- A name must be a single path component. A name containing a slash, or a name of `.` or `..`, is rejected.
- Names must be unique across the whole AgentTemplate. Because standalone skills and plugin skills are mounted into the same place, a standalone skill cannot reuse the name of an enabled plugin skill, and two plugin packages cannot both contribute the same name.

Plugin package names must also be unique. Two entries in `plugins` whose manifests declare the same name are rejected. Unlike skill names, kagent checks this as each package is fetched, because the name comes from the package's `plugin.json` manifest rather than from your AgentTemplate.

## Next steps

{{< cards >}}
  {{< card link=`{{< link path="examples/skills" >}}` title="Add skills to agents" subtitle="Walk through attaching a skill from an OCI artifact, end to end." >}}
  {{< card link=`{{< link path="skills-and-mcp/about-tools" >}}` title="About tools" subtitle="See how MCP tools and agent tools attach to an AgentTemplate." >}}
  {{< card link=`{{< link path="get-started/your-first-agent" >}}` title="Your first agent" subtitle="Apply a Harness and AgentTemplate, and talk to the AgentInstance they produce." >}}
{{< /cards >}}
