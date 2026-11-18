# Deployment and infrastructure details

## What's sensitive vs. fine to show

Usually fine, and good portfolio signal: *how* it deploys. Containerized, which CI, which kind of host, the shape of the pipeline.

Usually sensitive: *where*, and *with what access*:
- hostnames, IPs, SSH users/ports, bastion hosts, VPN endpoints
- cloud account/project/subscription IDs, ARNs, bucket names, resource group names
- internal or admin URLs, staging URLs not meant to be public, DNS layout
- `*.tfstate` (contains secrets in plain text, always ignore), `*.tfvars`, kubeconfig, inventory files
- provider-linked metadata dirs: `.vercel/`, `.netlify/`, `.firebaserc` project IDs, `.fly/`
- PM2 `ecosystem.config.js` deploy blocks (user@host + repo + path), Capistrano/Ansible inventories, nginx/Caddy configs naming real domains and paths

Ask the user which applies. "Hide deployment details" can mean anything from "don't leak my server IP" to "don't reveal I host on X at all".

## Options (most to least visible)

1. **Parameterize and template (default).** Real values come from env vars or an ignored local file. Commit a template with placeholders:
   - `deploy.example.sh`, `terraform.tfvars.example`, `docker-compose.override.example.yml`, `ecosystem.config.example.js`
   - workflows reference `${{ secrets.DEPLOY_HOST }}` / `${{ vars.X }}`
   - The process stays visible, the targets don't. Best for portfolio projects.
2. **Keep deploy config local only.** A `deploy/` folder that never gets committed. If the folder name itself is harmless, use `.gitignore`. If it reveals something (`deploy-hetzner-clientname/`), use `.git/info/exclude`. Downside: no version history for it.
3. **Separate private infra repo.** The best fit when infra is substantial (Terraform, k8s manifests, Ansible). The app repo stays clean; infra gets real history and review. The README can say "deployed via a separate infrastructure repo".
4. **Remove entirely.** Only if the user wants no trace and has a copy elsewhere. Never delete without confirming where it will live.

## Conventions worth adopting

- `*.local.*` and `.env.*.local` mean "this machine only" (in `assets/gitignore-base`)
- `docker-compose.override.yml` is local and ignored; commit `docker-compose.yml` with safe defaults
- `terraform.tfvars` ignored, `terraform.tfvars.example` committed, `*.tfstate*` and `.terraform/` always ignored, remote state backend preferred
- README "Deployment" section: describe the approach and the required variables, not the targets

## Check after fixing

Re-check the deployment patterns from `scan-checklist.md` §5. Also grep the README, docs and CI files for the real domain and host names, and add them to `.git/info/private-terms` so the hook keeps catching them.
