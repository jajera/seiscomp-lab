# Authoring notes (not published)

just-the-docs site, same split as [jajera/privatelink-conduit](https://github.com/jajera/privatelink-conduit): short README, long commands only in `walkthrough.md`.

This lab is **manual** AWS CLI + on-box commands. Do not add an SSM script pile. Preview: `./scripts/docs-serve.sh`. Exclude this file in `_config.yml`.

## Nav

| Order | Page                 | Job                      |
| ----- | -------------------- | ------------------------ |
| 1     | `index.md`           | What you get             |
| 2     | `architecture.md`    | VPC and processes        |
| 3     | `seiscomp.md`        | gsm, DB, bindings        |
| 4     | `desktop.md`         | XFCE / xrdp              |
| 5     | `walkthrough.md`     | Type this                |
| 6     | `prove.md`           | Checks                   |
| 7     | `troubleshooting.md` | Pitfalls                 |
| 8     | `reference.md`       | File map + one inventory |

## Do not publish

- RDP passwords (SSM parameter only)
- Session `ssm-*.sh` payloads
- Live IDs on every page (one table in `reference.md` is enough)

## Later

Terraform can replace the VPC/EC2 CLI. The on-box SeisComP steps can stay manual.
