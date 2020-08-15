# FlashSwap Paper Draft v1.0

## Notation

This is a preliminary paper giving an overview of the mathematical underpinnings of
flashswap.

We use a notation similar to Aave but with code in some places instead of formulas.

- **T** : current timestamp as in ```block.timestamp```
- **T(l)** : timestamp at last update
- **DeltaT** : **T** - **T(l)**
- **T** : number of seconds in a year : 31536000
- **Lt** : total liquidity
- **Bs** : total borrowed at stable rate
- **Bv** : total borrowed at variable rate
- **Bt** : total borrowed = **Bs** + **Bv**
- **U** : utilisation rate = **Bt**/**Lt**
- **U(opt)** : optimal utilisation rate targeted by the model
- **R(v0)** : base borrow rate
- **R(s1)** : interest rate slope lowerbound  (**U** < **U(opt)**)
- **R(s2)** : interest rate upperbound (**U** > **U(opt)**)
- **R(v)** : variable borrow rate
- **R(s)** : stable borrow rate
- **Mavg(r)** : average market lending rate
- **Rs(t)** : average stable borrow rate at time *t*
- **R(o)** : overall borrow rate
- **R(l)** :current liquidity rate
- **C(t,i)** : cumulated    liquidity index
- **B(t,vc)** : cumulated variable borrow index
- **B(x)** : user's balance
- **B(x,c)** : user's compound balance
- **H(f)** : health factor

### Functional Definitions

```python
# Rv
def variable_borrow_rate(U,Uopt):
    if U < Uopt :
        return Rv0 + (U/Uopt) * Rs1
    else :
        return Rv0 + Rs1 + (U-Uopt) / (1-Uopt) * Rs2

# Mavg
 def average_market_rate(lending_rates,borrow_rates):
    return sum([x*y for (x,y) in zip(lending_rates,borrow_rates)]) / sum(borrow_rates)

def average_borrow_rate(Bs,Rs,new_borrow_rate,rate):
    return (Bs * average_borrow_rate(old_borrow_rate,rate)) + (new_borrow_rate*Rs)/(Bs + new_borrow)

def current_borrow_rate(Bt,Bv,Bs,Ravg):
    if Bt == 0 :
        return 0
    else :
        return (Bv*Rv + Bs * Ravg) / Bt

def current_liquidity_rate(Bt,Bv,Bs,Ravg,U):
    return current_borrow_rate(Bt,Bv,Bs,Ravg) / U

def cumulated_variable_borrow(Rv,T,D(t)):
    return (1 + Rv/T)**D(t) * cumulated_variable_borrow(Rv,T,(D(t-1)))

def stable_user_borrow_balance(Rv,Bx,D(t)):
    return (1 + Rv / T)**D(t) * Bx

def variable_user_borrow_balance(Bvcx,Bvc,Rv,Bx,D(t)):
    return (Bvc / Bvcx) * stable_user_borrow_balance(Rv,Bx,D(t))

def health_factor(totalCollateral,Bt,LQ,totalFees):
    return (totalCollateral * LQ) / (Bt + totalFees)

 ```

## Overview

FlashSwap is a pool based algorithmic lending system, interest rates and rewards
for borrowers and lenders are decided given the current pool state.

For borrowers, it depends on the basis cost of current pooled assets at any time.
As funds are borrowed, the pool's total liquidity decreases and interests rates reciprocally
increase.

For lenders, their earnings are decided based on the interest rates and a margin of security
to allow withdrawals at anytime.

The more liquidity in the pool the lower interests are and at the same time the higher earnings
are, this builds a complex game of incentives where the goal is to essentially stabilize rates
on a long enough timeline.

## On Chain Banking

FlashSwap liquidity pools and algorithmic interest rates represent a move from TradFi model of rent-seeking
to one where protocol incentivies are the main drivers of the economy.

FlashSwap implements synthetic tokenized asset that natively accrue fees. On a deposit lenders
receive a 1:1 fToken equivalent of their deposit.

fToken's balance for each lenders grows with time, driven by perpetual acrual of deposit interest.
fToken's exchange rate is taken in consideration of the borrow/liquidity.

Once the underlying asset is redeemed the corresponding fTokens are burned.

## On Chain Leveraged Trading


Flash loans are considered fixed rate loans, the period of a loan is stable (1 block) therefore for loans
that will be repaid on the same block the funds are held by a contract that satisfies the interface of ```IFlashFlashLoan``` meaning that flash loans are accessible to smart contracts only.

Essentially, users can algorithmically trade on Uniswap, 1.inch... using leveraged positions but at the same time pay no collateral.

## Interest Rate Mechanics

The interest rate contracts calculates and updates interest rates for each reserve.
Given the estimated interest rate slop below and above optimal utilisation, the current
variable rate borrow rate is defined as by the function ```variable_borrow_rate(U,Uopt)```

This model allows a simple linear update of interest rates, if optimal utilisation is 0 the
variable interest rate is equal to the base variable rate. If the current utilisation rate
is optimal the new variable rate is updated by the interest rate slope, above optimal levels
interest rates raise sharply to account for capital costs.

Using a fixed rate model on top of a pool is doomed to fail.  Because,  fixed rates are hard to handle algorithmically, given that borrow costs depend on market conditions and the available liquidity.
Volatility, dried liquidity...  all play a goal is defining interest ratessince they are time and economical constraints. If a loan has a stable duration, it should survive extreme market conditions, as the borrower must repay at the end of the loan period.

## Governance

The rights of the protocol are controlled by the FSWP token.  Initially, the FlashSwap Protocol will be launched with a on-chain governance based on a Uniswap Liquidity Offering.

There are two levels of governance FSWP level which concerns protocol evolution and fToken level
which concerns pool decision.

The protocol governance is separate from the pool governance, protocol governance is controlled
by FSWP tokens which are essentially protocol shares.
Pool governance is controlled by the pool's liquidity providers by their weighted average of liquidity
in fTokens.
