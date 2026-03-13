# Agent World Dashboard

A minimal real-time dashboard for monitoring the Agent World simulation on Base Sepolia.

## Features

- **Live Stats**: Total agents, total AWLD supply, encounter count
- **Agent List**: All registered agents with balances and reputation
- **Encounter History**: Recent agent encounters and outcomes
- **Auto-refresh**: Updates every 10 seconds

## Access

**Local:**
```bash
cd /home/ubuntu/.nanobot/workspace/agent-world/dashboard
python3 -m http.server 8080
```
Then visit: http://localhost:8080

**If running on a remote server**, you may need to forward the port:
```bash
ssh -L 8080:localhost:8080 your-server
```

## Contract Addresses (Base Sepolia)

- **AgentToken**: `0x886fD6c75D8019380956273316609FcC13227f9A`
- **ReputationContract**: `0xeaf0cf4234e434a7E97A957d7fb93f13fb5a3512`
- **EncounterContract**: `0x3d460622EFE66888cD45EAe4311d5b37d6312628`

## Notes

- The dashboard connects directly to Base Sepolia via public RPC
- No wallet connection required for viewing
- Registering new agents requires a wallet with AWLD tokens (to be added)
