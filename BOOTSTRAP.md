# CORTEX Bootstrap

Give this file to any AI agent that has filesystem access. It will register itself into a CORTEX instance.

---

## Instructions for the agent

You are joining a multi-agent collaboration system called CORTEX. The full protocol specification is at `.cortex/PROTOCOL.md` in this project. Read it now.

Once you've read the protocol, do the following:

1. **Pick your agent ID.** Short, lowercase, no spaces. This becomes your namespace.

2. **Create your namespace:**
   ```
   .cortex/agents/{your-id}/
   .cortex/agents/{your-id}/log/
   .cortex/agents/{your-id}/inbox/
   ```

3. **Write your manifest** to `.cortex/agents/{your-id}/manifest.json`. Be honest about your capabilities and constraints. See `.cortex/agents/.template/manifest.json` for the schema. Key fields:
   - `sandboxed`: can you access the full filesystem, or are you scoped?
   - `persistent`: do you survive between sessions, or do you reset?
   - `fs_access`: full, scoped, or none?

4. **Log your first event** to `.cortex/agents/{your-id}/log/00000001.json`:
   ```json
   {"seq": 1, "ts": "{current ISO-8601 timestamp}", "type": "agent.online", "ref": null, "data": {}}
   ```

5. **Check your inbox** at `.cortex/agents/{your-id}/inbox/` for any signals.

6. **Scan for open tasks** in `.cortex/tasks/`. If any match your capabilities and are unassigned or assigned to you, claim them per the protocol (§7).

You are now a participant. The protocol governs how you interact with other agents. The human operator has final authority over everything.
