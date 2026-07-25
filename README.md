# Hermes OCI capacity hunter

Retries an Oracle Cloud **Always Free** Ampere A1 launch (2 OCPU / 12 GB) in
`il-jerusalem-1` until capacity frees up, then relaunches the Hermes instance
from its **preserved boot volume** and pushes a phone notification.

Runs on GitHub's infrastructure — your laptop can be closed.

## Why this exists

The original instance (4 OCPU / 24 GB) was reclaimed after Oracle cut the
Always Free Ampere ceiling to **2 OCPU / 12 GB**. The boot volume survived, so
the data is intact — it just needs a host with free A1 capacity, which in
Jerusalem is frequently exhausted (`Out of host capacity`).

## Setup

### 1. Create a least-privilege OCI user

In the OCI Console (region **Israel Central (Jerusalem)**):

1. **Identity → Domains → Default → Users → Create user** → `hermes-launcher`
2. **Groups → Create group** `hermes-launchers`, add that user to it
3. **Policies → Create policy** in the root compartment, `hermes-launch-policy`:
   ```
   Allow group hermes-launchers to manage instance-family in tenancy
   Allow group hermes-launchers to manage volume-family in tenancy
   Allow group hermes-launchers to use virtual-network-family in tenancy
   ```
   `instance-family` (not bare `instances`) is required: launching from an
   existing boot volume creates a **boot-volume-attachment**, which bare
   `instances` does not cover. With the narrower form the launch fails with
   `NotAuthorizedOrNotFound`.
4. **Users → hermes-launcher → API keys → Add API key → Generate API key pair**
   → **Download the private key**, click Add, and copy the config snippet
   (it contains the user OCID and fingerprint).

### 2. Push this repo to GitHub

Use a **public** repo — scheduled workflows get unlimited free minutes there.
A private repo only has 2,000 min/month, which a 5-minute cron burns in days.
Secrets stay encrypted either way and are never exposed to forks.

```bash
cd ~/hermes-oci-hunter
git init && git add . && git commit -m "Hermes A1 capacity hunter"
gh repo create hermes-oci-hunter --public --source=. --push
```

### 3. Add the secrets

**Settings → Secrets and variables → Actions → New repository secret.**
Paste these yourself — never share the private key over chat or email.

| Secret | Value |
|---|---|
| `OCI_CLI_USER` | `ocid1.user.oc1..` OCID of `hermes-launcher` |
| `OCI_CLI_TENANCY` | `ocid1.tenancy.oc1..aaaaaaaai4pwjwxbob6uzjxzisy4c76dgvkdwnomstbmagobsc5lmai65yma` |
| `OCI_CLI_FINGERPRINT` | fingerprint shown in the console |
| `OCI_CLI_KEY_CONTENT` | **entire** private key file, including the `-----BEGIN/END PRIVATE KEY-----` lines |
| `OCI_CLI_REGION` | `il-jerusalem-1` |
| `NTFY_TOPIC` | a random, hard-to-guess topic, e.g. `hermes-il-7q3x9k` |

### 4. Get notified

Install the **ntfy** app (iOS/Android), subscribe to the same topic string.
ntfy topics are public to anyone who knows the name — make it unguessable.

### 5. Start it

The cron starts automatically once pushed. To test immediately:
**Actions → Hermes A1 capacity hunt → Run workflow.**

## Safety properties

- **Never exceeds the free tier.** Hard-coded to 2 OCPU / 12 GB, the exact
  Always Free ceiling. Boot volume is 100 GB, under the 200 GB free limit.
- **Never launches two VMs.** Every run first checks for a non-terminated
  instance and bails out if one exists; `concurrency` blocks overlapping runs.
- **Self-disables on success**, so the cron stops the moment you have the box.
- **Least privilege.** The key can manage compute/network/volumes only — not
  billing, not IAM.

## Notes

- GitHub delays scheduled runs under load; `*/5` often means 10–30 min in
  practice. Fine for a capacity hunt.
- GitHub auto-disables cron workflows after 60 days of repo inactivity.
- To stop early: **Actions → … → Disable workflow**.
- Re-enable after a future reclaim: enable the workflow again.

## Cleanup when done

Once Hermes is back, delete the API key in the OCI Console
(**Users → hermes-launcher → API keys → Delete**) so the credential stops
being live.
