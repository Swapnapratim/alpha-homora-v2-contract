# Alpha Homora v2 Protocol: Comprehensive Technical Research Documentation

## Summary

Alpha Homora v2 is a sophisticated DeFi protocol that implements leveraged yield farming through a modular smart contract architecture. The protocol's core innovation lies in its debt tracking management system, which utilizes a bitmap-based approach for efficient position management across multiple lending protocols. This document provides a comprehensive analysis of the protocol's technical architecture, debt management mechanisms, and integration patterns.

## Table of Contents

1. Protocol Architecture Overview
2. Core Smart Contract Analysis
3. Debt Tracking Management System
4. Lending Protocol Integration
5. Spell System Architecture
6. Oracle Infrastructure
7. Wrapper Contract System
8. Mathematical Foundations
9. Security Mechanisms
10. Deployment and Testing Infrastructure
11. Technical Specifications
12. References

## 1. Protocol Architecture Overview

### 1.1 Core Components

The Alpha Homora v2 protocol consists of several interconnected smart contract modules:

- **HomoraBank**: Central contract managing positions, debt, and lending protocol integration
- **Spell System**: Strategy execution contracts for different DeFi protocols
- **Oracle Infrastructure**: Multi-source price feed system with fallback mechanisms
- **Wrapper Contracts**: ERC1155 tokenization layer for LP tokens and staking positions
- **Utility Contracts**: Mathematical libraries and helper functions

### 1.2 System Architecture

The protocol follows a modular design pattern where:
- HomoraBank acts as the central coordinator
- Spells execute specific DeFi strategies
- Oracles provide price feeds for collateral and debt valuation
- Wrapper contracts standardize different token types into ERC1155 format

## 2. Core Smart Contract Analysis

### 2.1 HomoraBank Contract

#### 2.1.1 Contract Structure

```solidity
contract HomoraBank is Governable, ERC1155NaiveReceiver, IBank {
    struct Bank {
        bool isListed;
        uint8 index;
        address cToken;
        uint reserve;
        uint totalDebt;
        uint totalShare;
    }
    
    struct Position {
        address owner;
        address collToken;
        uint collId;
        uint collateralSize;
        uint debtMap;
        mapping(address => uint) debtShareOf;
    }
}
```

#### 2.1.2 Key State Variables

- `_GENERAL_LOCK`: Reentrancy protection mechanism
- `_IN_EXEC_LOCK`: Execution scope protection
- `POSITION_ID`: Current executing position identifier
- `SPELL`: Current executing spell contract address
- `caster`: HomoraCaster contract for untrusted execution
- `oracle`: Price oracle system reference
- `feeBps`: Protocol fee in basis points (default: 1000 = 10%)

#### 2.1.3 Bank Management

The Bank struct represents a lending market with:
- `isListed`: Market existence flag
- `index`: Position in allBanks array (0-255)
- `cToken`: Lending protocol token address
- `reserve`: Protocol reserve allocation
- `totalDebt`: Aggregate debt across all positions
- `totalShare`: Total debt shares outstanding

#### 2.1.4 Position Management

The Position struct tracks user positions with:
- `owner`: Position owner address
- `collToken`: ERC1155 collateral token address
- `collId`: Collateral token identifier
- `collateralSize`: Collateral amount
- `debtMap`: Bitmap of active debt banks
- `debtShareOf`: Debt shares per bank token

### 2.2 HomoraCaster Contract

```solidity
contract HomoraCaster {
    function cast(address target, bytes calldata data) external payable {
        (bool ok, bytes memory returndata) = target.call{value: msg.value}(data);
        if (!ok) {
            if (returndata.length > 0) {
                assembly {
                    let returndata_size := mload(returndata)
                    revert(add(32, returndata), returndata_size)
                }
            } else {
                revert('bad cast call');
            }
        }
    }
}
```

The HomoraCaster provides a secure execution layer by:
- Isolating spell execution from the main bank contract
- Preventing direct spell access to bank state
- Implementing proper error handling and revert propagation

## 3. Debt Tracking Management System

### 3.1 Debt Share Mechanism

The protocol implements a share-based debt tracking system to handle interest accrual efficiently:

```solidity
function borrow(address token, uint amount) external override inExec poke(token) {
    Bank storage bank = banks[token];
    Position storage pos = positions[POSITION_ID];
    uint totalShare = bank.totalShare;
    uint totalDebt = bank.totalDebt;
    uint share = totalShare == 0 ? amount : amount.mul(totalShare).ceilDiv(totalDebt);
    
    bank.totalShare = bank.totalShare.add(share);
    uint newShare = pos.debtShareOf[token].add(share);
    pos.debtShareOf[token] = newShare;
    
    if (newShare > 0) {
        pos.debtMap |= (1 << uint(bank.index));
    }
}
```

#### 3.1.1 Share Calculation

Debt shares are calculated using ceiling division to prevent precision loss:
```solidity
uint share = amount.mul(totalShare).ceilDiv(totalDebt);
```

#### 3.1.2 Debt Balance Computation

Current debt balance is computed as:
```solidity
function borrowBalanceStored(uint positionId, address token) public view returns (uint) {
    uint totalDebt = banks[token].totalDebt;
    uint totalShare = banks[token].totalShare;
    uint share = positions[positionId].debtShareOf[token];
    
    if (share == 0 || totalDebt == 0) {
        return 0;
    } else {
        return share.mul(totalDebt).ceilDiv(totalShare);
    }
}
```

### 3.2 Debt Map Bitmap System

#### 3.2.1 Bitmap Implementation

The debt map is implemented as a 256-bit integer where each bit represents a bank index:

```solidity
uint debtMap; // Bitmap: i^th bit is set iff debt share of i^th bank is nonzero
```

#### 3.2.2 Bit Setting and Clearing

Bits are set when borrowing:
```solidity
if (newShare > 0) {
    pos.debtMap |= (1 << uint(bank.index));
}
```

Bits are cleared when repaying all debt:
```solidity
if (newShare == 0) {
    pos.debtMap &= ~(1 << uint(bank.index));
}
```

#### 3.2.3 Efficient Debt Iteration

The bitmap enables O(k) iteration where k is the number of active debt banks:

```solidity
function getBorrowETHValue(uint positionId) public view returns (uint) {
    uint value = 0;
    Position storage pos = positions[positionId];
    uint bitMap = pos.debtMap;
    uint idx = 0;
    
    while (bitMap > 0) {
        if ((bitMap & 1) != 0) {
            address token = allBanks[idx];
            uint share = pos.debtShareOf[token];
            Bank storage bank = banks[token];
            uint debt = share.mul(bank.totalDebt).ceilDiv(bank.totalShare);
            value = value.add(oracle.asETHBorrow(token, debt, owner));
        }
        idx++;
        bitMap >>= 1;
    }
    return value;
}
```

### 3.3 Interest Accrual Mechanism

#### 3.3.1 Automatic Interest Accrual

Interest is accrued automatically through the `accrue` function:

```solidity
function accrue(address token) public override {
    Bank storage bank = banks[token];
    require(bank.isListed, 'bank not exist');
    uint totalDebt = bank.totalDebt;
    uint debt = ICErc20(bank.cToken).borrowBalanceCurrent(address(this));
    
    if (debt > totalDebt) {
        uint fee = debt.sub(totalDebt).mul(feeBps).div(10000);
        bank.totalDebt = debt;
        bank.reserve = bank.reserve.add(doBorrow(token, fee));
    } else if (totalDebt != debt) {
        bank.totalDebt = debt;
    }
}
```

#### 3.3.2 Fee Collection

The protocol collects fees on interest accrual:
- Fee rate: Configurable basis points (default: 1000 = 10%)
- Fee collection: Automatic through reserve allocation
- Fee distribution: Governed by protocol governance

## 4. Lending Protocol Integration

### 4.1 Multi-Protocol Support

The protocol integrates with multiple lending protocols through a unified interface:

#### 4.1.1 ICErc20 Interface

```solidity
interface ICErc20 {
    function borrow(uint borrowAmount) external returns (uint);
    function repayBorrow(uint repayAmount) external returns (uint);
    function borrowBalanceCurrent(address account) external returns (uint);
    function borrowBalanceStored(address account) external view returns (uint);
}
```

#### 4.1.2 Supported Protocols

Based on deployment scripts and contract analysis:

1. **Compound Protocol**
   - cDAI: 0x92B767185fB3B04F881e3aC8e5B0662a027A1D9f
   - cUSDC: 0x44fbebd2f576670a6c33f6fc0b00aa8c5753b322
   - cUSDT: 0x797AAB1ce7c01eB727ab980762bA88e7133d2157

2. **CREAM Finance (Iron Bank)**
   - Integration through comptroller credit limits
   - Address: 0x6d5a7597896a703fe8c85775b23395a48f971305

3. **Aave Protocol**
   - Mock integration for testing purposes
   - Full integration capability through ICErc20 interface

### 4.2 Bank Addition Process

```solidity
function addBank(address token, address cToken) external onlyGov {
    Bank storage bank = banks[token];
    require(!cTokenInBank[cToken], 'cToken already exists');
    require(!bank.isListed, 'bank already exists');
    require(allBanks.length < 256, 'reach bank limit');
    
    cTokenInBank[cToken] = true;
    bank.isListed = true;
    bank.index = uint8(allBanks.length);
    bank.cToken = cToken;
    
    IERC20(token).safeApprove(cToken, 0);
    IERC20(token).safeApprove(cToken, uint(-1));
    allBanks.push(token);
    
    emit AddBank(token, cToken);
}
```

#### 4.2.1 Bank Limit Constraint

The protocol supports a maximum of 256 banks due to bitmap implementation constraints:
```solidity
require(allBanks.length < 256, 'reach bank limit');
```

## 5. Spell System Architecture

### 5.1 BasicSpell Contract

The BasicSpell provides the foundation for all strategy execution:

```solidity
abstract contract BasicSpell is ERC1155NaiveReceiver {
    IBank public immutable bank;
    IWERC20 public immutable werc20;
    address public immutable weth;
    
    mapping(address => mapping(address => bool)) public approved;
}
```

#### 5.1.1 Core Functions

1. **Asset Transmission**
   ```solidity
   function doTransmit(address token, uint amount) internal {
       if (amount > 0) {
           bank.transmit(token, amount);
       }
   }
   ```

2. **Borrowing Operations**
   ```solidity
   function doBorrow(address token, uint amount) internal {
       if (amount > 0) {
           bank.borrow(token, amount);
       }
   }
   ```

3. **Collateral Management**
   ```solidity
   function doPutCollateral(address token, uint amount) internal {
       if (amount > 0) {
           ensureApprove(token, address(werc20));
           werc20.mint(token, amount);
           bank.putCollateral(address(werc20), uint(token), amount);
       }
   }
   ```

### 5.2 Protocol-Specific Spells

#### 5.2.1 UniswapV2SpellV1

Implements Uniswap V2 liquidity provision strategies:

```solidity
contract UniswapV2SpellV1 is WhitelistSpell {
    IUniswapV2Factory public immutable factory;
    IUniswapV2Router02 public immutable router;
    mapping(address => mapping(address => address)) public pairs;
}
```

Key features:
- Optimal deposit calculation for balanced liquidity provision
- LP token wrapping and collateralization
- Staking rewards integration through WStakingRewards

#### 5.2.2 CurveSpellV1

Implements Curve Finance strategies:

```solidity
contract CurveSpellV1 is WhitelistSpell {
    ICurveRegistry public immutable registry;
    mapping(address => mapping(uint => bool)) public registeredPools;
}
```

Features:
- Multi-token pool support (2-4 tokens)
- Automatic gauge staking through WLiquidityGauge
- CRV token reward harvesting

#### 5.2.3 BalancerSpellV1

Implements Balancer strategies:

```solidity
contract BalancerSpellV1 is WhitelistSpell {
    IBalancerPool public immutable pool;
    mapping(address => bool) public approvedTokens;
}
```

Features:
- Weighted pool liquidity provision
- BAL token reward harvesting
- Flexible token weight management

### 5.3 Spell Execution Flow

1. **User calls HomoraBank.execute()**
2. **HomoraBank creates/updates position**
3. **HomoraCaster forwards call to spell**
4. **Spell executes strategy using bank functions**
5. **Position health validation**
6. **Execution completion**

## 6. Oracle Infrastructure

### 6.1 CoreOracle Contract

The CoreOracle provides routing to specific price sources:

```solidity
contract CoreOracle is IBaseOracle, Governable {
    mapping(address => address) public routes;
    
    function getETHPx(address token) external view override returns (uint) {
        uint px = IBaseOracle(routes[token]).getETHPx(token);
        require(px != 0, 'price oracle failure');
        return px;
    }
}
```

### 6.2 ProxyOracle Contract

The ProxyOracle implements risk parameters and token factors:

```solidity
contract ProxyOracle is IOracle, Governable {
    struct TokenFactors {
        uint16 borrowFactor;      // Borrow factor (≥10000 = 100%)
        uint16 collateralFactor;  // Collateral factor (≤10000 = 100%)
        uint16 liqIncentive;      // Liquidation incentive (10000-20000 = 100%-200%)
    }
    
    IBaseOracle public immutable source;
    mapping(address => TokenFactors) public tokenFactors;
    mapping(address => bool) public whitelistERC1155;
}
```

#### 6.2.1 Risk Parameter Validation

```solidity
function setTokenFactors(address[] memory tokens, TokenFactors[] memory _tokenFactors) external onlyGov {
    require(_tokenFactors[idx].borrowFactor >= 10000, 'borrow factor must be at least 100%');
    require(_tokenFactors[idx].collateralFactor <= 10000, 'collateral factor must be at most 100%');
    require(_tokenFactors[idx].liqIncentive >= 10000, 'incentive must be at least 100%');
    require(_tokenFactors[idx].liqIncentive <= 20000, 'incentive must be at most 200%');
}
```

### 6.3 Oracle Sources

#### 6.3.1 SimpleOracle

Basic price oracle with manual price setting:
```solidity
contract SimpleOracle is IBaseOracle {
    mapping(address => uint) public prices;
    
    function setETHPx(address[] calldata tokens, uint[] calldata prices) external onlyGov {
        for (uint idx = 0; idx < tokens.length; idx++) {
            prices[tokens[idx]] = prices[idx];
        }
    }
}
```

#### 6.3.2 UniswapV2Oracle

Uniswap V2 pool-based price oracle:
```solidity
contract UniswapV2Oracle is IBaseOracle {
    function getETHPx(address token) external view override returns (uint) {
        // Implementation uses Uniswap V2 reserves for price calculation
    }
}
```

#### 6.3.3 CurveOracle

Curve pool-based price oracle:
```solidity
contract CurveOracle is IBaseOracle {
    ICurveRegistry public immutable registry;
    
    function getETHPx(address token) external view override returns (uint) {
        // Implementation uses Curve pool reserves and virtual price
    }
}
```

## 7. Wrapper Contract System

### 7.1 WERC20 Contract

The WERC20 contract provides ERC1155 tokenization for ERC20 tokens:

```solidity
contract WERC20 is ERC1155('WERC20'), ReentrancyGuard, IWERC20 {
    function mint(address token, uint amount) external override nonReentrant {
        uint balanceBefore = IERC20(token).balanceOf(address(this));
        IERC20(token).safeTransferFrom(msg.sender, address(this), amount);
        uint balanceAfter = IERC20(token).balanceOf(address(this));
        _mint(msg.sender, uint(token), balanceAfter.sub(balanceBefore), '');
    }
    
    function burn(address token, uint amount) external override nonReentrant {
        _burn(msg.sender, uint(token), amount);
        IERC20(token).safeTransfer(msg.sender, amount);
    }
}
```

#### 7.1.1 Token ID Mapping

Token IDs are derived from token addresses:
```solidity
function getUnderlyingToken(uint id) external view override returns (address) {
    address token = address(id);
    require(uint(token) == id, 'id overflow');
    return token;
}
```

### 7.2 WLiquidityGauge Contract

Wraps Curve liquidity gauge positions:

```solidity
contract WLiquidityGauge is ERC1155('WLiquidityGauge'), ReentrancyGuard, IWLiquidityGauge {
    ILiquidityGauge public immutable gauge;
    IERC20 public immutable lpToken;
    IERC20 public immutable crv;
}
```

Features:
- LP token staking in Curve gauges
- CRV token reward accumulation
- ERC1155 representation of staked positions

### 7.3 WMasterChef Contract

Wraps Sushiswap masterchef staking:

```solidity
contract WMasterChef is ERC1155('WMasterChef'), ReentrancyGuard, IWMasterChef {
    IMasterChef public immutable masterchef;
    IERC20 public immutable lpToken;
    IERC20 public immutable sushi;
}
```

Features:
- LP token staking in Sushiswap masterchef
- SUSHI token reward accumulation
- Flexible staking and unstaking

### 7.4 WStakingRewards Contract

Wraps general staking reward contracts:

```solidity
contract WStakingRewards is ERC1155('WStakingRewards'), ReentrancyGuard, IWStakingRewards {
    IStakingRewards public immutable stakingRewards;
    IERC20 public immutable stakingToken;
    IERC20 public immutable rewardsToken;
}
```

Features:
- Generic staking reward integration
- Multiple reward token support
- Standardized ERC1155 interface

## 8. Mathematical Foundations

### 8.1 HomoraMath Library

The HomoraMath library provides essential mathematical functions:

```solidity
library HomoraMath {
    function divCeil(uint lhs, uint rhs) internal pure returns (uint) {
        return lhs.add(rhs).sub(1) / rhs;
    }
    
    function fmul(uint lhs, uint rhs) internal pure returns (uint) {
        return lhs.mul(rhs) / (2**112);
    }
    
    function fdiv(uint lhs, uint rhs) internal pure returns (uint) {
        return lhs.mul(2**112) / rhs;
    }
    
    function sqrt(uint x) internal pure returns (uint) {
        // Babylonian method implementation
    }
}
```

#### 8.1.1 Ceiling Division

Ceiling division prevents precision loss in debt calculations:
```solidity
function ceilDiv(uint a, uint b) internal pure returns (uint) {
    return a.add(b).sub(1).div(b);
}
```

#### 8.1.2 Fixed-Point Arithmetic

Fixed-point operations for price calculations:
- `fmul`: Fixed-point multiplication (÷2^112)
- `fdiv`: Fixed-point division (×2^112)

#### 8.1.3 Square Root Implementation

Efficient square root calculation using Babylonian method:
```solidity
function sqrt(uint x) internal pure returns (uint) {
    if (x == 0) return 0;
    uint xx = x;
    uint r = 1;
    
    // Bit manipulation for initial approximation
    if (xx >= 0x100000000000000000000000000000000) {
        xx >>= 128;
        r <<= 64;
    }
    // ... additional bit checks
    
    // Seven iterations of Babylonian method
    r = (r + x / r) >> 1;
    // ... repeated iterations
    
    uint r1 = x / r;
    return (r < r1 ? r : r1);
}
```

### 8.2 Optimal Deposit Algorithm

The protocol implements sophisticated algorithms for optimal liquidity provision:

```solidity
function optimalDeposit(uint amtA, uint amtB, uint resA, uint resB) internal pure returns (uint swapAmt, bool isReversed) {
    if (amtA.mul(resB) >= amtB.mul(resA)) {
        swapAmt = _optimalDepositA(amtA, amtB, resA, resB);
        isReversed = false;
    } else {
        swapAmt = _optimalDepositA(amtB, amtA, resB, resA);
        isReversed = true;
    }
}
```

#### 8.2.1 Mathematical Formula

The optimal deposit calculation uses the formula:
```
swapAmt = (√(b² + 4ac) - b) / (2a)
```

Where:
- `a = 997`
- `b = 1997 × resA`
- `c = ((amtA × resB) - (amtB × resA)) × 1000 × resA / (amtB + resB)`

## 9. Security Mechanisms

### 9.1 Reentrancy Protection

#### 9.1.1 General Lock

```solidity
modifier lock() {
    require(_GENERAL_LOCK == _NOT_ENTERED, 'general lock');
    _GENERAL_LOCK = _ENTERED;
    _;
    _GENERAL_LOCK = _NOT_ENTERED;
}
```

#### 9.1.2 Execution Lock

```solidity
modifier inExec() {
    require(POSITION_ID != _NO_ID, 'not within execution');
    require(SPELL == msg.sender, 'not from spell');
    require(_IN_EXEC_LOCK == _NOT_ENTERED, 'in exec lock');
    _IN_EXEC_LOCK = _ENTERED;
    _;
    _IN_EXEC_LOCK = _NOT_ENTERED;
}
```

### 9.2 Access Control

#### 9.2.1 Governance Controls

```solidity
modifier onlyGov() {
    require(msg.sender == governor, 'not governor');
    _;
}
```

#### 9.2.2 Whitelisting System

```solidity
mapping(address => bool) public whitelistedTokens;
mapping(address => bool) public whitelistedSpells;
mapping(address => bool) public whitelistedUsers;
```

### 9.3 Position Health Validation

```solidity
function execute(uint positionId, address spell, bytes memory data) external payable lock onlyEOAEx returns (uint) {
    // ... execution logic ...
    
    uint collateralValue = getCollateralETHValue(positionId);
    uint borrowValue = getBorrowETHValue(positionId);
    require(collateralValue >= borrowValue, 'insufficient collateral');
    
    // ... completion logic ...
}
```

### 9.4 Liquidation Mechanism

```solidity
function liquidate(uint positionId, address debtToken, uint amountCall) external override lock poke(debtToken) {
    uint collateralValue = getCollateralETHValue(positionId);
    uint borrowValue = getBorrowETHValue(positionId);
    require(collateralValue < borrowValue, 'position still healthy');
    
    Position storage pos = positions[positionId];
    (uint amountPaid, uint share) = repayInternal(positionId, debtToken, amountCall);
    
    uint bounty = Math.min(
        oracle.convertForLiquidation(debtToken, pos.collToken, pos.collId, amountPaid),
        pos.collateralSize
    );
    
    pos.collateralSize = pos.collateralSize.sub(bounty);
    IERC1155(pos.collToken).safeTransferFrom(address(this), msg.sender, pos.collId, bounty, '');
}
```

## 10. Deployment and Testing Infrastructure

### 10.1 Main Deployment Script

The main deployment script (`deploy_v2.py`) orchestrates the complete protocol setup:

```python
def main():
    # Deploy core contracts
    homora = HomoraBank.deploy({'from': admin})
    homora.initialize(oracle, 1000, {'from': admin})  # 10% fee
    
    # Deploy spells
    uniswap_spell = UniswapV2SpellV1.deploy(homora, werc20, router, {'from': admin})
    curve_spell = CurveSpellV1.deploy(homora, werc20, registry, {'from': admin})
    
    # Deploy oracles
    core_oracle = CoreOracle.deploy({'from': admin})
    oracle = ProxyOracle.deploy(core_oracle, {'from': admin})
    
    # Configure oracles
    oracle.setWhitelistERC1155([werc20, wgauge], True, {'from': admin})
    core_oracle.setRoute([dai, usdc, usdt], [simple_oracle, simple_oracle, simple_oracle])
    
    # Add banks
    homora.addBank(dai, crdai, {'from': admin})
    homora.addBank(usdc, crusdc, {'from': admin})
    homora.addBank(usdt, crusdt, {'from': admin})
```

### 10.2 Test Infrastructure

#### 10.2.1 Uniswap Spell Testing

```python
def test_uniswap_spell(uniswap_spell, homora, oracle):
    # Test position opening
    homora.execute(
        0,
        uniswap_spell,
        uniswap_spell.addLiquidityWERC20.encode_input(
            dai, weth,
            [10**18, 10**18, 0, 10**18, 10**18, 0, 0, 0]
        ),
        {'from': alice}
    )
    
    # Test position closing
    homora.execute(
        position_id - 1,
        uniswap_spell,
        uniswap_spell.removeLiquidityWERC20.encode_input(
            dai, weth,
            [2**256-1, 0, 2**256-1, 2**256-1, 0, 0, 0]
        ),
        {'from': alice}
    )
```

#### 10.2.2 Curve Spell Testing

```python
def test_curve_spell(curve_spell, homora, oracle):
    # Test 3-token pool
    homora.execute(
        0,
        curve_spell,
        curve_spell.addLiquidity3.encode_input(
            lp, [10**18, 10**6, 10**6], 0, 0, 0, 0, 0, 0
        ),
        {'from': alice}
    )
```

### 10.3 Network Configuration

#### 10.3.1 Mainnet Addresses

```python
# Mainnet contract addresses
KP3R_NETWORK = '0x73353801921417F465377c8d898c6f4C0270282C'
CRV_REGISTRY = '0x7D86446dDb609eD0F5f8684AcF30380a356b2B4c'
CRV_TOKEN = '0xD533a949740bb3306d119CC777fa900bA034cd52'
MASTERCHEF = '0xc2EdaD668740f1aA35E4D8f227fB8E17dcA888Cd'
BANK = '0x5f5Cd91070960D13ee549C9CC47e7a4Cd00457bb'
```

#### 10.3.2 Testnet Configuration

Test scripts use mock contracts and forked mainnet state for comprehensive testing.

## 11. Technical Specifications

### 11.1 Solidity Version

- **Protocol Version**: Solidity 0.6.12
- **OpenZeppelin**: 3.4.0
- **Experimental Features**: ABIEncoderV2

### 11.2 Gas Optimization

#### 11.2.1 Bitmap Implementation

- **Bank Limit**: 256 banks maximum
- **Iteration Complexity**: O(k) where k = active debt banks
- **Storage Efficiency**: Single uint256 for debt tracking

#### 11.2.2 Share-Based Calculations

- **Precision**: Ceiling division prevents dust amounts
- **Gas Cost**: Minimal computation for debt updates
- **Scalability**: Efficient handling of large numbers

### 11.3 Integration Patterns

#### 11.3.1 Lending Protocol Integration

- **Interface Standardization**: ICErc20 interface
- **Multi-Protocol Support**: Compound, CREAM, Aave
- **Extensibility**: Easy addition of new protocols

#### 11.3.2 DeFi Protocol Integration

- **Spell System**: Modular strategy execution
- **Wrapper Contracts**: Standardized token representation
- **Oracle Integration**: Multi-source price feeds

### 11.4 Risk Management

#### 11.4.1 Collateral Requirements

- **Collateral Factors**: Configurable per token (≤100%)
- **Borrow Factors**: Configurable per token (≥100%)
- **Liquidation Incentives**: 100%-200% range

#### 11.4.2 Position Monitoring

- **Real-time Health**: Oracle-based valuation
- **Automatic Liquidation**: Permissionless liquidation
- **Risk Mitigation**: Multiple safety checks

## 12. References

### 12.1 Smart Contract Addresses

- **HomoraBank**: Deployed per network
- **Compound cTokens**: Mainnet addresses in deployment scripts
- **CREAM Finance**: 0x6d5a7597896a703fe8c85775b23395a48f971305

### 12.2 Technical Documentation

- **Protocol Design**: https://blog.alphafinance.io/byot/
- **Mathematical Formulas**: Optimal deposit algorithms
- **Security Audits**: PeckShield, Quantstamp audit reports

### 12.3 Code References

- **Repository**: Alpha Finance Labs GitHub
- **Solidity Version**: 0.6.12
- **OpenZeppelin**: 3.4.0 contracts

### 12.4 Related Protocols

- **Compound**: Lending protocol integration
- **CREAM Finance**: Iron Bank integration
- **Aave**: Lending protocol integration
- **Uniswap V2**: DEX integration
- **Curve Finance**: Stablecoin DEX integration
- **Balancer**: Weighted pool integration

---

**Document Version**: 1.0  
**Last Updated**: December 2024  
**Research Scope**: Complete codebase analysis  
**Technical Depth**: Advanced smart contract engineering  
**Audience**: DeFi researchers, smart contract developers, protocol architects
