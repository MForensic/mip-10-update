---
mip: 11
title: Automatic Priority Fee Distribution
description: Automatically distribute priority fees to delegators.
author: <your-name-here>
discussions-to: <forum-link-or-thread>
status: Draft
type: Standards Track
category: Core
created: 2026-04-01
---

## Abstract

We propose that priority fees are automatically distributed to delegators.

## Motivation
Currently, priority fees are credited directly to a validator's beneficiary address. To ensure consistent compensation for all stakers, this proposal introduces a mechanism to automatically distribute priority fees directly to delegators.

## Specification

### Overview

There are two components for automated priority fee distribution:

1. Introduce an account for capturing priority fees. This will be referred to as the `distribution account`. 
2. Introduce logic at the end of block execution that calls `external_rewards` for corresponding validator pool with the priority fees accumulated in `distribution account.` 

The `beneficiary` remains settable by the block proposer. Within execution context, block.coinbase address remains the beneficiary address.

### Execution Flow
The following changes are applied during block execution:


1. The beneficiary remains set by the block proposer and is still represented by block.coinbase in execution context.

2. For each transaction: The priority fee is credited to the distribution account instead of the beneficiary balance.


3. At the end of block execution: The system calls syscall_distribute on the distribution account. This function forwards the full accumulated balance to the staking contract via external_rewards.

The distribution account will have the following logic. 

```
Class distribution_account: 
    
	# only system can call this function
	def syscall_distribute(address block_leader):
	    priority_fees = get_balance(address(this))
	    
	    # This exists as its the same value as syscall_reward block_leader 
	    val_id = staking_contract.val_id(block_leader)
	    
	    # Need to check min priority fees possible in block in staking constants
	    staking_contract.external_rewards(val_id){msg.value = priority_fees}
```
## Rationale
Priority fees are a component of validator revenue.  Delegators should receive this allocation as participants for securing the network. 

This design ensures:

1. Native inclusion of priority fees within staking rewards; 
2. Elimination of reliance on off-chain or external distribution logic.


## Backwards Compatibility
This change modifies the flow of priority fees. Priority fees will no longer appear in the balance of block.coinbase as a direct credit.

This update distributes priority fees to all delegators within a validator's pool. This may impact third party delegation contracts. Such contracts will experience a dilution in their share of priority fees if users bypass the external contract and stake directly with the validator pool.

## Security Considerations

The primary consideration is the effects of precision accuracy within the reward accumulator.

`External_rewards` requires a minimum amount, defined as the `dust_threshold`, to guarantee a certain decimal accuracy within the accumulator. The minimum threshold for non-zero priority fees must align with the `external_rewards` minimum balance requirement. Any fees under this threshold will not be distributed.

Some edge cases to consider: 
1. Blocks with zero priority fees will result in a no-op distribution. 
2. Small fee amounts must be validated against dust_threshold. The result being these sub threshold fees are burned. 


## Copyright
Copyright and related rights waived via CC0.