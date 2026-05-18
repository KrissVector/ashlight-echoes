# User Communication Preferences

- Treat Chinese phrases such as "记得" and "记住" as explicit long-term memory signals when the user states a general preference or working rule.
- When explaining project content, put the relevant content directly in the chat instead of only giving file paths or telling the user to look it up.
- When mentioning English field names, include a direct Chinese explanation of what each field means and where it is used.
- Prefer converging on general reusable fields instead of introducing many narrow one-off fields; remove stale legacy wording and conflicting versions instead of leaving compatibility notes that pollute the project.
- For Ashlight Echoes design discussion, stay strictly within the existing documented rules and content framework. Do not invent mechanics, item properties, or new setting details; when a new design idea is needed, label it as a proposed addition and discuss it with the user before treating it as part of the project.
- Before every commit, sync the current local LLM memory files into the project `memory/` directory. After every pull, sync the project `memory/` directory back into the local LLM memory directory before continuing work. Keep the project README documenting this rule so other LLMs know to follow it after pulling the repo.
- The Ashlight Echoes project is intended to become a public beta / release-quality product, not a demo. Documentation and specs should be written to a standard that Codex or Claude Code can directly implement from. Do not leave half-migrated old versions, contradictory compatibility notes, or stale rules in the docs; make changes thoroughly, delete or replace obsolete content, and prioritize a complete coherent project over fast superficial fixes.
