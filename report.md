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


- **Multiple Vulnerabilities in `refundExpiredSubmitClaimRequest` Allow Draining of `HATArbitrator` Funds**

  An attacker can exploit the `HATArbitrator` contract by calling the `refundExpiredSubmitClaimRequest()` function with a non-existent claim. This would allow the attacker to repeatedly drain the contract's funds. Key issues contributing to this vulnerability include:

1. The function does not validate whether the `_internalClaimId` corresponds to a genuine `submitClaimRequest`.
2. The function's if statement can be bypassed because a non-existent claim's `submittedAt` defaults to zero.
3. There is no check to ensure that the `submitClaimRequest.submitter` is a non-zero address.
4. The transfer amount is hardcoded instead of referenced from the actual submitted claim.
5. Anyone can call `refundExpiredSubmitClaimRequest()`, regardless of whether they submitted a claim request.

Additionally, several popular ERC20 tokens like USDT, DAI, and WETH allow transfers to the zero address, contributing further to the issue.

To resolve this, it is recommended to:
- Validate that `SubmitClaimRequest` fields (`submittedAt`, `bond`, `submitter`) are non-zero.
- Use `submitClaimRequest.bond` instead of a hard-coded `bondsNeededToStartDispute`.
- Restrict function calls to only the original submitter of the claim.

These mitigations could prevent unauthorized users from draining the contract and ensure the function operates as intended.

Here’s an example of the function with the recommended changes applied:
```solidity
	error InvalidSubmitClaimRequest();
	error OnlySubmitterCanCall();

	function refundExpiredSubmitClaimRequest(bytes32 _internalClaimId) external {
		SubmitClaimRequest memory submitClaimRequest = submitClaimRequests[_internalClaimId];

		if (submitClaimRequest.submitter == address(0) || submitClaimRequest.submittedAt == 0 || submitClaimRequest.bond == 0) {
			revert InvalidSubmitClaimRequest();
		}

		if (submitClaimRequest.submitter != msg.sender) {
			revert OnlySubmitterCanCall();
		}

		if (block.timestamp <= submitClaimRequest.submittedAt + submitClaimRequestReviewPeriod) {
			revert ClaimReviewPeriodDidNotEnd();
		}

		delete submitClaimRequests[_internalClaimId];
		token.safeTransfer(submitClaimRequest.submitter, submitClaimRequest.bond);

		emit SubmitClaimRequestExpired(_internalClaimId);
	}
```


  **Link**: [Issue #36](https://github.com/hats-finance/HATs-Arbitration-Contracts-0x79a618f675857b45934ca1c413fd5f409cf89735/issues/36)


- **`HATArbitrator.reclaimBond() vulnerability allows claim bond refund abuse and insufficient balance transfer`**

  The `HATArbitrator.reclaimBond()` function mistakenly refunds previously dismissed claims, which should not occur. This loophole permits an attacker to exploit the system by creating spam or malicious disputes and subsequently recouping those bonds using successive claims, thereby depleting the bonds allocated to the Expert Committee or future disputers.

For example, once Claim A is dismissed and its bonds transferred to the Expert Committee, a new Claim B can be disputed, providing an opportunity for the malicious user to reclaim the previously used bonds from Claim A. This results in an insufficient balance in the `HATArbitrator` contract when the dispute over Claim B is eventually dismissed.

To fix this, it's recommended to adjust the `reclaimBond()` method so that reclaimed amounts are subtracted from the `totalBondsOnClaim` for the given claim. Additionally, similar changes should be applied to the `_confiscateDisputers()` function to maintain consistency in bond tracking. Also, updating the `dismissDispute()` function to reset `totalBondsOnClaim` to zero after transferring bonds to the Expert Committee ensures proper state management, preventing unauthorized reclaiming of bonds.


  **Link**: [Issue #37](https://github.com/hats-finance/HATs-Arbitration-Contracts-0x79a618f675857b45934ca1c413fd5f409cf89735/issues/37)

## Medium severity issues


- **Unvested and unreleased tokens stuck in TokenLock after contract revocation**

  In the `TokenLock.revoke()` function, unvested tokens are sent to the owner and then the contract is destroyed. However, the function only accounts for unvested tokens, ignoring vested tokens that haven't been released and any surplus amount. This means that the combined value of `releasableAmount()` and `surplusAmount()` will be permanently stuck in the `TokenLock` once the contract is destroyed. To avoid this, it is recommended to return these amounts to either the owner or the beneficiary before the contract is terminated. Two potential solutions are proposed, one returning the tokens to the owner and the other to the beneficiary.


  **Link**: [Issue #54](https://github.com/hats-finance/HATs-Arbitration-Contracts-0x79a618f675857b45934ca1c413fd5f409cf89735/issues/54)

## Low severity issues


- **Deprecated Transfer Function Causes Transaction Failures for Specific Wallets**

  Using the deprecated `transfer` function may cause transaction failures, particularly with multisig wallets requiring more than 2300 gas to receive funds. The `send()` function also shares this issue and lacks checks for fund receipt, leading to potential fund loss. It is recommended to use the `call()` function instead. 

Affected Lines:
1. HatKlerosConnector: #168 and #322
2. HatKlerosConnectorV2: #100


  **Link**: [Issue #16](https://github.com/hats-finance/HATs-Arbitration-Contracts-0x79a618f675857b45934ca1c413fd5f409cf89735/issues/16)


- **`HATPaymentSplitterFactory.predictSplitterAddress Could Result in Lost Funds without Input Validation`**

  The `predictSplitterAddress` function from `HATPaymentSplitterFactory` uses `Clones.predictDeterministicAddress` to predict deployment addresses. However, `HATPaymentSplitter.__PaymentSplitter_init` has input validation that may prevent some payment splitters from being created. This can lead to loss of funds if sent to these predicted addresses before deployment. Adding input validation to `predictSplitterAddress` is recommended to prevent this issue.


  **Link**: [Issue #48](https://github.com/hats-finance/HATs-Arbitration-Contracts-0x79a618f675857b45934ca1c413fd5f409cf89735/issues/48)



## Conclusion

The HATs Arbitration Contracts Audit Competition on Hats.finance employed decentralized audit competitions to enhance the security of Web3 projects by engaging numerous auditors in a results-driven environment. Participants competed over a two-week period, with a maximum reward of $28,000, resulting in a platform payout of $24,900.4 to 8 contributors.

Several contracts were audited, with an emphasis on changes since version 2.0, particularly the newly added arbitration procedure aimed at resolving disputes via an Expert Committee or Kleros court. Critical findings included high severity issues like vulnerabilities in the `refundExpiredSubmitClaimRequest` function, which could be exploited to drain contract funds, and flaws in the `HATArbitrator.reclaimBond` function leading to bond refund abuse. Medium severity issues highlighted the potential for unvested tokens to be stuck post-contract revocation. Additionally, low severity issues included transaction failures due to deprecated transfer functions and the risk of lost funds from the `predictSplitterAddress` function without input validation.

The report underscores the effectiveness of Hats.finance's decentralized audit model in timely identifying and addressing security vulnerabilities, thereby enhancing the robustness of Web3 projects.

## Disclaimer


This report does not assert that the audited contracts are completely secure. Continuous review and comprehensive testing are advised before deploying critical smart contracts.


The HATs Arbitration Contracts audit competition illustrates the collaborative effort in identifying and rectifying potential vulnerabilities, enhancing the overall security and functionality of the platform.


Hats.finance does not provide any guarantee or warranty regarding the security of this project. Smart contract software should be used at the sole risk and responsibility of users.

