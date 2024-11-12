- Feature Name: oasis_liquidity_program
- Start Date: 2024-11-10
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)
- Author: Bulldozer <bulldozer@dozer.finance>

# Summary

Project Oasis is a liquidity incentive program designed to bootstrap liquidity for HTR pools, primarily focusing on bridged tokens including stablecoins and high-value assets such as BTC and ETH through a Nano Contract that provides impermanent loss protection, upfront bonuses, and fee sharing mechanisms. The program enables single-sided liquidity provision with tiered rewards based on lock duration, focusing on creating deep, sustainable liquidity pools while protecting liquidity providers.

# Motivation

Deep liquidity is fundamental to any DeFi ecosystem, but initial liquidity acquisition presents several challenges:

1. Initial liquidity providers face higher risks due to impermanent loss and uncertain returns
2. Early stage protocols often lack sufficient trading volume to generate attractive fee revenue
3. Traditional dual-token liquidity provision creates a high barrier to entry
4. Without protection mechanisms, liquidity tends to be unstable and short-term focused

Project Oasis addresses these challenges by:

- Providing immediate value through upfront HTR bonuses
- Protecting liquidity providers from impermanent loss
- Enabling single-sided liquidity provision
- Creating long-term aligned liquidity through timelocks
- Generating sustainable returns through trading fee distribution

The expected outcome is a foundation of deep, stable liquidity that can support the growth of DeFi on Hathor Network.

# Guide-level explanation

## Program Overview

Project Oasis introduces a new way for users to provide liquidity to Hathor Network's DeFi ecosystem. Users can deposit hUSDT, hETH or hBTC into the program, and the protocol will:

1. Match their deposit with an equal value of HTR
2. Provide an immediate bonus in HTR based on lock duration
3. Enable earning of trading fees from the provided liquidity
4. Protect against impermanent loss

## Lock Duration Options

Users can choose from three lock durations:

- 6 months: 10% upfront bonus in HTR
- 9 months: 15% upfront bonus in HTR
- 12 months: 20% upfront bonus in HTR

## Participation Example

Alice wants to provide liquidity with 10,000 hUSDT:

1. Alice chooses a 12-month lock period
2. She deposits 10,000 hUSDT
3. The protocol:
   - Matches with $10,000 worth of HTR
   - Immediately sends Alice a 20% bonus ($2,000 worth of HTR)
   - Creates LP position in Dozer Finance
4. Over the next 12 months:
   - Alice earns trading fees
   - Her position is protected from impermanent loss
   - She can monitor her position through the Dozer dApp interface

## User Benefits

1. Immediate Value
   - Upfront bonus in HTR
   - No need to provide the HTR
   - Protection against losses

2. Ongoing Returns
   - Trading fees from pool
   - Potential HTR appreciation
   - Protected principal

3. Simple User Experience
   - Single token deposit
   - Clear lock period
   - Automated protection

# Reference-level explanation

## Mathematical Framework

### Core Components

1. Initial Parameters & Definitions
Let:

- ```D_usdt:``` Initial USDT deposit amount
- ```P_htr_i:``` Initial HTR price
- ```P_htr_f:``` Final HTR price
- ```B_rate:``` Bonus rate (based on timelock)
- ```F_rate:``` Annual DEX fee rate (25%)
- ```T:``` Lock duration in months

2. Initial Position Calculations
Initial matched HTR amount:

```
M_htr = D_usdt / P_htr_i
```

Initial HTR bonus amount:

```
B_htr = (D_usdt × B_rate) / P_htr_i
```

3. Constant Product AMM Mathematics

The core AMM formula maintains:

```
x × y = K
```

Where:

- ```x:``` USDT amount in pool
- ```y:``` HTR amount in pool
- ```K:``` Constant product

Price ratio:

```
R = P_htr_f / P_usdt (where P_usdt = 1)
```

For any new price ratio R, new pool balances are:

```python
y/x = R
x × x × R = K
x = sqrt(K/R)
y = x × R
```

4. Protection & Withdrawal Mechanics

USDT deficit calculation:

```python
USDT_deficit = max(0, D_usdt - x_f)
```

Where ```x_f``` is final hUSDT pool balance

Protected withdrawal amounts:

- If USDT_deficit > 0:

```
HTR_protection = USDT_deficit/P_htr_f
```

- Else:

```python
HTR_protection = 0
```

1. Fee Generation

Annual fee accrual:

```python
F_annual = D_usdt × F_rate
```

Fees at time t months:

```
F_accrued = F_annual × (t/12)
```

6. Position Value Calculations

LP position value:

```
LP_value = USDT_withdrawal + (HTR_withdrawal × P_htr_f)
```

Bonus value scenarios:

```
Immediate sale: B_value_sell = B_htr × P_htr_i
Hold until withdrawal: B_value_hold = B_htr × P_htr_f
```

Total position value:

```
With sold bonus: V_total_sell = LP_value + B_value_sell
With held bonus: V_total_hold = LP_value + B_value_hold
```

7. Return Metrics

Return vs USDT hold:

```
Return_% = ((V_total / D_usdt) - 1) × 100
```

## Technical Implementation

### Contract State Variables

```python
class OasisPool(Blueprint):
    dozer_pool: ContractId  # Reference to AMM pool contract
    dev_address: Address    # Contract owner address
    dev_balance: Amount     # Available HTR for matching/rewards
    user_deposit: dict      # User hUSDT deposit amounts
    user_liquidity: dict    # User oasis  shares
    total_liquidity: Amount # Total oasis shares
    user_withdrawal_time: dict  # User timelock expiry timestamps
    user_balances: dict     # User reward/bonus balances
```

### Constants

```python
MIN_DEPOSIT = 10000_00  # Minimum HTR deposit
PRECISION = 10**20      # Decimal precision for calculations
MONTHS_IN_SECONDS = 30 * 24 * 3600  # Timelock period conversion
```

### Public Methods

#### initialize

```python
@public
def initialize(self, ctx: Context, dozer_pool: ContractId, token_b: TokenUid) -> None:

```

Initializes the Oasis contract by:

- Validating that token_a is HTR and token_b matches the provided token
- Requiring minimum HTR deposit from initializer (MIN_DEPOSIT=10000_00)
- Setting the contract owner address and initial balances
- Storing the Dozer pool contract reference

#### dev_deposit

```python
@public
def dev_deposit(self, ctx: Context) -> None:
```

Allows contract owner to deposit additional HTR for:

- Providing liquidity matching
- Funding IL protection reserves
- Adding bonus reward capacity

Only accepts HTR token deposits.

#### dev_withdraw

```python
@public
def dev_withdraw(self, ctx: Context) -> None:
```

Enables contract owner to withdraw HTR balance:

- Requires valid withdraw action
- Amount must be less than available dev_balance
- Only callable by contract owner address
- Updates dev_balance after withdrawal

#### user_deposit

```python
@public
def user_deposit(self, ctx: Context, timelock: int) -> None:
```

Handles user deposits of token_b with following flow:

- Validates timelock period (6, 9, or 12 months)
- Calculates matching HTR amount from pool
- Determines bonus reward based on timelock
- Updates user liquidity and total liquidity tracking
- Sets withdrawal timelock period
- Deposits funds into Dozer pool contract
- Updates user balances and timelock tracking

#### user_withdraw

```python
def user_withdraw(self, ctx: Context) -> None:
```

Processes user withdrawals with:

- Timelock validation check
- Calculates removable liquidity from Dozer pool
- Handles impermanent loss compensation
- Returns token_b principal + rewards
- Returns earned HTR bonus rewards
- Updates all tracking balances
- Only allows withdrawal after timelock expiry

# Drawbacks

1. Capital efficiency depends on Liquidity Pool Volume
2. Initial focus on single token (hUSDT) limits broader ecosystem growth
3. Protection mechanism may be expensive for Hathor Labs in extreme market conditions

# Rationale and alternatives

The chosen design prioritizes:

1. Single-sided deposits for accessibility
2. Fixed lock periods for predictable liquidity
3. Direct IL compensation mechanism
4. Integration with existing Dozer AMM

Alternatives considered:

1. Traditional dual-sided LP
   - Higher barrier to entry
   - No IL protection needed
   - Less capital efficient

2. Pure mining rewards
   - No upfront value
   - Higher ongoing costs
   - Less predictable returns

3. Variable lock periods
   - More complex management
   - Less predictable liquidity
   - Higher computational costs

# Prior art

Project Oasis draws inspiration from Radix's Ignition program while adapting it for Hathor's specific needs. Key differences include:

1. Focus on single token (hUSDT) vs multiple tokens
2. Simplified fee structure
3. Clear documentation for technical implementation and for program participants

# Unresolved questions

1. Is it necessary to include minimum and maximum deposit amounts?
2. Is it possible to upgrade the Protocol while it is live?

# Future possibilities

1. Extension to other tokens beyond hUSDT
2. Integration with future DeFi protocols
3. Governance-controlled parameters
4. Secondary market for locked positions
