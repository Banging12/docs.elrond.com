---
id: delegation-manager
title: The Delegation Manager
---

## Introducing staking pools

A **staking pool** is defined as a custom delegation smart contract, the associated nodes and the funds staked in the pool by participants. **Node operators** may wish to set up a staking pool for their nodes, which can then be funded by anyone in exchange for a proportion of the validator rewards. This form of funding the stake for validators is called **delegation**.

Staking pools bridge the gap between node operators, who need funds to stake for their nodes, and fund holders who wish to earn rewards by staking their funds, but are not interested in managing validator nodes.

Node operators can set up a staking pool to manage one or more validator nodes. For this purpose, they may use the **delegation manager** built into the Elrond Protocol to create their own **delegation contract**. A delegation contract automates certain tasks required for the management of a staking pool, such as keeping track of every account that has funded the staking pool, keeping track of the nodes themselves, as well as providing information to the delegators.

:::important
A staking pool requires 1250 EGLD deposited by the node operator at the moment of its creation. However, 2500 EGLD is required to stake a single validator node and start earning rewards.
:::

This page describes how to request a new delegation contract from the delegation manager and how to use it. It will focus on the delegation _contract_ more than the delegation _manager_, but the two concepts are intimately linked. However, it is important to remember that it is the delegation _contract_ which handles the staking pool and the nodes associated with it.

Note that the delegation manager is not required to set up a staking pool. For example, it is also possible to set up delegation using a regular smart contract, although that is a more complex process and is not discussed here.

Node operators may also choose to set up a delegation dashboard, although they may use any user interface or none whatsoever. As an example, the boilerplate for such a delegation dashboard can be found here: https://github.com/ElrondNetwork/starter-dapp/tree/master/react-delegationdashboard.


A detailed description of the delegation process can be consulted at https://github.com/ElrondNetwork/elrond-specs/blob/main/sc-delegation-specs.md.

## Creating a new delegation contract

The delegation contract for a new staking pool can be created by issuing a request to the delegation manager. This is done by submitting a transaction of the following form:
```
NewDelegationContractTransaction {
    Sender: <account address of the node operator>
    Receiver: erd1qqqqqqqqqqqqqqqpqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqylllslmq6y6
    Value: 1250000000000000000000 (1250 EGLD)
    GasLimit: 60000000
    Data: "createNewDelegationContract" +
          "@" + <total delegation cap in EGLD, fully denominated, in hexadecimal encoding> +
          "@" + <service fee as hundredths of percents, in hexadecimal encoding>
}
```

The `Receiver` address is set to `erd1qqqqqqqqqqqqqqqpqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqylllslmq6y6`, which is the fixed address of the delegation manager, located on the Metachain.

The `Value` is set to 1250 EGLD, which will be automatically added into the funds of the newly created delegation contract, i.e. this is the initial amount of EGLD in the staking pool. This amount of EGLD always belongs to the owner of the delegation contract.

:::important
The initial 1250 EGLD count towards the total delegation cap, like all the funds in the staking pool.

The initial amount of 1250 EGLD added to the pool makes the owner the first delegator of the staking pool. This means that the owner is also entitled to a proportion of the rewards, which can be claimed like any other delegator.
:::

In the `Data field`, the first argument passed to `createNewDelegationContract` is the total delegation cap (the maximum possible size of the staking pool). It is expressed as a fully denominated amount of EGLD, meaning that it is the number of $10^{-18}$ subdivisions of the EGLD, and not the actual number of EGLD tokens. The fully denominated total delegation cap must then be encoded hexadecimally. Make sure not to encode the ASCII string representing the total delegation cap.

:::tip
For example, to obtain the fully denominated form of 7231.941 EGLD, the amount must be multiplied by $10^{18}$, resulting in 7231941000000000000000. Do not encode the ASCII string `"7231941000000000000000"`, but encode the integer 7231941000000000000000 itself. This would result in `"01880b57b708cf408000"`.
:::

Setting the total delegation cap to 0 ("00" in hexadecimal) specifies an unlimited total delegation amount. It can always be modified later (see [Delegation cap](/validators/delegation-manager#delegation-cap)).

The second argument passed to `createNewDelegationContract` is the service fee that will be reserved for the owner of the delegation contract. It is computed as a proportion of the total rewards earned by the validator nodes. The remaining rewards apart from this proportion will be available to delegators to either claim or redelegate. The service fee is expressed as hundredths of a percent.

:::tip
For example, a service fee of 37.45% is expressed by the integer 3745. This integer must then be encoded hexadecimally (3745 becomes `"0ea1"`).
:::

Setting the service fee to 0 (`"00"` in hexadecimal) specifies that no rewards are reserved for the owner of the delegation contract - all rewards will be available to the delegators. The service fee can always be modified later (see [Service fee](/validators/delegation-manager#service-fee)).

The following is a complete example of a transaction requesting the creation of a new delegation contract:
```
NewDelegationContractTransaction {
    Sender: <account address of the node operator>
    Receiver: erd1qqqqqqqqqqqqqqqpqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqylllslmq6y6
    Value: 1250000000000000000000
    GasLimit: 60000000
    Data: "createNewDelegationContract" +
          "@01880b57b708cf408000" +
          "@0ea1"
}
```

The above transaction creates a new delegation contract owned by the sender, with total delegation cap of 7231.941 EGLD and service fee of 37.45% from the rewards. Moreover, the newly created delegation contract will start with a staking pool of 1250 EGLD.


## Configuring the delegation contract

The owner of the delegation contract has a number of operations at their disposal.

### Metadata

The delegation contract can store information that identifies the staking pool: its human-readable name, its website and its associated keybase.io identity.

```
SetMetadataTransaction {
    Sender: <account address of the delegation contract owner>
    Receiver: <address of the delegation contract>
    Value: 0
    GasLimit: 2000000
    Data: "setMetaData"
          "@" + <name of the staking pool, in hexadecimal encoding> +
          "@" + <website of the staking pool, in hexadecimal encoding > +
          "@" + <keybase.io identity of the staking pool, in hexadecimal encoding>
}
```

An example for the `Data` field that sets the name to `"Elrond Staking"`, the website to `"elrond.staking"` and the keybase.io identifier to `"elrondstaking"` is:
```
    "setMetaData" + 
    "@456c726f6e64205374616b696e67"     //Elrond Staking
    "@656c726f6e642e7374616b696e67"     //elrond.staking     
    "@656c726f6e647374616b696e67"       //elrondstaking
```

:::important
Setting the keybase.io identity of the staking pool in the metadata is the **first step** in connecting the delegation contract and a keybase.io identity. The second step is explained in the next section [Display information](/validators/delegation-manager#display-information) where the inverse connection is made: from the keybase.io identity to the delegation contract address.
:::

### Display information

To customize the information for your delegation contract, which will be available in the lists displayed on the delegation pages both in Maiar and the web wallet, some additional information has to be added on the keybase.io account. Please fill in the **avatar picture** and edit the profile providing the **name** and **bio**. This information together with the **service fee, percentage filled** and **APR** will be displayed for every delegation contract on the delegation pages in the web wallet and Maiar. If this information cannot be found a generic logo and the delegation contract's address is displayed.

In order to complete the matching between the delegation contract and keybase.io identity of the staking pool an empty file with the name set to the delegation contract's address has to be added in the `/public/<keybase.io identity>/elrond/` folder.

An example for the path to the empty file for the `"elrondstaking"` keybase.io identity would be:

```
 public/elrondstaking/elrond/erd1qqqqqqqqqqqqqqqpqqqqqqqqqqqqqqqqqqqqqqqqqqqqqr0llllsj732py
```

:::important
The **second step** in connecting the delegation contract and the keybase.io identity is finalized by adding the empty file, which has the name equal to the delegation contract's address, and certifying that the referenced keybase.io identity, set in the [metadata](/validators/delegation-manager#metadata), also references the delegation contract address. This way the connection from both directions is sealed.
:::

:::tip
To be able to connect a **testnet** or **devnet** contract to a keybase.io identity, a new folder - named `"testnet"` for the testnet, or `"devnet"` for the devnet, has to be created inside the `/elrond` folder. An example for the same delegation contract on the `testnet` would be:

```
public/elrondstaking/elrond/testnet/erd1qqqqqqqqqqqqqqqpqqqqqqqqqqqqqqqqqqqqqqqqqqqqqr0llllsj732py
```
:::

An example of how the delegation contract will be displayed based on the information provided in the keybase.io is provided below.

![stakingpool](/img/stakingpool.png)

:::important
To also connect the validators themselves to a specific keybase.io staking pool identity, two additional steps have to be completed:
1. Create an empty file with the name set to the `"<BLS key>"` for every validator and add the empty file to the `/elrond` folder on your keybase.io identity: `public/<keybase.io identity>/elrond/<BLS key>`
2. Set the `Identity` of each validator in the `config/prefs.toml` file to the keybase.io staking pool identity.

```
[Preferences]
   # Identity represents the keybase's identity
   Identity = "<keybase.io identity>"    // e.g.  Identity = "elrondstaking"
```


:::

### Service fee

The service fee is a percentage of the validator rewards that will be reserved for the owner of the delegation contract. The rest of the rewards will be available to delegators to either claim or redelegate.

The service fee can be changed at any time using a transaction of the form:
```
ChangeServiceFeeTransaction {
    Sender: <account address of the delegation contract owner>
    Receiver: <address of the delegation contract>
    Value: 0
    GasLimit: 2000000
    Data: "changeServiceFee" +
          "@" + <service fee as hundredths of percents, in hexadecimal encoding>
}
```

In the `Data` field, the only argument passed to `changeServiceFee` is the new value of the service fee, expressed as hundredths of a percent.

Setting the service fee to 0 (`"00"` in hexadecimal) specifies that no rewards are reserved for the owner of the delegation contract - all rewards will be available to the delegators. The service fee can always be modified later.

:::tip
For example, a service fee of 37.45% is expressed by the integer 3745. This integer must then be encoded hexadecimally (3745 becomes `"0ea1"`).

Finally, a `Data` field containing `changeServiceFee@0ea1` will change the service fee to 37.45%.
:::


### Automatic activation

When automatic activation is enabled, the delegation contract will activate (stake) inactive nodes as soon as funds have become available in sufficient amount. Consequently, any [delegation transaction](/validators/delegation-manager#delegating-funds) can potentially trigger the activation of inactive nodes, assuming the transaction has sufficient gas.

Automatic activation can be enabled or disabled using a transaction of the form:
```
SetAutomaticActivationTransaction {
    Sender: <account address of the delegation contract owner>
    Receiver: <address of the delegation contract>
    Value: 0
    GasLimit: 2000000
    Data: "setAutomaticActivation" +
          "@" + <"true" or "false" in hexadecimal encoding>
}
```

The only argument passed to `setAutomaticActivation` is either `true` or `false`, as an ASCII string encoded hexadecimally. For reference, `true` is `"74727565"` and `false` is `"66616c7365"`.

:::tip
For example, a `Data` field containing `"setAutomaticActivation@74727565"` enables automatic activation.
:::


### Delegation cap

The total delegation cap is the maximum possible size amount of EGLD which can be held by the delegation contract. After reaching the total delegation cap, the contract will reject any subsequent funds.

The total delegation cap can be modified at any time using a transaction of the form:
```
ModifyTotalDelegationCapTransaction {
    Sender: <account address of the delegation contract owner>
    Receiver: <address of the delegation contract>
    Value: 0
    GasLimit: 2000000
    Data: "modifyTotalDelegationCap" +
          "@" + <total delegation cap in EGLD, fully denominated, in hexadecimal encoding>
}
```

In the `Data` field, the only argument passed to `modifyTotalDelegationCap` is the new value for the delegation cap. It is expressed as a fully denominated amount of EGLD, meaning that it is the number of $10^{-18}$ subdivisions of the EGLD, and not the actual number of EGLD tokens. Take sure not to encode the ASCII string representing the total delegation cap.

:::tip
For example, to obtain the fully denominated form of 7231.941 EGLD, the amount must be multiplied by the denomination factor $10^{18}$, resulting in 7231941000000000000000. Do not encode the ASCII string `"7231941000000000000000"`, but encode the integer 7231941000000000000000 itself. This would result in "01880b57b708cf408000".

Finally, a `Data` field containing `"modifyTotalDelegationCap@01880b57b708cf408000"` will change the total delegation cap to 7231.941 EGLD.
:::

Setting the total delegation cap to 0 (`"00"` in hexadecimal) specifies an unlimited total delegation amount. It can always be modified later.

:::important
The total delegation cap cannot be set to a value lower than the amount staked for currently active nodes. It must be either higher than that amount or set to 0 (infinite cap).
:::


## Managing nodes

### Adding nodes

When a delegation contract is first created, it contains no information about nodes. The owner of the contract must then register nodes into the contract, so that they can be later activated. Any newly added node is "inactive" by default.

Adding nodes requires the BLS key pairs belonging to each of them, which the owner of the contract uses to prove that they have access to the nodes. This proof consists of signing the address of the delegation contract itself with the secret BLS key of each node, individually. This results in as many signed messages as there are nodes.

Adding `N` nodes to the delegation contract is done by submitting a transaction with the values set as follows:
```
AddNodesTransaction {
    Sender: <account address of the delegation contract owner>
    Receiver: <address of the delegation contract>
    Value: 0
    GasLimit: 1000000 + N·6000000
    Data: "addNodes" +
          "@" + <public BLS key of the first node in hexadecimal encoding> +
          "@" + <address of the delegation contract signed with the secret BLS key of the first node, in hexadecimal encoding> +
          "@" + <public BLS key of the second node in hexadecimal encoding> +
          "@" + <address of the delegation contract signed with the secret BLS key of the second node, in hexadecimal encoding> +
          <...> +
          "@" + <public BLS key of the Nth node in hexadecimal encoding> +
          "@" + <address of the delegation contract signed with the secret BLS key of the Nth node, in hexadecimal encoding>
}
```

As shown above, the `Data` field contains an enumeration of `N` pairs. Such a pair consists of the public BLS key of a node along with the message produced by signing the address of the delegation contract with the secret BLS key of the respective node. There are as many pairs as there are nodes to add.


### Staking nodes

When the staking pool held by the delegation contract contains a sufficient amount of EGLD, the inactive (non-staked) nodes can be staked (activated). This promotes the nodes to the status of **validator**, which means they participate in consensus and earn rewards.

This subsection describes the _manual_ staking (activation) of nodes. To automatically stake (activate) nodes when funds become available, [automatic activation](/validators/delegation-manager#automatic-activation) can be enabled.

To stake specific nodes manually, a transaction of the following form can be submitted:

```
StakeNodesTransaction {
    Sender: <account address of the delegation contract owner>
    Receiver: <address of the delegation contract>
    Value: 0
    GasLimit: 1000000 + N·6000000
    Data: "stakeNodes" +
          "@" + <public BLS key of the first node in hexadecimal encoding> +
          "@" + <public BLS key of the second node in hexadecimal encoding> +
          <...> +
          "@" + <public BLS key of the Nth node in hexadecimal encoding> +
}
```

The `Data` field contains an enumeration of `N` public BLS keys corresponding to the nodes to be staked.


### Unstaking nodes

Validator nodes that are already staked (active) can be manually unstaked.

:::important
Validators are demoted to validator status at the beginning of the next epoch _after unstaking_. This means that they stop receiving rewards.

Unstaking _does not_ mean that the staked amount returns to the staking pool (see [undelegating](/validators/delegation-manager#undelegating-funds) and [withdrawing](/validators/delegation-manager#withdrawing)).
:::

To cancel the deactivation before the unstaking is complete, the nodes can be [restaked](/validators/delegation-manager#restaking-nodes).

To begin the deactivation process for a selection of validator nodes, a transaction of the following form is used:
```
UnstakeNodesTransaction {
    Sender: <account address of the delegation contract owner>
    Receiver: <address of the delegation contract>
    Value: 0
    GasLimit: 1000000 + N·6000000
    Data: "unStakeNodes" +
          "@" + <public BLS key of the first node in hexadecimal encoding> +
          "@" + <public BLS key of the second node in hexadecimal encoding> +
          <...> +
          "@" + <public BLS key of the Nth node in hexadecimal encoding> +
}
```
The `Data` field contains an enumeration of `N` public BLS keys corresponding to the nodes to be unstaked.


### Restaking nodes

Validator nodes that have been unstaked can be restaked (reactivated) before their deactivation is complete. To cancel their deactivation, a transaction of the following form is used:
```
RestakeNodesTransaction {
    Sender: <account address of the delegation contract owner>
    Receiver: <address of the delegation contract>
    Value: 0
    GasLimit: 1000000 + N·6000000
    Data: "reStakeUnStakedNodes" +
          "@" + <public BLS key of the first node in hexadecimal encoding> +
          "@" + <public BLS key of the second node in hexadecimal encoding> +
          <...> +
          "@" + <public BLS key of the Nth node in hexadecimal encoding> +
}
```
The `Data` field contains an enumeration of `N` public BLS keys corresponding to the nodes to be restaked.


### Unbonding nodes

Nodes that have been [unstaked](/validators/delegation-manager#unstaking-nodes) can be completely deactivated, a process called **unbonding**.

:::important
Validators are demoted to observer status at the beginning of the next epoch _after unstaking_, not unbonding. See [unstaking](/validators/delegation-manager#unstaking-nodes) above.
:::

Validator nodes that have been unbonded cannot be restaked (reactivated). They must be staked anew.

```
UnbondNodesTransaction {
    Sender: <account address of the delegation contract owner>
    Receiver: <address of the delegation contract>
    Value: 0
    GasLimit: 1000000 + N·6000000
    Data: "unBondNodes" +
          "@" + <public BLS key of the first node in hexadecimal encoding> +
          "@" + <public BLS key of the second node in hexadecimal encoding> +
          <...> +
          "@" + <public BLS key of the Nth node in hexadecimal encoding> +
}
```
The `Data` field contains an enumeration of `N` public BLS keys corresponding to the nodes to be unbonded.


### Removing nodes

Inactive (not staked, unbonded) nodes can be removed from the delegation contract by the owner at any time. Neither active (staked) nor unstaked nodes cannot be removed.

Unlike [adding nodes](/validators/delegation-manager#adding-nodes), this step does not require the BLS key pairs of the nodes.

Removing `N` nodes from the delegation contract is done by submitting a transaction with the values set as follows:
```
RemoveNodesTransaction {
    Sender: <account address of the delegation contract owner>
    Receiver: <address of the delegation contract>
    Value: 0
    GasLimit: 1000000 + N·6000000
    Data: "removeNodes" +
          "@" + <public BLS key of the first node in hexadecimal encoding> +
          "@" + <public BLS key of the second node in hexadecimal encoding> +
          <...> +
          "@" + <public BLS key of the Nth node in hexadecimal encoding> +
}
```
The `Data` field contains an enumeration of `N` public BLS keys corresponding to the nodes to be removed.


### Unjailing nodes

When active validator nodes perform poorly or to the detriment of the network, they are penalized by having their rating reduced. Rating is essential to earning rewards, because it directly determines the likelihood of a validator to be [selected for consensus](/technology/secure-proof-of-stake).

However, it can happen that rating of a validator might drop under the acceptable threshold. As a consequence, the validator will begin its next epoch **jailed**, which prevents it from participating in consensus.

:::important
A jailed validator does not lose its stake nor its status. It remains active, but it cannot earn rewards while in jail.
:::

Recovering a validator from jail and restoring it is called **unjailing**, for which a fine of 2.5 EGLD must be paid. Multiple validators can be recovered from jail at the same time by paying 2.5 EGLD for each validator. The format of the unjailing transaction is as follows:
```
UnjailNodesTransaction {
    Sender: <account address of the delegation contract owner>
    Receiver: <address of the delegation contract>
    Value: 2.5 EGLD × <number of nodes to be unjailed>
    GasLimit: 1000000 + N·6000000
    Data: "unJailNodes" +
          "@" + <public BLS key of the first node in hexadecimal encoding> +
          "@" + <public BLS key of the second node in hexadecimal encoding> +
          <...> +
          "@" + <public BLS key of the Nth node in hexadecimal encoding> +
}
```

Note that the `Value` field depends on `N`, the number of validators to unjail.

The `Data` field contains an enumeration of `N` public BLS keys corresponding to the nodes to be unjailed.


## Delegating and managing delegated funds

Accounts that delegate their own funds to the staking pool are called **delegators**. The delegation contract offers them a set of actions as well. This means that these actions are available to the owner of the delegation contract as well.


### Delegating funds

Accounts become delegators by funding the staking pool, i.e. they delegate their funds. The delegators are rewarded for their contribution with a proportion of the rewards earned by the validator nodes. By default, the owner of the delegation contract is the first delegator, having already contributed 1250 EGLD to the staking pool at its creation.

:::important
Extra funds received by the delegation contract from delegators will be immediately used to top-up the stake of the existing active validators, consequently increasing their rewards.
:::

Submitting a delegation transaction takes into account the status of [automatic activation](/validators/delegation-manager#automatic-activation): if the delegated funds cause the amount in the staking pool to become sufficient for the staking of extra nodes, it can trigger their activation automatically. This happens only if the transaction contains enough gas.

But if gas is insufficient, or if automatic activation is disabled, the amount received through the delegation transaction simply becomes top-up for the stake of already active validators. Subsequent [manual staking](/validators/delegation-manager#staking-nodes) will be necessary to use the funds for staking, assuming they are sufficient.

Funds can be delegated by any fund holder by submitting a transaction of the following form:
```
DelegateTransaction {
    Sender: <account address of funds holder>
    Receiver: <address of the delegation contract>
    Value: minimum 1 EGLD
    GasLimit: 12000000
    Data: "delegate"
}
```

If the transaction is successful, the funds' holder has become a delegator and the funds either become a top-up amount for the stake of active validators, or may trigger the staking of inactive nodes, as described above.


### Claiming rewards

A portion of the rewards earned by validator nodes is reserved for each delegator. To claim the rewards, a delegator may issue a transaction of the following form:
```
ClaimRewardsTransaction {
    Sender: <account address of existing delegator>
    Receiver: <address of the delegation contract>
    Value: 0
    Gas: 6000000
    Data: "claimRewards"
}
```

If the transaction is successful, the delegator receives the proportion of rewards they are entitled to.

### Redelegating rewards

Current delegation rewards can also be immediately delegated instead of [claimed](/validators/delegation-manager#claiming-rewards). This makes it an operation very similar to [delegation](/validators/delegation-manager#delegating-funds).

:::important
Just like delegation, redelegation of rewards takes into account the status of [automatic activation](/validators/delegation-manager#automatic-activation): if the redelegated rewards cause the amount in the staking pool to become sufficient for the staking of extra nodes, it can trigger their activation automatically (requires sufficient gas in the redelegation transaction).
:::

Rewards are redelegated using a transaction of the form:
```
RedelegateRewardsTransaction {
    Sender: <account address of existing delegator>
    Receiver: <address of the delegation contract>
    Value: 0
    Gas: 12000000
    Data: "reDelegateRewards"
}
```

If the transaction is successful, the delegator does not receive any EGLD at the moment, but the rewards they were entitled to will be added to their delegated amount.


### Undelegating funds

Delegators may express the intent to withdraw a specific amount of EGLD from the staking pool. However, this process cannot happen at once and may take a few epochs before the amount is actually available for withdrawal, because the funds may already be used to stake for active validators and this means that unstaking of nodes may be necessary.

:::important
If the amount to undelegate requested by the delegator will cause the staking pool to drop below the sufficient amount required to keep all the current validators active, some validators will inevitably end up unstaked. The owner of the delegation contract may intervene and add extra funds to prevent such situations.
:::

Funds that have been previously used as stake for validators have been transferred into a separate system smart contract at the moment of staking, therefore the delegation contract itself does not hold these funds. But submitting an undelegation request will cause the delegation contract to attempt their retrieval.

The delegation contract may receive the funds immediately if they're not currently used as stake; this makes them available for subsequent [withdrawal](/validators/delegation-manager#withdrawing). This is the case where previously delegated funds acted as top-up to the stake of existing validators.

On the other hand, if the requested funds are currently in use as stake, the delegation contract cannot receive them yet.

:::important
Funds used as stake can only be retrieved after 144000 blocks have been built on the Metachain (a little over 10 chronological epochs). It doesn't matter whether the validator was already demoted to observer, or whether it has been decomissioned entirely - the funds may not return until the aforementioned time has passed.

After 144000 blocks, the funds can be [withdrawn](/validators/delegation-manager#withdrawing) normally by their rightful owner.
:::

To express the intention of future withdrawal of funds from the staking pool, a delegator may submit the following transaction:

```
UndelegateTransaction {
    Sender: <account address of existing delegator>
    Receiver: <address of the delegation contract>
    Value: 0
    Gas: 12000000
    Data: "unDelegate"
          "@" + <amount to undelegate in EGLD, minimum 1 EGLD, fully denominated, in hexadecimal encoding>
}
```

In the `Data` field, the only argument passed to `unDelegate` is the desired amount of EGLD to undelegate and later withdraw. It is expressed as a fully denominated amount of EGLD, meaning that it is the number of $10^{-18}$ subdivisions of the EGLD, and not the actual number of EGLD tokens. The fully denominated amount must then be encoded hexadecimally. Make sure not to encode the ASCII string representing the amount.


### Withdrawing

After submitting an [undelegation transaction](/validators/delegation-manager#undelegating-funds), a delegator may finally withdraw funds from the staking pool.

:::important
Funds must always be [undelegated](/validators/delegation-manager#undelegating-funds) first. They cannot be directly withdrawn.
:::

This action withdraws _all the currently undelegated funds_ belonging to the specific delegator.

Withdrawing funds is done using a transaction of the following form:

```
WithdrawTransaction {
    Sender: <account address of existing delegator>
    Receiver: <address of the delegation contract>
    Value: 0
    GasLimit: 12000000
    Data: "withdraw"
}
```

If the transaction is successful, the delegator receives all the EGLD they have previously requested to undelegate. The amount is removed from the staking pool.


## Delegation contract view functions

The delegation contract can be queried using the following view functions. These queries should be done on a local proxy on the `/vm-values/query` endpoint.

The following documentation sections only show the value of the relevant `returnData` field and omit the other fields for simplicity.

```json
{
  "data": {
    "data": {
      "returnData": [
      ],
      "returnCode": "ok",
      "returnMessage": "",
      "gasRemaining": 0,
      "gasRefund": 0,
      "outputAccounts": null,
      "deletedAccounts": null,
      "touchedAccounts": null,
      "logs": []
    }
  },
  "error": "",
  "code": "successful"
}
```


### <span class="badge badge-success">POST</span> Contract config

The response contains an array of the properties in a fixed order (base64 encoded): owner address, service fee, maximum delegation cap, initial owner funds, automatic activation, with delegation cap, can change service fee, check cap on redelegate, nonce on creation and unbond period.

<!--DOCUSAURUS_CODE_TABS-->

<!--Request-->
```
https://proxy:port/vm-values/query
```
```json
{
  "scAddress": "<address of the delegation contract in bech32 encoding>",
  "funcName": "getContractConfig"
}
```

<!--Response-->
Only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response
```json
{
  "returnData": [
    "<owner address bytes in base64 encoding of the hexadecimal encoding>",
    "<service fee as hundredths of percents base64 of the hexadecimal encoding>",
    "<maximum delegation cap in base64 of the hexadecimal encoding>",
    "<initial owner funds in base64 of the hexadecimal encoding>",
    "<has automatic activation - true/false in base64 encoding",
    "<has delegation cap - true/false in base64 encoding",
    "<changeable service fee - true/false in base64 encoding",
    "<check cap on redelegate - true/false in base64 encoding",
    "<blockchain nonce at creation in base64 of the hexadecimal encoding>",
    "<unbond period in rounds in base64 of the hexadecimal encoding>"
  ]
}
```

<!--Example-->
Request
```json
{
  "scAddress": "erd1qqqqqqqqqqqqqqqpqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqhllllsajxzat",
  "funcName": "getContractConfig"
}
```

Response (only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response)
```json
{
  "returnData": [
    "gKzHUD288mzScNX6nEmkGm4CHneMdrrPhJyPET9iGA8=",
    null,
    "",
    "Q8M8GTdWSAAA",
    "dHJ1ZQ==",
    "ZmFsc2U=",
    "dHJ1ZQ==",
    "AuU=",
    "+g=="
  ]

}
```

<!--END_DOCUSAURUS_CODE_TABS-->


### <span class="badge badge-success">POST</span> Contract metadata

The response contains an array of the properties in a fixed order (base64 encoded): staking provider name, website and identifier.

<!--DOCUSAURUS_CODE_TABS-->

<!--Request-->
```
https://proxy:port/vm-values/query
```
```json
{
  "scAddress": "<address of the delegation contract in bech32 encoding>",
  "funcName": "getMetaData"
}
```

<!--Response-->
Only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response
```json
{
  "returnData": [
    "<staking provider name in base64 encoding>",
    "<staking provider website in base64 encoding>",
    "<staking provider identifier in base64 encoding>"
  ]
}
```


<!--Example-->
Request
```json
{
  "scAddress": "erd1qqqqqqqqqqqqqqqpqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqhllllsajxzat",
  "funcName": "getMetaData"
}
```

Response (only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response)
```json
{
  "returnData": [
    "U3Rha2luZyBwcm92aWRlciB0ZXN0",
    "d3d3LmVscm9uZHN0YWtpbmcuY29t",
    "dGVzdEtleWJhc2VJZGVudGlmaWVy"
  ]
}
```

<!--END_DOCUSAURUS_CODE_TABS-->


### <span class="badge badge-success">POST</span> Number of delegators

The response contains a value representing the number of delegators in base64 encoding of the hex encoding.


<!--DOCUSAURUS_CODE_TABS-->

<!--Request-->
```
https://proxy:port/vm-values/query
```
```json
{
  "scAddress": "<address of the delegation contract in bech32 encoding>",
  "funcName": "getNumUsers"
}
```

<!--Response-->
Only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response
```json
 {
  "returnData": [
    "<number of distinct delegators in base64 encoding of the hexadecimal encoding>"
  ]
}
```

<!--Example-->
Request
```json
{
  "scAddress": "erd1qqqqqqqqqqqqqqqpqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqhllllsajxzat",
  "funcName": "getNumUsers"
}
```

Response (only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response)
```json
 {
  "returnData": [
    "BQ=="
  ]
}
```

<!--END_DOCUSAURUS_CODE_TABS-->


### <span class="badge badge-success">POST</span> Number of nodes

The response contains the number of nodes in base64 encoding of the hex encoding.


<!--DOCUSAURUS_CODE_TABS-->

<!--Request-->
```
https://proxy:port/vm-values/query
```
```json
{
  "scAddress": "<address of the delegation contract in bech32 encoding>",
  "funcName": "getNumNodes"
}
```

<!--Response-->
Only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response
```json
{
  "returnData": [
    "<number of nodes in base64 encoding of the hexadecimal encoding>"
  ]
}
```

<!--Example-->
Request
```json
{
  "scAddress": "erd1qqqqqqqqqqqqqqqpqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqhllllsajxzat",
  "funcName": "getNumNodes"
}
```

Response (only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response)
```json
{
  "returnData": [
    "Dg=="
  ]
}
```

<!--END_DOCUSAURUS_CODE_TABS-->


### <span class="badge badge-success">POST</span> Nodes states

The response contains an enumeration of alternating status codes and BLS keys. Each status code is followed by the BLS key of the node it describes. Both status codes and BLS keys are encoded in base64.


<!--DOCUSAURUS_CODE_TABS-->

<!--Request-->
```
https://proxy:port/vm-values/query
```
```json
{
  "scAddress": "<address of the delegation contract in bech32 encoding>",
  "funcName": "getAllNodeStates"
}
```

<!--Response-->
Only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response
```json
 {
  "returnData": [
    "<state in base64 encoding>",
    "<public BLS key of the node in hexadecimal encoding>"
  ]
}
```

<!--Example-->
Request
```json
{
  "scAddress": "erd1qqqqqqqqqqqqqqqpqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqhllllsajxzat",
  "funcName": "getAllNodeStates"
}
```

Response (only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response)
```json
 {
  "returnData": [
    "c3Rha2Vk",
    "KJ6auG3rKQydktc9soWvyBOa5UPA7DYezttTqlS6JIIvsvOaH8ghs2Qruc4aXLUXNJ1if7Ot9gbt5dNUrmNfkLtZl1hpLvPllrGmFP4bKCzZ25UNiTratwOMcXhhCmSD",
    "bm90U3Rha2Vk",
    "7gJzQ3GQ4htSx6CYvOkXPDdwGfzdahuDY4agZkGhIAMfB44K08FP6z3wLQEnn2IULfZ8/Hds38LEu3Xq+mJZ4FktF0vm8C1T34b5uAEpZWtDZLICAEFCuQZrqS5Qb1CR",
    "vTyNQ/vDxg0L8LmoGuKP+4/wsbyWv8RaqeQ+WH+xrMvk1m7Q3wjheOpjYtQPz80YZ1CrwKj6ObsCUejP4uuvi3MQ1oMEGKg5yh3kRgybRb4TXAWEpAPszYMLIQhrIn2P",
    "9TbGQCcrbyXH9HBAhzIWOuH/cdSNO1dwxO5foM2L28tWU0p9Kos6DKsPMtKMx4sAeRal08K3Dk0gQxeTSAvC2fb3DAQt01rmPSAqCSXZetSX12BVcTi+pYGUHaXKJ/OW"
  ]
}
```

<!--END_DOCUSAURUS_CODE_TABS-->


### <span class="badge badge-success">POST</span> Total active stake

The response contains a value representing the total active stake in base64 encoding of the hex encoding.


<!--DOCUSAURUS_CODE_TABS-->

<!--Request-->
```
https://proxy:port/vm-values/query
```
```json
{
  "scAddress": "<address of the delegation contract in bech32 encoding>",
  "funcName": "getTotalActiveStake"
}
```

<!--Response-->
Only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response
```json
 {
  "returnData": [
    "<total active stake in base64 encoding of the hex encoding>"
  ]
}
```

<!--Example-->
Request
```json
{
  "scAddress": "erd1qqqqqqqqqqqqqqqpqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqhllllsajxzat",
  "funcName": "getTotalActiveStake"
}
```

Response (only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response)
```json
 {
  "returnData": [
    "ArXjrxaxiAAA"
  ]
}
```

<!--END_DOCUSAURUS_CODE_TABS-->


### <span class="badge badge-success">POST</span> Total unstaked stake

The response contains a value representing the total unstaked stake in base64 encoding of the hex encoding.


<!--DOCUSAURUS_CODE_TABS-->

<!--Request-->
```
https://proxy:port/vm-values/query
```
```json
{
  "scAddress": "<address of the delegation contract in bech32 encoding>",
  "funcName": "getTotalUnStaked"
}
```

<!--Response-->
Only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response
```json
 {
  "returnData": [
    "<total unstaked stake in base64 encoding of the hex encoding>"
  ]
}
```

<!--Example-->
Request
```json
{
  "scAddress": "erd1qqqqqqqqqqqqqqqpqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqhllllsajxzat",
  "funcName": "getTotalUnStaked"
}
```

Response (only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response)
```json
 {
  "returnData": [
    "ArXjrxaxiAAA"
  ]
}
```

<!--END_DOCUSAURUS_CODE_TABS-->


### <span class="badge badge-success">POST</span> Total cumulated rewards

The response contains a value representing the sum of all accumulated rewards in base64 encoding of the hex encoding.


<!--DOCUSAURUS_CODE_TABS-->

<!--Request-->
```
https://proxy:port/vm-values/query
```
```json
{
  "scAddress": "<address of the delegation contract in bech32 encoding>",
  "funcName": "getTotalCumulatedRewards",
  "caller": "erd1qqqqqqqqqqqqqqqpqqqqqqqqlllllllllllllllllllllllllllsr9gav8"
}
```

<!--Response-->
Only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response
```json
 {
  "returnData": [
    "<total accumulated rewards in base64 encoding of the hex encoding>"
  ]
}
```

<!--Example-->
Request
```json
{
  "scAddress": "erd1qqqqqqqqqqqqqqqpqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqhllllsajxzat",
  "funcName": "getTotalCumulatedRewards",
  "caller": "erd1qqqqqqqqqqqqqqqpqqqqqqqqlllllllllllllllllllllllllllsr9gav8"
}
```

Response (only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response)
```json
 {
  "returnData": [
    "czSCSSYZr8E="
  ]
}
```

<!--END_DOCUSAURUS_CODE_TABS-->


### <span class="badge badge-success">POST</span> Delegator claimable rewards

The response contains a value representing the total claimable rewards for the delegator in base64 encoding of the hex encoding.


<!--DOCUSAURUS_CODE_TABS-->

<!--Request-->
```
https://proxy:port/vm-values/query
```
```json
{
  "scAddress": "<address of the delegation contract in bech32 encoding>",
  "funcName": "getClaimableRewards",
  "args" : ["<delegator address in hexadecimal encoding of the bech32 decoded value>"]
}
```

<!--Response-->
Only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response
```json
 {
  "returnData": [
    "<delegator claimable rewards in base64 encoding of the hex encoding>"
  ]
}
```

<!--Example-->
Request
```json
{
  "scAddress": "erd1qqqqqqqqqqqqqqqpqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqhllllsajxzat",
  "funcName": "getClaimableRewards",
  "args":["ebfd923cd251f857ed7639e87143ac83f12f423827abc4a0cdde0119c3e37915"]
}
```

Response (only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response)
```json
 {
  "returnData": [
    "Ft9RZzF7Dyc"
  ]
}
```

<!--END_DOCUSAURUS_CODE_TABS-->


### <span class="badge badge-success">POST</span> Delegator total accumulated rewards

The response contains a value representing the total accumulated rewards for the delegator in base64 encoding of the hex encoding.


<!--DOCUSAURUS_CODE_TABS-->

<!--Request-->
```
https://proxy:port/vm-values/query
```
```json
{
  "scAddress": "<address of the delegation contract in bech32 encoding>",
  "funcName": "getTotalCumulatedRewardsForUser",
  "args" : ["<delegator address in hexadecimal encoding of the bech32 decoded value>"]
}
```

<!--Response-->
Only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response
```json
 {
  "returnData": [
    "<delegator total cumulated rewards in base64 encoding of the hex encoding>"
  ]
}
```

<!--Example-->
Request
```json
{
  "scAddress": "erd1qqqqqqqqqqqqqqqpqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqhllllsajxzat",
  "funcName": "getTotalCumulatedRewardsForUser",
  "args":["ebfd923cd251f857ed7639e87143ac83f12f423827abc4a0cdde0119c3e37915"]
}
```

Response (only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response)
```json
 {
  "returnData": [
    "Ft9RZzF7Dyc"
  ]
}
```

<!--END_DOCUSAURUS_CODE_TABS-->


### <span class="badge badge-success">POST</span> Delegator active stake

The response contains a value representing the active stake for the delegator in base64 encoding of the hex encoding.


<!--DOCUSAURUS_CODE_TABS-->

<!--Request-->
```
https://proxy:port/vm-values/query
```
```json
{
  "scAddress": "<address of the delegation contract in bech32 encoding>",
  "funcName": "getUserActiveStake",
  "args" : ["<delegator address in hexadecimal encoding of the bech32 decoded value>"]
}
```

<!--Response-->
Only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response
```json
 {
  "returnData": [
    "<active stake in base64 encoding of the hex encoding>"
  ]
}
```

<!--Example-->
Request
```json
{
  "scAddress": "erd1qqqqqqqqqqqqqqqpqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqhllllsajxzat",
  "funcName": "getUserActiveStake",
  "args":["ebfd923cd251f857ed7639e87143ac83f12f423827abc4a0cdde0119c3e37915"]
}
```

Response (only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response)
```json
 {
  "returnData": [
    "slsrv1so8QAA"
  ]
}
```

<!--END_DOCUSAURUS_CODE_TABS-->


### <span class="badge badge-success">POST</span> Delegator unstaked stake

The response contains a value representing the unstaked stake for the delegator in base64 encoding of the hex encoding.


<!--DOCUSAURUS_CODE_TABS-->

<!--Request-->
```
https://proxy:port/vm-values/query
```
```json
{
  "scAddress": "<address of the delegation contract in bech32 encoding>",
  "funcName": "getUserUnStakedValue",
  "args" : ["<delegator address in hexadecimal encoding of the bech32 decoded value>"]
}
```

<!--Response-->
Only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response
```json
 {
  "returnData": [
    "<delegator unstaked stake in base64 encoding of the hex encoding>"
  ]
}
```

<!--Example-->
Request
```json
{
  "scAddress": "erd1qqqqqqqqqqqqqqqpqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqhllllsajxzat",
  "funcName": "getUserUnStakedValue",
  "args":["ebfd923cd251f857ed7639e87143ac83f12f423827abc4a0cdde0119c3e37915"]
}
```

Response (only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response)
```json
 {
  "returnData": [
    "ARWORgkT0AAA"
  ]
}
```
<!--END_DOCUSAURUS_CODE_TABS-->


### <span class="badge badge-success">POST</span> Delegator unbondable stake

The response contains a value representing the unbondable stake in base64 encoding of the hex encoding.


<!--DOCUSAURUS_CODE_TABS-->

<!--Request-->
```
https://proxy:port/vm-values/query
```
```json
{
  "scAddress": "<address of the delegation contract in bech32 encoding>",
  "funcName": "getUserUnBondable",
  "args" : ["<delegator address in hexadecimal encoding of the bech32 decoded value>"]
}
```

<!--Response-->
Only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response
```json
 {
  "returnData": [
    "<delegator unbondable stake in base64 encoding of the hex encoding>"
  ]
}
```

<!--Example-->
Request
```json
{
  "scAddress": "erd1qqqqqqqqqqqqqqqpqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqhllllsajxzat",
  "funcName": "getUserUnBondable",
  "args":["ebfd923cd251f857ed7639e87143ac83f12f423827abc4a0cdde0119c3e37915"]
}
```

Response (only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response)
```json
 {
  "returnData": [
    "ARWORgkT0AAA"
  ]
}
```

<!--END_DOCUSAURUS_CODE_TABS-->


### <span class="badge badge-success">POST</span> Delegator undelegated stake

The response contains an enumeration representing the different undelegated stake values in base64 encoding of the hex encoding.


<!--DOCUSAURUS_CODE_TABS-->

<!--Request-->
```
https://proxy:port/vm-values/query
```
```json
{
  "scAddress": "<address of the delegation contract in bech32 encoding>",
  "funcName": "getUserUnDelegatedList",
  "args" : ["<delegator address in hexadecimal encoding of the bech32 decoded value>"]
}
```

<!--Response-->
Only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response
```json
 {
  "returnData": [
    "<undelegated stake in base64 encoding of the hex encoding>"
  ]
}
```

<!--Example-->
Request
```json
{
  "scAddress": "erd1qqqqqqqqqqqqqqqpqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqhllllsajxzat",
  "funcName": "getUserUnDelegatedList",
  "args":["ebfd923cd251f857ed7639e87143ac83f12f423827abc4a0cdde0119c3e37915"]
}
```

Response (only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response)
```json
{
  "returnData": [
    "Q8M8GTdWSAAA",
    "iscjBInoAAA="
  ]
}
```

<!--END_DOCUSAURUS_CODE_TABS-->


### <span class="badge badge-success">POST</span> Delegator funds data

The response contains an enumeration for the delegator encoded base64 of the hexadecimal encoding of the following: active stake, unclaimed rewards, unstaked stake and unbondable stake.


<!--DOCUSAURUS_CODE_TABS-->

<!--Request-->
```
https://proxy:port/vm-values/query
```
```json
{
  "scAddress": "<address of the delegation contract in bech32 encoding>",
  "funcName": "getDelegatorFundsData",
  "args" : ["<delegator address in hexadecimal encoding of the bech32 decoded value>"]
}
```

<!--Response-->
Only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response
```json
 {
  "returnData": [
    "<active stake in base64 encoding of the hex encoding>",
    "<unclaimed rewards in base64 encoding of the hex encoding>",
    "<unstaked stake in base64 encoding of the hex encoding>",
    "<unbondable stake in base64 encoding of the hex encoding>"
  ]
}
```

<!--Example-->
Request
```json
{
  "scAddress": "erd1qqqqqqqqqqqqqqqpqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqhllllsajxzat",
  "funcName": "getUserUnDelegatedList",
  "args":["ebfd923cd251f857ed7639e87143ac83f12f423827abc4a0cdde0119c3e37915"]
}
```

Response (only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response)
```json
{
  "returnData": [
    "REAihYg4zAAA",
    "Q8M8GTdWSAAA",
    "REAihYg4zAAA",
    "Q8M8GTdWSAAA"
  ]
}
```

<!--END_DOCUSAURUS_CODE_TABS-->


### <span class="badge badge-success">POST</span> Get reward data for epoch

The response contains an enumeration for the specified epoch representing the base64 encoding of the hexadecimal encoding for the rewards to distribute, total active stake and service fee.


<!--DOCUSAURUS_CODE_TABS-->

<!--Request-->
```
https://proxy:port/vm-values/query
```
```json
{
  "scAddress": "<address of the delegation contract in bech32 encoding>",
  "funcName": "getRewardData",
  "args" :["<epoch number in hexadecimal encoding>"]
}
```

<!--Response-->
Only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response
```json
 {
  "returnData": [
    "<rewards to distribute in base64 encoding of the hexadecimal encoding>",
    "<total active stake in base64 encoding of the hexadecimal encoding>",
    "<service fee in base64 encoding of the hexadecimal encoding>"
  ]
}
```

<!--Example-->
```json
{
  "scAddress": "erd1qqqqqqqqqqqqqqqpqqqqqqqqqqqqqqqqqqqqqqqqqqqqqp0llllswfeycs",
  "funcName": "getRewardData",
  "args" : ["fc2b"]
}
```

Response (only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response)
```json
{
  "returnData": [
    "REAihYg4zAAA",
    "Q8M8GTdWSAAA",
    "REAihYg4zAAA"
  ]
}
```

<!--END_DOCUSAURUS_CODE_TABS-->


## Delegation mananger view functions


### <span class="badge badge-success">POST</span> All contract addresses

The response contains an enumeration of bech32 keys bytes in base64 encoding.


<!--DOCUSAURUS_CODE_TABS-->

<!--Request-->
```
https://proxy:port/vm-values/query
```
```json
{
  "scAddress": "erd1qqqqqqqqqqqqqqqpqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqylllslmq6y6",
  "funcName": "getAllContractAddresses"
}
```


<!--Response-->
Only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response
```json
{
  "returnData": [
    "<address bytes of the system delegation smart contract in base64 encoding of the hexadecimal encoding>"
  ]
}
```

<!--Example-->
Request
```json
{
  "scAddress": "erd1qqqqqqqqqqqqqqqpqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqylllslmq6y6",
  "funcName": "getAllContractAddresses"
}
```

Response (only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response)
```json
{
  "returnData": [
    "AAAAAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAL///8=",
    "AAAAAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAP///8=",
    "AAAAAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAT///8=",
    "AAAAAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAX///8=",
    "AAAAAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAb///8=",
    "AAAAAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAf///8=",
    "AAAAAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAj///8=",
    "AAAAAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAn///8=",
    "AAAAAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAr///8="
  ]
}
```

<!--END_DOCUSAURUS_CODE_TABS-->


### <span class="badge badge-success">POST</span> Contract config

The response contains an enumeration of the properties in a fixed order (base64 encoded): current number of contracts, last created contract address, minimum and maximum service fee, minimum deposit and delegation.


<!--DOCUSAURUS_CODE_TABS-->

<!--Request-->
```
https://proxy:port/vm-values/query
```
```json
{
  "scAddress": "erd1qqqqqqqqqqqqqqqpqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqylllslmq6y6",
  "funcName": "getContractConfig"
}
```

<!--Response-->
Only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response
```json
{
  "returnData": [
    "<current number of contracts in base64 encoding>",
    "<last used address bytes in base64 encoding of the hexadecimal encoding>",
    "<minimum service fee as hundredths of percents in hexadecimal encoding>",
    "<maximum service fee as hundredths of percents in hexadecimal encoding>",
    "<minimum deposit amount in hexadecimal encoding>",
    "<minimum delegation amount in hexadecimal encoding>"
  ]
}
```

<!--Example-->

Request
```json
{
  "scAddress": "erd1qqqqqqqqqqqqqqqpqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqylllslmq6y6",
  "funcName": "getContractConfig"
}
```

Response (only `returnData` shown below; see [view functions](/validators/delegation-manager#delegation-contract-view-functions) for complete response)
```json
{
  "returnData": [
    "Gw==",
    "AAAAAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAABz///8=",
    "",
    "JxA=",
    "Q8M8GTdWSAAA",
    "iscjBInoAAA="
  ]
}
```

<!--END_DOCUSAURUS_CODE_TABS-->
