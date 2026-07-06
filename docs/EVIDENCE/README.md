# EVIDENCE

Drop screenshots/logs here, named so a grader knows what each proves:

- `nodes-ready.png` — multi-node `kubectl get nodes`
- `pods-spread.png` — replicas on different nodes (`-o wide`)
- `tls-valid.png` — valid cert (curl -vI / SSL Labs)
- `pvc-persist.log` — data survives a Pod kill
- `zero-downtime.log` — unbroken 200s during a rollout
- `hpa-scale.png` — replicas climbing under load
- `argocd-synced.png` — Argo CD Synced + Healthy
- `failover.png` — app up after a node drain

## Live Demo Recording

https://drive.google.com/file/d/1q4HW6HsDUTkFy8B0OBbQT6KUkcdvbp_N/view?usp=sharing


Shows: taskapp.techproconsults.com live over HTTPS, login, task creation, Kanban board.
