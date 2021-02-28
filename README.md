# Time and Process

~8.5 hours in total.

2 hours reading related literature for background knowledge.

2 hours exploring some tools, namely
[Oyente](https://github.com/enzymefinance/oyente), 
[Slither](https://github.com/crytic/slither), and 
[Vandal](https://github.com/usyd-blockchain/vandal).

1 hour identifying a suitable issue for this project.

2 hours writing code and performing experiments.

1.5 hours writing this document.


## Choice for the Tool

First of all, given the limited amount of time, it is a good idea to start from 
an existing 
framework to detect some issues. Either parsing and analyzing the Solidity source code
or analyzing the EVM bytecode will require good infrastructures. 


[Vandal](https://github.com/usyd-blockchain/vandal) is used as the staring pointing 
for this project for the following reasons.

1. It accepts EVM bytecode as input. This enables the tool to cover a larger set 
of EVM based smart contracts (as supposed to taking Solidity code as input).

2. It provides right levels of abstractions. Vandal first decompiles the EVM bytecode
into three-address code (TAC). Then, Vandal extracts higher-level information 
(data flows, control flows, etc.) and express this information in Datalog rules.
One could write and analysis using both three-address code primitives and higher-level 
Datalog rules.


## Choice for the Issue

For this project, the Transaction order dependence (TOD) issue is chosen based on the
following criteria.

1. The issue has not been solved using the Vandal framework. So, issues including 
Reentrancy, Unchecked Call, and Out-of-Gas are not chosen. 
2. The issue involves some domain knowledge in EVM smart contracts. The TOD issue 
involves state variables, which are something special.

# Problem

Detecting Transaction Order Dependence

Transaction order dependence (TOD) is a know issue where different execution orders of 
transactions lead to different results.
Here is a typical scenario for this issue. We have two transactions T1 and T2 invoking 
contract C. T1 set a reward amount, and T2 send the reward to a user. One might observe 
such two transactions and manipulate the order of these two transactions being included
in the block, so the user being rewarded may receive different amounts of rewards.

This issue is documented here https://swcregistry.io/docs/SWC-114 with contract samples.
This is a minimal example demonstrate the TOD issue. If there is one transaction invoking
the setReward() function which updates the reward state variable and anther transaction 
claiming the reward by invoking the claimReward() function, different orders of these two
transactions will transfer different rewards to the receiver.

```solidity
/*
 * @source: https://github.com/ConsenSys/evm-analyzer-benchmark-suite
 * @author: Suhabe Bugrara
 */

pragma solidity ^0.4.16;

contract EthTxOrderDependenceMinimal {
    address public owner;
    bool public claimed;
    uint public reward;

    function EthTxOrderDependenceMinimal() public {
        owner = msg.sender;
    }

    function setReward() public payable {
        require (!claimed);

        require(msg.sender == owner);
        // owner.transfer(reward);
        reward = msg.value; 
    }

    function claimReward(uint256 submission) {
        require (!claimed);
        require(submission < 10);

        msg.sender.transfer(reward);
        claimed = true;
    }
}
``` 

This is a contract without the issue.


```solidity
/*
 * @source: https://github.com/ConsenSys/evm-analyzer-benchmark-suite
 * @author: Suhabe Bugrara
 */

pragma solidity ^0.4.16;

contract EthTxOrderDependenceMinimal {
    address public owner;
    bool public claimed;
    uint public reward = 5; // changed here 

    function EthTxOrderDependenceMinimal() public {
        owner = msg.sender;
    }

    function setReward() public payable {
        require (!claimed);

        require(msg.sender == owner);
        // owner.transfer(reward);
        // reward = msg.value; // changed here 
    }

    function claimReward(uint256 submission) {
        require (!claimed);
        require(submission < 10);

        msg.sender.transfer(reward);
        claimed = true;
    }
}
``` 

# Detecting TOD Issues

When TOD happens, there is one transaction T1 updates a state variable V.
To this end, we define the following Datalog rule.

```datalog
// write the value of Variable var to storage location val in Statement stmt
.decl SSTORE(stmt: Statement, val: Value, var: Variable)

SSTORE(stmt, val, var) :-
  op(stmt,"SSTORE"),
  use(index, stmt, 1), // write to the storage location indexed by the value of Variable index
  use(var, stmt, 2), // write the value of Variable var to the storage location
  value(index, val). // Variable index has value val
```

There is another transaction T2
makes a call whose argument $value is dependent on the state variable V.
Since the argument $value is dependent on V, this transaction reads from V.
we define the following Datalog rule.

```datalog
// read the value of storage location val to Variable var in Statement stmt
.decl SLOAD(stmt: Statement, val: Value, var: Variable)

SLOAD(stmt, val, var) :-
  op(stmt,"SLOAD"),
  use(index, stmt, 1), // read from the storage location indexed by the value of Variable index
  def(var, stmt), // the read result is in Variable var 
  value(index, val). // // Variable index has value val
```


Finally, putting these together, T1 sets the value of x1, and the value 
of x2 is dependent on x1, and T1 writes to storage location val with the value 
of x2. Then, T2 makes a storage load and loads the result into y1. 
Note that SSTORE and SLOAD share the same $val argument. This means the 
associated read and write operations access the same storage location (and the
same state variable). 
The value of y2 is dependent on y1, 
and the value of y2 is used as the argument $value in CALL.


```datalog
todCall(stmt) :-
  setBySource(x1),
  depends(x2, x1),
  SSTORE(_, val, x2),
  SLOAD(_, val, y1),
  depends(y2, y1),
  op_CALL(stmt, _, _, y2, _, _, _, _).
```

# Implementations and Experiments

## Implementation

The analysis is in ./datalog/tod.dl, and it depends on some other files in the
./datalog/lib folder.

## Install the Tool

The requirements for this tool could be generally installed using the following command:

```bash
pip install -r requirements.txt
``` 

Here are more 
[detailed instructions](https://github.com/usyd-blockchain/vandal/wiki/Getting-Started-with-Vandal#writing-analyses) 
from the original repo.


## Benchmarks

A contract with TOD issue: ./myExamples/tod/tod.sol

A contract without TOD issue: ./myExamples/tod/todNot.sol

## Running the experiments

From the root of the project,

```bash
cd myExamples/tod
solc --bin-runtime tod.sol | tail -n 1 > tod.hex
../../bin/analyze.sh tod.hex ../../datalog/tod.dl
```
The result is in the todCall.csv file. Similarly, for the contract without TOD issue,
```
solc --bin-runtime todNot.sol | tail -n 1 > todNot.hex
../../bin/analyze.sh todNot.hex ../../datalog/tod.dl
```
