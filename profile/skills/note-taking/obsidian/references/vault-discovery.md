# Vault discovery and fallback

Session finding:
- `OBSIDIAN_VAULT_PATH` was unset in this environment.
- The user-provided Windows path was not present here.
- No default vault existed at `~/Documents/Obsidian Vault`.
- A local vault was created successfully at:
  `/home/eliott_dardeau_pro/.hermes/profiles/deptflow/home/Obsidian Vault`

Practical workflow:
1. Check `OBSIDIAN_VAULT_PATH` first.
2. If unset, test `~/Documents/Obsidian Vault`.
3. If neither exists, create a local vault in the Hermes home directory and continue.
4. Always quote vault paths because they may contain spaces.
