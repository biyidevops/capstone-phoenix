# Cost Analysis — Capstone Phoenix

## Monthly Infrastructure Cost (eu-north-1, on-demand)

| Resource | Spec | Unit Cost | Monthly |
|----------|------|-----------|---------|
| EC2 control-plane | t3.medium (2 vCPU, 4GB) | $0.0416/hr | ~$30 |
| EC2 worker-1 | t3.medium (2 vCPU, 4GB) | $0.0416/hr | ~$30 |
| EC2 worker-2 | t3.medium (2 vCPU, 4GB) | $0.0416/hr | ~$30 |
| EBS gp3 volumes | 20GB × 3 nodes | $0.088/GB/mo | ~$5 |
| EBS PVC (Postgres) | 5GB gp3 | $0.088/GB/mo | ~$0.44 |
| Elastic IP (attached) | 1 EIP on running instance | Free | $0 |
| S3 remote state | Negligible (<1MB) | ~$0.023/GB | <$0.01 |
| DynamoDB lock table | On-demand, near-zero RCU/WCU | Pay-per-request | <$0.01 |
| Data transfer | ~10GB/mo estimate | $0.09/GB | ~$0.90 |
| **Total** | | | **~$96/month** |

## How to Cut Cost in Half

Switching all three nodes from on-demand t3.medium to **Spot instances** reduces 
EC2 cost by approximately 70% (Spot price for t3.medium in eu-north-1 is 
typically $0.012–0.015/hr vs $0.0416/hr on-demand). With a 3-node cluster and 
PodDisruptionBudgets ensuring at least one replica of each tier stays running 
during interruptions, Spot interruptions are survivable — Kubernetes reschedules 
evicted pods to remaining nodes within seconds. This alone brings the monthly 
bill from ~$96 to approximately $35–40. For a further reduction, downsizing 
workers to t3.small (1 vCPU, 2GB) once the cluster is stable under real load 
would save another $15/month, bringing total cost to roughly $25–30/month — 
less than a third of the current spend.
