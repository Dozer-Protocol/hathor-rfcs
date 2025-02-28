- Feature Name: oasis_liquidity_program
- Start Date: 2024-11-10
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)
- Authors: Bulldozer <bulldozer@dozer.finance> and Tothster <tothster@dozer.finance>

# Summary

Project Oasis is a liquidity incentive program designed to bootstrap liquidity for HTR pools. Through a Nano Contract that provides impermanent loss protection, upfront bonuses, and fee-sharing mechanisms, it primarily focuses on bridged tokens, including stablecoins and high-value assets such as USDC, BTC, and ETH. The program enables single-sided liquidity provision with tiered rewards based on lock duration, focusing on creating deep, sustainable liquidity pools while protecting liquidity providers.

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

The expected outcome is a foundation of deep, stable liquidity that can support DeFi's growth on the Hathor Network.

# Guide-level explanation

## Program Overview

Project Oasis introduces a new way for users to provide liquidity to Hathor Network's DeFi ecosystem. Users can deposit hUSDC, hETH, or hBTC into the program, and the protocol will:

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

Alice wants to provide liquidity with 10,000 hUSDC:

1. Alice chooses a 12-month lock period
2. She deposits 10,000 hUSDC
3. The protocol:
   - Matches with $10,000 worth of HTR
   - Immediately sends Alice a 20% bonus ($2,000 worth of HTR)
   - Creates LP position in Dozer Finance
4. Over the next 12 months:
   - Alice earns trading fees
   - Her position is protected from impermanent loss
   - She can monitor her position through the Dozer dApp interface

5. When her lock period expires:
   - Alice initiates position closing to remove liquidity from the pool
   - The protocol calculates and stores her exact withdrawal amounts
   - Alice can then withdraw these amounts in a separate transaction

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
   - Slippage-protected withdrawals

# Reference-level explanation

## Mathematical Framework

### Core Components

1. Initial Parameters & Definitions
Let:

- ```D_usdc:``` Initial USDC deposit amount
- ```P_htr_i:``` Initial HTR price
- ```P_htr_f:``` Final HTR price
- ```B_rate:``` Bonus rate (based on timelock)
- ```F_rate:``` Annual DEX fee rate (25%)
- ```T:``` Lock duration in months

2. Initial Position Calculations
Initial matched HTR amount:

```
M_htr = D_usdc / P_htr_i
```

Initial HTR bonus amount:

```
B_htr = (D_usdc × B_rate) / P_htr_i
```

3. Constant Product AMM Mathematics

The core AMM formula maintains:

```
x × y = K
```

Where:

- ```x:``` USDC amount in pool
- ```y:``` HTR amount in pool
- ```K:``` Constant product

Price ratio:

```
R = P_htr_f / P_usdc (where P_usdc = 1)
```

For any new price ratio R, new pool balances are:

```python
y/x = R
x × x × R = K
x = sqrt(K/R)
y = x × R
```

4. Protection & Withdrawal Mechanics

USDC deficit calculation:

```python
USDC_deficit = max(0, D_usdc - x_f)
```

Where ```x_f``` is final hUSDC pool balance

Protected withdrawal amounts:

- If USDC_deficit > 0:

```
HTR_protection = USDC_deficit/P_htr_f
```

- Else:

```python
HTR_protection = 0
```

5. Fee Generation

Annual fee accrual:

```python
F_annual = D_usdc × F_rate
```

Fees at time t months:

```
F_accrued = F_annual × (t/12)
```

6. Position Value Calculations

LP position value:

```
LP_value = USDC_withdrawal + (HTR_withdrawal × P_htr_f)
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

Return vs USDC hold:

```
Return_% = ((V_total / D_usdc) - 1) × 100
```

## Technical Implementation

### Contract State Variables

```python
class OasisPool(Blueprint):
    dozer_pool: ContractId  # Reference to AMM pool contract
    dev_address: Address    # Contract owner address
    dev_balance: Amount     # Available HTR for matching/rewards
    user_deposit: dict      # User hUSDC deposit amounts
    user_liquidity: dict    # User oasis shares
    total_liquidity: Amount # Total oasis shares
    user_withdrawal_time: dict  # User timelock expiry timestamps
    user_balances: dict     # User reward/bonus balances
    closed_position_balances: dict  # User balances after position close
    user_position_closed: dict  # Flag indicating if position is closed
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

#### close_position

```python
@public
def close_position(self, ctx: Context) -> None:
```

Closes a user's position with the following flow:

- Verifies timelock period has elapsed
- Checks that position exists and is not already closed
- Removes liquidity from Dozer pool
- Calculates impermanent loss compensation if needed
- Stores exact token amounts in closed_position_balances
- Moves any cashback/bonus funds to closed_position_balances
- Marks position as closed with user_position_closed flag
- Clears original user_balances entries

#### user_withdraw

```python
@public
def user_withdraw(self, ctx: Context) -> None:
```

Processes user withdrawals with:

- Verifies position has been closed first
- Transfers tokens based on stored closed_position_balances
- Updates closed_position_balances after withdrawal
- Cleans up user data if all funds are withdrawn
- Only allows withdrawal after position is closed

#### View Methods

```python
@view
def get_remove_liquidity_oasis_quote(self, ctx: Context, address: Address) -> tuple[Amount, Amount]:
```

Returns quote for liquidity removal:
- Returns values from closed_position_balances if position is closed
- Otherwise calculates expected withdrawal amounts based on current pool state

```python
@view
def user_info(self, ctx: Context, address: Address) -> tuple[Amount, Amount, Amount, bool, Amount, Amount]:
```

Returns user information including:
- Deposit amounts
- Liquidity shares
- Timelock expiry
- Position closed status
- Closed position token balances

# Drawbacks

1. Initial focus on single token (hUSDC) limits broader ecosystem growth
2. The protection mechanism may be expensive for Hathor Labs in extreme market conditions
3. The two-step withdrawal process requires users to submit multiple transactions

# Rationale and alternatives

The chosen design prioritizes:

1. Single-sided deposits for accessibility
2. Fixed lock periods for predictable liquidity
3. Direct IL compensation mechanism
4. Integration with existing Dozer AMM
5. Slippage protection through a two-step withdrawal process

Alternatives considered:

1. Traditional dual-sided LP
   - Higher barrier to entry
   - Less capital efficient

2. Pure mining rewards
   - No upfront value
   - Higher ongoing costs
   - Less predictable returns

3. Variable lock periods
   - More complex management
   - Less predictable liquidity
   - Higher computational costs

4. Single-step withdrawal with slippage
   - Simpler user experience
   - Risk of value loss between transaction submission and execution
   - Unpredictable final token amounts

# Prior art

Project Oasis draws inspiration from Radix's Ignition program while adapting it for Hathor's needs. Key differences include:

1. Focus on single token (hUSDC) vs multiple tokens
2. Simplified fee structure
3. Clear documentation for technical implementation and program participants
4. Two-step withdrawal process for slippage protection

# Unresolved questions

1. Is it necessary to include minimum and maximum deposit amounts?
2. Is it possible to upgrade the Protocol while it is live?
3. Should partial withdrawals be permitted from closed positions?
4. How should the dApp handle the new two-step withdrawal process?

# Future possibilities

1. Extension to other tokens beyond EVM Tokens
2. Integration with future DeFi protocols
3. Governance-controlled parameters
4. Secondary market for locked positions
5. Simplified one-click UI for two-step position closing/withdrawal
