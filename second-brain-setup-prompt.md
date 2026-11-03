# Prompt: install second-brain on a multi-harness machine (Amp + Grok, no MCP)

Paste the block below into an Amp or Grok session on the target machine.

---

Install the second-brain skill set globally, then set up a vault. Do not skip
the verification step.

1. Install (repeat -s once per skill; a comma list fails):

     npx skills add nicholasspisak/second-brain -s second-brain -s second-brain-ingest -s second-brain-query -s second-brain-lint -g -y

   Then run `npx skills ls` and confirm all four are listed.

2. Create the vault at ~/second-brain:

     raw/
     wiki/sources/  wiki/entities/  wiki/concepts/  wiki/synthesis/
     output/
     wiki/index.md      (# Index, sections: Sources, Entities, Concepts, Synthesis; one line per page: - [[Page]] — summary)
     wiki/log.md        (# Log, append-only, entries: ## [YYYY-MM-DD] op | title)

3. Write ~/second-brain/AGENTS.md. Take the schema verbatim from the installed
   skill at ~/.agents/skills/second-brain/references/wiki-schema.md (or wherever
   `npx skills ls` says it landed), then add three sections of your own:

   Reading order — read wiki/index.md first; open a page only when the index
   points at it; follow [[wikilinks]]; touch raw/ only as a last resort; prefer
   `qmd search` over eyeballing the index once past ~100 pages.

   Memory routing — preferences → wiki/concepts/preferences.md (one page,
   appended, never duplicated); per-project lessons →
   wiki/synthesis/lessons-<project>.md (what happened / what to do next / why);
   decisions → wiki/synthesis/decisions-<project>.md, superseded ones marked
   [superseded YYYY-MM-DD] instead of deleted.

   Never — modify raw/; invent a second directory layout; store secrets; answer
   from assumption when a grep of the wiki would settle it.

   Also add: at the end of a task, append one or two reusable bullets to the
   right page and index it if new. Do not dump session transcripts.

4. Point both harnesses at the vault. Add this line to ~/.config/AGENTS.md (Amp)
   and to ~/.grok/AGENTS.md (Grok):

   Personal knowledge base at ~/second-brain. When a question may already be
   answered there, read its AGENTS.md, then wiki/index.md, then only the pages
   the index points to. Write durable conclusions back before finishing.

5. Report: exact paths written, `npx skills ls` output, and whether both global
   AGENTS.md files were created or appended to. Show me the AGENTS.md you wrote
   before you start using it.

---

## Notes for the human

- The schema in step 3 is copied from the skill's own
  `references/wiki-schema.md`; do not let the agent invent a structure.
- Step 5 matters: `AGENTS.md` at the vault root is the only file loaded every
  session, so a sloppy one means nothing is actually shared.
- Amp reads `~/.config/AGENTS.md` (and `~/.config/amp/AGENTS.md`); Grok reads
  `~/.grok/AGENTS.md`. Both also read an `AGENTS.md` in the working directory
  and its parents, so running inside the vault needs no global config at all.
- Without MCP there is no automatic capture: the write-back line in `AGENTS.md`
  is the whole mechanism.
