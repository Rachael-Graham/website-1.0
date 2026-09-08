---
title: Add a skill to an agent
description: Package instructions and a script as a skill, publish it as an OCI image, and attach it to an AgentTemplate.
weight: 40
author: kagent.dev
---

A {{< gloss "Skill" >}}skill{{< /gloss >}} packages know-how that an agent picks up at run time: a `SKILL.md` file of instructions, together with the scripts and reference files those instructions depend on. This example builds a skill that turns raw commit subjects into release notes, publishes it as an OCI image, and attaches it to an {{< gloss "AgentTemplate" >}}AgentTemplate{{< /gloss >}}.

For the fields that attach a skill and the rules that govern their names, see [Skills]({{< link path="skills-and-mcp/skills" >}}). For the format of a multi-skill package, see [Plugins]({{< link path="skills-and-mcp/plugins" >}}).

## About skills at run time

kagent does not fetch a skill when you apply an AgentTemplate. The compiled revision records where each skill comes from, and the {{< gloss "Actor" >}}Actor{{< /gloss >}} fetches it when the agent starts. Every artifact is unpacked under `/plugins`, whether it holds one skill or a package of them, and each enabled skill is then copied to `/skills/<skill-name>`. The agent reads skills only from `/skills`.

As a consquence, the following potential gotchas can occur. 

* **A wrong source still compiles.** kagent validates skill names before it accepts an AgentTemplate, but it never checks that the artifact exists or that it holds a `SKILL.md` file. A bad digest produces a revision that reports `Ready`, and the agent then fails to start.
* **Scripts run in the runtime image.** A skill's scripts get whatever the Harness image provides. The kagent runtime image is Alpine Linux with `bash`, `git`, and the standard Alpine utilities, and it does **not** include Python.

### Skill tools

On a `kagent` Harness, attaching a skill adds seven tools to the agent, whether or not the skill ships a script. The first three read skills, and the rest let the agent act on their files. The `claude` and `codex` Harness runtimes take the same skills and expose them through their own coding agent's tools instead.

| Tool | What it does |
| ---- | ------------ |
| `list_skills` | Lists the attached skills with their names and descriptions. |
| `load_skill` | Reads a skill's full `SKILL.md` instructions. |
| `load_skill_resource` | Reads one file inside a skill directory, such as a reference document. |
| `read_file` | Reads a file from the skills directory or the session directory. |
| `write_file` | Writes a file to the session directory. |
| `edit_file` | Replaces an exact string in a file that the agent has already read. |
| `bash` | Runs a shell command in the session directory `/tmp/kagent/<session-id>/`. Commands time out after 30 seconds. |

Attaching a skill also changes what the agent is told. The runtime appends the name and description of every attached skill to the model request, along with an instruction to call `load_skill` before acting on one, so a skill reaches the model even before any tool is called.

> [!IMPORTANT]
> The `bash` tool gives the agent shell access inside its own Actor sandbox, and the sandbox is the boundary that contains it. Review a skill before you attach it, and treat the [egress]({{< link path="substrate-runtime/sandboxing" >}}) the Actor is granted as the reach the skill has. Writes are confined to the session directory, so a skill cannot modify `/skills` or another skill's files.

## Before you begin

1. [Install kagent]({{< link path="setup/installation" >}}).

2. [Create your first agent]({{< link path="get-started/your-first-agent" >}}), so that you have a Harness and an AgentTemplate to attach a skill to.

3. Install [Docker](https://docs.docker.com/get-started/get-docker/) to build the skill image.

4. Choose a container registry that your cluster can reach over HTTPS, and set it as an environment variable. Replace the example value with your own repository.
   ```bash
   export SKILL_REPO=ghcr.io/<your-org>/release-notes
   ```

   > [!WARNING]
   > kagent pulls a skill image over HTTPS with certificate verification, and v1alpha3 has no option to disable it. kagent 0.x accepted an `insecureSkipVerify` flag for a local registry, and that field does not exist in 1.x. A plain HTTP registry, and a `localhost` registry that only the host can reach, both fail at agent startup.

## Build the skill

A skill is a directory whose root holds a `SKILL.md` file. Everything else in the directory is available to the agent through the skill tools.

1. Create the skill directory.
   ```bash
   mkdir -p release-notes/scripts
   cd release-notes
   ```

2. Write `SKILL.md`. The YAML front matter must carry a `name` and a `description`, and the body holds the instructions the agent follows.
   ```bash
   cat > SKILL.md <<'EOF'
   ---
   name: release-notes
   description: Group a list of conventional commit subjects into release notes with Added, Fixed, and Changed sections. Use this skill whenever the user supplies raw commit subjects and wants them turned into release notes.
   ---
   # Release notes

   Turn raw commit subjects into release notes grouped by change type.

   ## Instructions

   1. Ask the user for the commit subjects if they have not supplied them. One subject per line.
   2. Write the subjects to `commits.txt` in your working directory with the `write_file` tool.
   3. Run `bash /skills/release-notes/scripts/group.sh commits.txt` with the `bash` tool.
   4. Return the script's output unchanged. Do not re-order or re-word the entries.

   ## Notes

   - The script reads conventional commit prefixes: `feat:` becomes Added, `fix:` becomes Fixed, and everything else becomes Changed.
   - A section with no entries is omitted.
   EOF
   ```

   The `description` decides whether the skill is ever used. The agent sees every attached skill's name and description, and chooses among them the same way it chooses any other tool, so state plainly when the skill applies. The instructions in the body are only read after the agent calls `load_skill`.

3. Add the script that the instructions call. The script runs in the agent's runtime image, so it uses `bash` rather than Python.
   ```bash
   cat > scripts/group.sh <<'EOF'
   #!/usr/bin/env bash
   # Group conventional commit subjects into release note sections.
   set -euo pipefail

   input="${1:?usage: group.sh <file>}"
   added="^feat(\([^)]*\))?!?:"
   fixed="^fix(\([^)]*\))?!?:"

   section() {
     local heading="$1" body="$2"
     [ -n "$body" ] || return 0
     printf '### %s\n%s\n\n' "$heading" "$body"
   }

   strip() {
     sed -E 's/^[a-z]+(\([^)]*\))?!?: *//; s/^/- /'
   }

   section Added   "$(grep -E  "$added" "$input" | strip || true)"
   section Fixed   "$(grep -E  "$fixed" "$input" | strip || true)"
   section Changed "$(grep -Ev "$added|$fixed" "$input" | strip || true)"
   EOF
   chmod +x scripts/group.sh
   ```

4. Confirm that the script works before you publish it. The `bash` tool returns a failed command's error to the model rather than to you, so a broken script produces an unreliable answer rather than a failed resource.
   ```bash
   printf 'feat: add checkpoint API\nfix: correct revision digest\ndocs: update install guide\n' > /tmp/commits.txt
   bash scripts/group.sh /tmp/commits.txt
   ```

   Example output:
   ```console
   ### Added
   - add checkpoint API

   ### Fixed
   - correct revision digest

   ### Changed
   - update install guide
   ```

## Publish the skill as an OCI image

kagent pulls an `oci` source as a container image and unpacks its flattened filesystem, so the image holds the skill directory and nothing else. Build it from `scratch`, which produces an image whose root **is** the skill root.

1. Create the Dockerfile.
   ```bash
   cat > Dockerfile <<'EOF'
   FROM scratch
   COPY . /
   EOF
   ```

2. Build and push the image. Build for the architecture your worker nodes run, because kagent pulls the `linux/amd64` or `linux/arm64` manifest that matches the node.
   ```bash
   docker buildx build --push --platform linux/amd64 -t "$SKILL_REPO:1.0.0" .
   ```

3. Read the image digest. An `oci` source must be pinned to a digest, and a tag alone is rejected.
   ```bash
   docker buildx imagetools inspect "$SKILL_REPO:1.0.0" | awk '/^Digest:/{print $2}'
   ```

   Example output:
   ```console
   sha256:9f2c1e4a7b3d5086c1a2f4e7b9d0c3a5e8f1b4d7a0c3e6f9b2d5a8c1e4f7b0d3
   ```

4. Save the digest reference.
   ```bash
   export SKILL_OCI="$SKILL_REPO@sha256:<your-digest>"
   ```

## Attach the skill to an AgentTemplate

1. Add a `skills` entry to the AgentTemplate that your Harness admits. Keep the labels and the model configuration that your existing template uses, and change only the name and the skill.
   ```bash
   kubectl apply -f - <<EOF
   apiVersion: kagent.dev/v1alpha3
   kind: AgentTemplate
   metadata:
     name: release-writer
     namespace: kagent
     labels:
       kagent.dev/harness: my-first-harness
   spec:
     modelConfig:
       name: default-model-config
     description: Writes release notes from commit subjects.
     systemPrompt: You help maintainers turn commit history into release notes.
     skills:
       - name: release-notes
         source:
           oci: ${SKILL_OCI}
   EOF
   ```

   Two fields carry the skill, and each is checked at a different time.

   | Field | Description |
   | ----- | ----------- |
   | `skills[].name` | The directory the skill is mounted under, and the name the agent sees. It must match no other skill on the template. |
   | `skills[].source.oci` | The digest-pinned image reference, in the form `<repository>@sha256:<digest>`. |

2. Confirm that the template compiled. The revision is ready when `desiredRevision` and `latestSuccessfulRevision` hold the same value.
   ```bash
   kubectl get agenttemplate release-writer -n kagent \
     -o jsonpath='{range .status.harnesses[*]}{.harness}{"\t"}{.desiredRevision}{"\t"}{.latestSuccessfulRevision}{"\n"}{end}'
   ```

   > [!NOTE]
   > A ready revision means that kagent accepted the reference, not that the image exists. kagent fetches the skill when the agent starts, so a wrong digest surfaces in the next step rather than this one.

3. Create an AgentInstance. An AgentInstance pins the revision it was created on, so an instance that already exists does not pick up the skill.
   ```bash
   kagent create agent-instance --harness my-first-harness --agent-template release-writer
   ```

4. Save the AgentInstance ID.
   ```bash
   export INSTANCE_ID=<your-agent-instance-id>
   ```

## Ask the agent to use the skill

1. Send the agent a request that matches the skill's description.
   ```bash
   kagent invoke --agent-instance $INSTANCE_ID \
     --task "Turn these commit subjects into release notes. feat: add checkpoint API. fix: correct revision digest. docs: update install guide."
   ```

2. Read the reply. The agent calls `load_skill` to read the instructions, `write_file` to stage the commit subjects, and `bash` to run the script, then returns the script's output.

   Example output:
   ```console
   ### Added
   - add checkpoint API

   ### Fixed
   - correct revision digest

   ### Changed
   - update install guide
   ```

3. Ask the agent what skills it holds, to confirm the attachment from the agent's own side.
   ```bash
   kagent invoke --agent-instance $INSTANCE_ID --task "What skills do you have?"
   ```

## Troubleshoot a skill that does not load

A skill that kagent cannot fetch stops the agent from starting at all, rather than producing an agent without that skill. The runtime logs the failure and exits, so the AgentInstance never reaches a state where you can talk to it.

1. Read the Actor's logs for the agent that will not start.
   ```bash
   kubectl logs -n kagent -l app.kubernetes.io/name=kagent-default --tail=50 | grep -i "materialize"
   ```

2. Match the message to its cause.

   | Message | Cause |
   | ------- | ----- |
   | `pull <image>: ... 401 Unauthorized` | The registry needs credentials that the cluster does not have. |
   | `pull <image>: ... x509` or a TLS error | The registry does not serve HTTPS with a certificate the runtime trusts. |
   | `SKILL.md is required` | The artifact was fetched, but no `SKILL.md` file sits at the root that `source.path` selects. |
   | `symlink "..." escapes artifact root` | A symlink in the artifact points outside it. |
   | `artifact contains more than 10000 filesystem entries`, or `artifact exceeds 104857600 bytes` | The artifact is over one of the package limits. |

> [!TIP]
> Build the skill image with `--platform` set to the architecture of your worker nodes. kagent asks the registry for the `linux/amd64` or `linux/arm64` manifest that matches the node it runs on, so an image published for one architecture alone fails on the other.

## Bundle the skill in a plugin package

A standalone source carries one skill. A {{< gloss "Plugin package" >}}plugin package{{< /gloss >}} carries several, and an AgentTemplate attaches the package once and names the skills it wants. Use a package when you ship a set of skills together, or when you want the same artifact to contribute [MCP servers]({{< link path="skills-and-mcp/plugins" >}}) as well.

1. Restructure the directory so that each skill sits under `skills/`, and add the manifest that makes it a package.
   ```bash
   cd ..
   mkdir -p release-tools/skills
   mv release-notes release-tools/skills/release-notes
   cd release-tools
   cat > plugin.json <<'EOF'
   {
     "$schema": "https://agent-plugins.org/schemas/1.0.0/plugin.schema.json",
     "name": "release-tools"
   }
   EOF
   ```

   > [!NOTE]
   > kagent compares the `$schema` value literally and rejects anything else, so copy it exactly. Of the remaining manifest fields, kagent reads only `name`.

2. Build and push the package, and read its digest. A package image is built the same way as a single skill, from `scratch`, so that the package root is the image root.
   ```bash
   export PLUGIN_REPO=ghcr.io/<your-org>/release-tools
   mv skills/release-notes/Dockerfile .
   docker buildx build --push --platform linux/amd64 -t "$PLUGIN_REPO:1.0.0" .
   docker buildx imagetools inspect "$PLUGIN_REPO:1.0.0" | awk '/^Digest:/{print $2}'
   ```

3. Attach the package with `plugins` instead of `skills`, and list the skills to enable.
   ```yaml
   spec:
     plugins:
       - source:
           oci: <your-plugin-digest-reference>
         skills:
           - release-notes
   ```

   > [!IMPORTANT]
   > Attaching a package enables nothing on its own. Only the names in `plugins[].skills` are turned on, so a package that gains a skill in a later version does not grant it to your agent until you add the name. An omitted or empty list is accepted and enables no skills.

## Publish a new version of the skill

A source is immutable, so changing a skill is a two-step change: publish new content, then point the AgentTemplate at it.

1. Edit the skill, then build and push it under a new tag and read the new digest.
   ```bash
   docker buildx build --push --platform linux/amd64 -t "$SKILL_REPO:1.1.0" .
   docker buildx imagetools inspect "$SKILL_REPO:1.1.0" | awk '/^Digest:/{print $2}'
   ```

2. Update `skills[].source.oci` on the AgentTemplate with the new digest, which compiles a new revision.

3. Create a new AgentInstance. Agents that are already running keep the skill content they started with, because their revision is pinned.

## Clean up

* Delete the AgentInstances that you created.
  ```bash
  kagent delete agent-instance $INSTANCE_ID
  ```
* Delete the AgentTemplate.
  ```bash
  kubectl delete agenttemplate release-writer -n kagent
  ```
* Delete the skill image from your registry, and remove the `release-notes` directory from your machine.

## Next steps

{{< cards >}}
  {{< card link=`{{< link path="skills-and-mcp/plugins" >}}` title="Plugins" subtitle="Bundle several skills, and MCP servers, into one package that an AgentTemplate attaches at once." >}}
  {{< card link=`{{< link path="skills-and-mcp/skills" >}}` title="Skills" subtitle="Read the full set of skill fields, source kinds, and naming rules." >}}
  {{< card link=`{{< link path="get-started/your-first-mcp-tool" >}}` title="Your first MCP tool" subtitle="Give the same agent a tool from an MCP server alongside its skills." >}}
{{< /cards >}}
