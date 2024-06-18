# **HATs Arbitration Contracts Audit Competition on Hats.finance** 


## Introduction to Hats.finance


Hats.finance builds autonomous security infrastructure for integration with major DeFi protocols to secure users' assets. 
It aims to be the decentralized choice for Web3 security, offering proactive security mechanisms like decentralized audit competitions and bug bounties. 
The protocol facilitates audit competitions to quickly secure smart contracts by having auditors compete, thereby reducing auditing costs and accelerating submissions. 
This aligns with their mission of fostering a robust, secure, and scalable Web3 ecosystem through decentralized security solutions​.

## About Hats Audit Competition


Hats Audit Competitions offer a unique and decentralized approach to enhancing the security of web3 projects. Leveraging the large collective expertise of hundreds of skilled auditors, these competitions foster a proactive bug hunting environment to fortify projects before their launch. Unlike traditional security assessments, Hats Audit Competitions operate on a time-based and results-driven model, ensuring that only successful auditors are rewarded for their contributions. This pay-for-results ethos not only allocates budgets more efficiently by paying exclusively for identified vulnerabilities but also retains funds if no issues are discovered. With a streamlined evaluation process, Hats prioritizes quality over quantity by rewarding the first submitter of a vulnerability, thus eliminating duplicate efforts and attracting top talent in web3 auditing. The process embodies Hats Finance's commitment to reducing fees, maintaining project control, and promoting high-quality security assessments, setting a new standard for decentralized security in the web3 space​​.

## HATs Arbitration Contracts Overview

We turn black hat hackers into white hat hackers using the right incentives. Creating the future of security on _EVM

## Competition Details


- Type: A public audit competition hosted by HATs Arbitration Contracts
- Duration: 2 weeks
- Maximum Reward: $28,000
- Submissions: 74
- Total Payout: $24,900.4 distributed among 8 participants.

## Scope of Audit

The scope of the competition includes all the contracts that make up the HATs system, but we are specifically interested in the changes made since v2.0.

There are a number of changes since our last release:

- We added an arbitration procedure: if there is any kind of dispute about the payout, the decision can be  escalated to an Expert Committee, and if that committee cannot find consensus, to the Kleros court. The intended behavior of that system is documented here: https://github.com/hats-finance/hats-contracts/blob/audit/docs/claims.md
- The functionality of the main contract, HATVault.sol, was split into two new files (HATVault.sol and HATClaimsManager.sol) 
- We added some helper contracts, such as the PaymentSplitter.sol

The contracts that are currently in use, v2.0, can be found here https://github.com/hats-finance/hats-contracts/tree/v2.0.  


In this competition, we are interested in the usual programming errors, but we are also interested in any scenarios in which the arbitration procedure can be abused or hijacked. (For such attacks, it is the behavior that is implemented that serves as a reference - pointing out mistakes in documentation is appreciated, but will not be rewarded). 





## High severity issues


- **Vulnerability in HATArbitrator Contract Allows Draining All User Funds**

  An attacker can exploit a vulnerability in the `HATArbitrator` contract to drain all user funds. By repeatedly invoking the `refundExpiredSubmitClaimRequest()` function with a non-existent claim, an attacker can transfer the contract's funds to the zero address, effectively burning them. This flaw arises due to several oversight issues in the function:

1. The function allows calls with non-existent `submitClaimRequest` values.
2. The if-statement checking the claim submission period can be bypassed with non-existing claims, since `submitClaimRequest.submittedAt` will always be zero.
3. There is no verification that the submitter’s address is non-zero.
4. The transfer amount is hardcoded instead of using the actual bond amount associated with a valid claim.
5. The function can be called by anyone, not just the original submitter.
6. Popular ERC20 tokens, such as USDT, DAI, and WETH, allow transfers to the zero address.

Proof of concept demonstrates how to set up and execute the vulnerability using a live ERC20 token like DAI to drain the contract incrementally. 

To mitigate this issue, it’s recommended to validate the `SubmitClaimRequest` by checking that the `submittedAt`, `bond`, and `submitter` fields are non-zero. The transfer amount should be based on the `submitClaimRequest.bond` field instead of a hardcoded value. Additionally, restrict access to the `refundExpiredSubmitClaimRequest()` function, allowing only the original submitter to invoke it.


  **Link**: [Issue #36](https://github.com/hats-finance/HATs-Arbitration-Contracts-0x79a618f675857b45934ca1c413fd5f409cf89735/issues/36)


- **"Fix Needed for reclaimBond() to Prevent Dispute Bond Refund Exploit"**

  The function `HATArbitrator.reclaimBond()` erroneously allows refunds for previously dismissed claims, leading to a potential vulnerability where malicious actors can exploit the system. Specifically, an attacker can make numerous spam or malicious disputes and subsequently get these refunded using the bonds from successive claims. This manipulation is at the expense of the Expert committee or future disputers.

In a scenario illustrating this issue: a claim (Claim A) is submitted and disputed. The dispute is dismissed, and the bond is transferred to the Expert Committee. When a new claim (Claim B) is then submitted and disputed, the attacker reclaims the bond originally used for Claim A. This unauthorized action depletes the bond pool, eventually leading to failed transfers when the Expert Committee dismisses the dispute over Claim B, as the bond amount becomes insufficient.

To mitigate this, it is recommended to adjust the `reclaimBond()` function to subtract reclaimed values from `totalBondsOnClaim[_vault][_claimId]`, preventing any unauthorized reclaiming of bonds post-dispute dismissal. Similar modifications should be made to the `_confiscateDisputers()` function for consistency. Additionally, in the `dismissDispute()` function, after transferring the bond to the Expert Committee, the `totalBondsOnClaim[_vault][_claimId]` should be reset to zero, ensuring the accurate tracking and avoidance of bond depletion.


  **Link**: [Issue #37](https://github.com/hats-finance/HATs-Arbitration-Contracts-0x79a618f675857b45934ca1c413fd5f409cf89735/issues/37)

## Medium severity issues


- **Issue with `TokenLock.revoke()` Potentially Permanently Locking `releasableAmount` and `surplusAmount`**

  In the `TokenLock.revoke()` function, unvested tokens are sent to the owner, and the contract is subsequently destroyed. However, the function only accounts for unvested tokens, ignoring other tokens within `TokenLock`, such as vested tokens (not yet released by the beneficiary) and surplus amounts. Since these additional tokens remain within the contract after revocation and its destruction, they become permanently inaccessible. It is recommended to transfer the combined amount of `releasableAmount()` and `surplusAmount()` to either the owner or the beneficiary before destructing the contract. This can prevent any tokens from being forever trapped within the contract. Detailed code adjustments are provided to facilitate this change.


  **Link**: [Issue #54](https://github.com/hats-finance/HATs-Arbitration-Contracts-0x79a618f675857b45934ca1c413fd5f409cf89735/issues/54)

## Low severity issues


- **Using Deprecated Transfer Function Causes Transaction Failures and Potential Fund Loss**

  Using the deprecated transfer and send functions can cause transaction failures, especially with multisig wallets requiring more than 2300 gas. These failures can result in fund loss due to the lack of additional checks. It is recommended to use the call() function to send funds to avoid these issues. Affected contracts include HatKlerosConnector and HatKlerosConnectorV2.


  **Link**: [Issue #16](https://github.com/hats-finance/HATs-Arbitration-Contracts-0x79a618f675857b45934ca1c413fd5f409cf89735/issues/16)


- **Function to Predict Payment Splitter Address Lacks Input Validation Causing Funds Loss**

  The `predictSplitterAddress` function in `HATPaymentSplitterFactory` may predict addresses of `HATPaymentSplitter` that cannot be created due to input validation. If users send funds to these addresses before the payment splitter is deployed, the funds will be lost. Adding input validation to `predictSplitterAddress` is recommended to prevent this issue.


  **Link**: [Issue #48](https://github.com/hats-finance/HATs-Arbitration-Contracts-0x79a618f675857b45934ca1c413fd5f409cf89735/issues/48)



## Conclusion

**Conclusion:**

The audit competition on Hats.finance revealed critical vulnerabilities and inefficiencies within the HATs arbitration contracts, emphasizing the need for comprehensive improvements. The most severe issue involved a vulnerability in the HATArbitrator contract, allowing attackers to drain user funds through the `refundExpiredSubmitClaimRequest()` function. Another high-severity flaw pertained to the `reclaimBond()` function, which permitted the exploitation of the dispute bond refund system, enabling unauthorized fund reclamation. Medium-severity risks included the `TokenLock.revoke()` function potentially locking substantial token amounts permanently. Low-severity issues highlighted the use of deprecated transfer functions causing transaction failures and potential funds loss, and a lack of input validation in the `predictSplitterAddress` function leading to possible fund misallocation. Addressing these vulnerabilities with proper validation, restrictive access, and updated transfer methods will significantly enhance the security and efficiency of the HATs Arbitration Contracts, aligning with Hats.finance’s mission to establish a robust decentralized security infrastructure in Web3.

## Disclaimer


This report does not assert that the audited contracts are completely secure. Continuous review and comprehensive testing are advised before deploying critical smart contracts.


The HATs Arbitration Contracts audit competition illustrates the collaborative effort in identifying and rectifying potential vulnerabilities, enhancing the overall security and functionality of the platform.


Hats.finance does not provide any guarantee or warranty regarding the security of this project. Smart contract software should be used at the sole risk and responsibility of users.

