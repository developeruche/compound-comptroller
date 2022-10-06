## COMPOUND COMPTROLLER CONTRACT REVIEW

The Compound cTokens, which are self-contained borrowing and lending contracts.

The risk model contract, which validates permissible user actions and disallows actions if they do not fit certain risk parameters. For instance, the Comptroller enforces that each borrowing user must maintain a sufficient collateral balance across all cTokens.


The compound protocol Comptroller serves as the risk management section of the compound protocol. The comptroller determines how much underline collateral a user is required to maintain and whether a user have been liquidated and by how much. Every time a user interacts with the cToken, the Comptroller is asked to approve or deny the transaction.


The Comptroller contract's implementation is an upgradeable proxy. The Unitroller proxies all logic to the Comptroller implementation contract but storage values are set on the Unitroller. This process is carried out using delegate call. Functions from the Comptroller's abi are called on the Unitroller then the call is proxied to the Comptroller implementation.


### Import Statement 
This Comptroller made some imports which was used to get this done and they would be explained briefly.


```solidity
    import "./CToken.sol";
    import "./ErrorReporter.sol";
    import "./PriceOracle.sol";
    import "./ComptrollerInterface.sol";
    import "./ComptrollerStorage.sol";
    import "./Unitroller.sol";
    import "./Governance/Comp.sol";
```

1. CToken.sol: 
Each asset supported by the Compound Protocol is integrated through a cToken contract, which is an EIP-20 compliant representation of balances supplied to the protocol. 

2. ErrorReporter.sol:
This solidity file holds ComptrollerErrorReporter and TokenErrorReporter this contracts house custom errors and event used in the Comptroller contract.

3. PriceOracle.sol: This is an abstract contract having one function used to interact with the price oracle to get the price of cToken.

4. ComptrollerInterface.sol: This is the interface of the Comptroller contract which would be used as a guide with which all the function must be implemented.

5. ComptrollerStorage.sol: This contract holds all the state variable used on the Comptroller contract. Since the Comptroller contract is upgradeable, the arrangement of the state variable is very important because if the state variable layout is changed, the contract storage is useless.

6. Unitroller.sol: This is the proxy contract that proxies most logic call to the comptroller contract

7. Comp.sol: This is the Compound protocol governance token


### Inheritance 

1. ComptrollerV7Storage: This contract holds all the state variables. Since the contract is an upgradeable contract the state storage layout must be well maintained. The compound used inheritance strategy to make this seemless.

2. ComptrollerInterface: All functions in this inherited interface must be implemented

3. ComptrollerErrorReporter: This contract handles errors using custom errors

4. ExponentialNoError: Exp is a struct which stores decimals with a fixed precision of 18 decimal places.


## Events 

1. MarketListed: This event would be emitted when an admin supports a new market

2. MarketEntered: This event would be emitted when a an account enters the market





## Function

1. getAssetsIn(address _account): 

2. checkMembership(address account, CToken cToken): 

3.  enterMarkets(address[] memory cTokens): 

```solidity 

    function enterMarkets(address[] memory cTokens) override public returns (uint[] memory) {
        uint len = cTokens.length;

        uint[] memory results = new uint[](len);
        for (uint i = 0; i < len; i++) {
            CToken cToken = CToken(cTokens[i]);

            results[i] = uint(addToMarketInternal(cToken, msg.sender));
        }

        return results;
    }
    
```

