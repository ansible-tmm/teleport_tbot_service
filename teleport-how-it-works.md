# How Teleport tbot Works

## High-Level Flow

### 1) Human Bootstrap (Interactive, MFA)

You (a human) log in once using:
- Username + password
- MFA / passkey / SSO
- `tsh login`

**Purpose:** This only exists so you're allowed to administer Teleport, not so jobs can run.

### 2) Create or Re-issue a Bot Join Token

You create (or re-issue) a machine enrollment token for a bot identity:

```bash
tctl bots instances add aap-bot
```

This token is:
- Single-use
- Short-lived (e.g. 59 min)
- Only for initial enrollment

**Think of this like:** "Prove this machine is allowed to become aap-bot"

### 3) tbot start Enrolls the Machine

You start tbot using that token:

```bash
tbot start \
  --destination-dir=... \
  --data-dir=... \
  --join-method=token \
  --token=...
```

**What happens here:**
- The token is consumed
- Teleport issues a machine identity
- That identity is stored in `--data-dir`
- SSH certs + ssh_config are written to `--destination-dir`

**After this step:** The machine is now trusted

### 4) tbot Runs Continuously

This is the most important part.

**tbot:**
- Stays running
- Periodically refreshes short-lived certs
- Does not require MFA again
- Does not require new tokens
- Keeps SSH certs valid forever

**Ansible, SSH, SCP, etc:**
- Just read files from destination-dir
- Never talk directly to Teleport APIs
- Never prompt for auth
