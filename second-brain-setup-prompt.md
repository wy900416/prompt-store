# Prompt: install second-brain on a multi-harness machine (Amp + Grok + Copilot, no npx, no MCP)

Paste the block below into an Amp, Grok or Copilot session on the target machine.

---

Install the second-brain skill set by cloning and copying (no npx, no skills CLI),
then set up a vault. Do not skip the verification step.

1. Install:

     git clone --depth 1 https://github.com/nicholasspisak/second-brain /tmp/sb
     mkdir -p ~/.agents/skills
     cp -r /tmp/sb/skills/second-brain /tmp/sb/skills/second-brain-ingest \
           /tmp/sb/skills/second-brain-query /tmp/sb/skills/second-brain-lint \
           ~/.agents/skills/
     mkdir -p ~/.grok/skills
     for s in second-brain second-brain-ingest second-brain-query second-brain-lint; do
       ln -sfn ~/.agents/skills/$s ~/.grok/skills/$s
     done
     rm -rf /tmp/sb

   Verify before moving on: all four directories exist under ~/.agents/skills/ and
   each contains a SKILL.md; ~/.grok/skills/ holds four symlinks resolving into them.
   If verification fails, do not delete the clone — report what is missing.

   Why ~/.agents/skills: Amp, Copilot (personal skills) and Codex read it natively.
   Grok does not, hence the symlinks; Grok also reads ~/.claude/skills/ if that is
   easier on this machine. For repo-scoped skills use <repo>/.agents/skills/ or
   <repo>/.github/skills/ and commit them.

2. Create the vault at ~/second-brain:

     raw/
     wiki/sources/  wiki/entities/  wiki/concepts/  wiki/synthesis/
     output/
     wiki/index.md      (# Index, sections: Sources, Entities, Concepts, Synthesis; one line per page: - [[Page]] — summary)
     wiki/log.md        (# Log, append-only, entries: ## [YYYY-MM-DD] op | title)

3. Write ~/second-brain/AGENTS.md. Take the schema verbatim from
   ~/.agents/skills/second-brain/references/wiki-schema.md, then add three sections
   of your own:

   Reading order — read wiki/index.md first; open a page only when the index points
   at it; follow [[wikilinks]]; touch raw/ only as a last resort; prefer
   `qmd search` over eyeballing the index once past ~100 pages.

   Memory routing — preferences → wiki/concepts/preferences.md (one page, appended,
   never duplicated); per-project lessons → wiki/synthesis/lessons-<project>.md
   (what happened / what to do next / why); decisions →
   wiki/synthesis/decisions-<project>.md, superseded ones marked
   [superseded YYYY-MM-DD] instead of deleted.

   Never — modify raw/; invent a second directory layout; store secrets; answer from
   assumption when a grep of the wiki would settle it.

   Also add: at the end of a task, append one or two reusable bullets to the right
   page and index it if new. Do not dump session transcripts.

4. Point all three harnesses at the vault. Add this line to ~/.config/AGENTS.md (Amp),
   ~/.grok/AGENTS.md (Grok) and, for Copilot, either the same file if your setup reads
   it or .github/copilot-instructions.md in a repo:

   Personal knowledge base at ~/second-brain. When a question may already be answered
   there, read its AGENTS.md, then wiki/index.md, then only the pages the index points
   to. Write durable conclusions back before finishing.

5. Report: exact paths written, the verification listing for step 1, and whether each
   global instructions file was created or appended to. Show me the AGENTS.md you wrote
   before you start using it.

---

## Notes for the human

- The day-to-day skills work through the folders in `~/.agents/skills/`; nothing else
  is needed. `.grok/skills/` only exists because Grok does not read the shared location.
- The schema in step 3 is copied from the skill's own `references/wiki-schema.md`;
  do not let the agent invent a structure.
- Step 5 matters: `AGENTS.md` at the vault root is the only file loaded every session,
  so a sloppy one means nothing is actually shared.
- Without MCP there is no automatic capture: the write-back line in `AGENTS.md` is the
  whole mechanism.
- Updating later is `git -C <clone> pull` then repeat the `cp`/`ln` lines — the four
  folders are self-contained, so overwriting them is safe.
