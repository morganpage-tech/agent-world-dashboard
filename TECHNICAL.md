# Agent World - Technical Breakdown

## Overview

Agent World is a multi-agent economic simulation running on the Base Sepolia testnet. It models repeated game theory interactions between autonomous agents using the Prisoner's Dilemma framework, extended with reputation tracking, economic incentives, and **LLM-powered AI decision making**.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Agent World System                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐        │
│  │   Agent 1   │     │   Agent 2   │     │   Agent N   │        │
│  │  (Python)   │     │  (Python)   │     │  (Python)   │        │
│  └──────┬──────┘     └──────┬──────┘     └──────┬──────┘        │
│         │                   │                   │                │
│         │    ┌──────────────┼──────────────┐   │                │
│         │    │              │              │   │                │
│         │    │   Rule-based│   LLM Agents│   │                │
│         │    │  Strategies  │ (GPT-4o-mini)│  │                │
│         │    └──────────────┼──────────────┘   │                │
│         └───────────────────┼───────────────────┘                │
│                             │                                    │
│                    ┌────────▼────────┐                          │
│                    │  Runner Script   │                          │
│                    │  (continuous)    │                          │
│                    └────────┬────────┘                          │
│                             │                                    │
│                    ┌────────▼────────┐                          │
│                    │ AgentWorldClient│                          │
│                    │     (ethers)    │                          │
│                    └────────┬────────┘                          │
│                             │                                    │
│         ┌───────────────────┼───────────────────┐                │
│         │                   │                   │                │
│  ┌──────▼──────┐    ┌──────▼──────┐    ┌──────▼──────┐         │
│  │ Encounter   │    │ Reputation  │    │  Agent      │         │
│  │  Contract   │    │  Contract   │    │   Token     │         │
│  └─────────────┘    └─────────────┘    └─────────────┘         │
│                                                                 │
│                          Base Sepolia                           │
└─────────────────────────────────────────────────────────────────┘
```

## Smart Contracts

### EncounterContract
**Address:** `0x3d460622EFE66888cD45EAe4311d5b37d6312628`

The core game logic contract that manages agent encounters and resolution.

#### Key Functions

**createEncounter(address opponent, uint256 stakeAmount)**
- Creates a new encounter between two agents
- Both agents must approve stakeAmount of AWLD tokens
- Returns unique encounter ID
- Emits `EncounterCreated` event

**submitAction(uint256 encounterId, Action action)**
- Agents submit their action: COOPERATE, DEFECT, or CHALLENGE
- Both actions trigger automatic resolution
- Emits `ActionSubmitted` event

#### Payoff Matrix

| Agent 1 | Agent 2 | Result |
|---------|---------|--------|
| COOPERATE | COOPERATE | Both get stake + 25% bonus (+15 reputation) |
| COOPERATE | DEFECT | Agent 2 gets all stake (-20 vs -5 reputation) |
| DEFECT | COOPERATE | Agent 1 gets all stake (-5 vs -20 reputation) |
| DEFECT | DEFECT | Both lose 50% stake (-10 reputation each) |
| CHALLENGE | * | Winner takes all based on random/parity |

### ReputationContract
Tracks agent reputation scores based on their actions.

#### Key Functions
- `registerAgent()` - Register new agent
- `updateReputation(address agent, int256 delta, string action)` - Update score
- `getReputation(address agent)` - Get agent stats
- `getLeaderboard()` - Get all agents sorted by score

### AgentToken
Standard ERC20 token used for staking and rewards.

## Agent Strategies (Python)

### Implemented Strategies

#### Rule-Based Strategies (10 agents)

| Strategy | Description | Behavior |
|----------|-------------|----------|
| **AlwaysCooperate** | Trusts everyone | Always COOPERATE |
| **AlwaysDefect** | Exploitative | Always DEFECT |
| **Random** | No strategy | Random action |
| **TitForTat** | Reciprocal | COOPERATE first, copy opponent's last move |
| **Friedman** (Grudger) | Zero tolerance | COOPERATE until opponent defects once |
| **Joss** (TitForTwoTats) | Forgiving | COOPERATE until 2 consecutive defections |
| **Pavlov** | Win-Stay, Lose-Shift | Repeat if reward was good, switch if bad |
| **Reputation** | Trust-based | COOPERATE only with high-reputation agents |
| **Hunter** | Exploitative | Targets high-reputation cooperators |
| **Adaptive** | Mode-switching | Adjusts strategy based on performance |

#### LLM-Powered AI Agents (3 agents - GPT-4o-mini)

| Strategy | Temperature | Description | Behavior |
|----------|-------------|-------------|----------|
| **LLM_Conservative** | 0.3 (Low) | Predictable, cautious | Favors cooperation, builds long-term relationships, risk-averse |
| **LLM_Balanced** | 0.7 (Medium) | Balanced exploration/exploitation | Adapts to opponent patterns, strategic flexibility |
| **LLM_Aggressive** | 1.0 (High) | Exploratory, risk-taking | Tests novel strategies, unconventional approaches |

### LLM Agent Architecture

```python
class LLMAgent(Agent):
    """
    AI agent that uses GPT-4o-mini to make strategic decisions.
    """

    def decide(self, opponent: str) -> Action:
        # Gather context: history, reputation, patterns
        context = self._build_context(opponent)

        # Build prompt with game theory framework
        prompt = self._build_prompt(context)

        # Query LLM with temperature-controlled sampling
        response = openai.chat.completions.create(
            model="gpt-4o-mini",
            messages=[
                {"role": "system", "content": "You are an AI agent in Agent World..."},
                {"role": "user", "content": prompt}
            ],
            temperature=self.temperature,  # 0.3, 0.7, or 1.0
            max_tokens=100
        )

        # Parse LLM decision into Action
        return self._parse_decision(response.choices[0].message.content)
```

### Agent Implementation (Base Class)

```python
class Agent:
    def __init__(self, address: str, strategy: str):
        self.address = address
        self.strategy = strategy
        self.history = {}  # opponent -> [Encounters]
        self.my_last_actions = {}  # opponent -> last_action

    def decide(self, opponent: str) -> Action:
        # Override in strategy classes
        pass
```

## Runner System

### continuous_runner.py
The main orchestrator that:
1. Continuously generates random agent pairings
2. Creates encounters on-chain (30 second delay between rounds)
3. Submits agent actions based on their strategies
4. Monitors encounter resolution
5. Logs all activities

### Key Features
- **Stake Amount:** 10 AWLD per encounter
- **Delay:** 30 seconds between rounds
- **Error Recovery:** Automatically retries failed transactions
- **Log Parsing:** Extracts encounter IDs from transaction logs

## Dashboard

### Technology Stack
- **Frontend:** Vanilla JavaScript + HTML
- **Blockchain:** ethers.js v6.9.0
- **Charts:** Chart.js 4.4.0
- **AI/LLM:** OpenAI GPT-4o-mini API (3 agents)
- **Deployment:** Vercel (auto-deploys from GitHub)

### Features
1. **Encounter Table** - Shows last 10 encounters with actions and rewards
2. **Real-time Updates** - Refresh button to fetch latest data
3. **Analytics**
   - Action distribution (COOPERATE vs DEFECT vs CHALLENGE)
   - Top 5 reputation leaderboard
   - Cooperation rate trend over time
4. **Rate Limiting Handling** - Retry logic with delays for RPC calls

### Performance Optimizations
- Sequential fetching with delays (avoid RPC rate limiting)
- Retry logic (up to 3 attempts)
- Efficient data caching
- Lazy loading of charts

## Game Theory Insights

### The Prisoner's Dilemma Extension
Agent World extends the classic Prisoner's Dilemma with:
1. **Reputation System** - Long-term consequences for actions
2. **Economic Incentives** - Token staking and rewards
3. **Multiple Strategies** - Competition between different approaches
4. **Repetition** - Agents learn from past encounters

### Strategy Performance
- **TitForTat** historically dominates in repeated games
- **Pavlov** performs well in noisy environments
- **AlwaysDefect** exploits naive cooperators but gets punished
- **Adaptive** strategies can adapt to the meta-game

## Development

### Smart Contract Development
```bash
forge build          # Compile contracts
forge test           # Run tests
forge script Deploy.s.sol --rpc-url https://sepolia.base.org --broadcast
```

### Running the Runner
```bash
cd /home/ubuntu/.nanobot/workspace/agent-world
source agents/venv/bin/activate
# Set OpenAI API key for LLM agents
export OPENAI_API_KEY=your-api-key-here
python3 scripts/continuous_runner.py
```

### Checking State
```bash
python3 scripts/check_state.py  # Show encounter status
```

### LLM Agent Setup
The LLM agents require an OpenAI API key:
1. Set `OPENAI_API_KEY` environment variable
2. Or pass `api_key` parameter when creating LLM agents
3. Default model: `gpt-4o-mini` (fast, cost-effective)
4. Temperature values: 0.3 (conservative), 0.7 (balanced), 1.0 (aggressive)

## Deployment Information

### Base Sepolia Testnet
- **Network:** Base Sepolia
- **Explorer:** https://sepolia.basescan.org/
- **RPC:** https://sepolia.base.org
- **Encounter Contract:** `0x3d460622EFE66888cD45EAe4311d5b37d6312628`
- **Stake per Encounter:** 10 AWLD
- **Total Encounters:** 200 (planned)

### Agent Configuration
Located at: `/home/ubuntu/.nanobot/workspace/agent-world/agents/agent-config.json`

## Future Enhancements

1. **More AI Agents** - Implement additional LLM models (Claude, Gemini) for comparison
2. **Hybrid Strategies** - Combine game theory with LLM decision making
3. **Skill-based Challenges** - Replace random winner with skill metric
4. **Agent Migration** - Allow agents to move between contracts
5. **Tournament Mode** - Season-based competitions
6. **Advanced Analytics** - Machine learning insights on strategy evolution
7. **Prompt Engineering** - Optimize LLM prompts for better strategic reasoning
8. **Multi-turn Conversations** - Allow LLM agents to negotiate before encounters

## References

- Axelrod, Robert. *The Evolution of Cooperation* (1984)
- Prisoner's Dilemma: https://en.wikipedia.org/wiki/Prisoner%27s_dilemma
- Foundry Documentation: https://book.getfoundry.sh/
- ethers.js: https://docs.ethers.org/

---

*Built for EF Bounty: Agent World - Multi-Agent Economic Simulation*
