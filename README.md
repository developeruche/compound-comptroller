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

3. MarketExited: This event would be emitted when an account exits a market

4. NewCloseFactor: Emitted when close factor is changed by admin (remember this account can only be done by the admin).

5. NewLiquidationIncentive: Emitted when liquidation incentive is changed by admin (the function emitting the event can only be call by the admin).

6. NewPriceOracle: The event would be emitted when the address on the price oracle is changed. The parameter emitted during this event is the address of the old oracle and the address of the new price oracle.

7. NewPauseGuardian: this event would be emitted when the account that can call a pause action is changed.

8. ActionPaused: the event is emitted when a pause is called on the contract.

9. CompBorrowSpeedUpdated: this function is emitted when a new borrow-side COMP speed is calculated for a market

10. CompSupplySpeedUpdated: This event is emitted when a new supply-side COMP speed is calculated for a market

11. ContributorCompSpeedUpdated: This event is emitted when a new COMP speed is set for a contributor

12. DistributedSupplierComp: This event is emitted when COMP is distrubuted to a supplier 

13. DistributedBorrowerComp: This event is emitted when COMP is distributed to a borrower

14. NewBorrowCap: This function is emitted when the borrow cap of a for cToken is changed

15. NewBorrowCapGuardian: This event is emitted when the borrow cap of account is changed

16. CompGranted: Tis event is emitted when COMP is granted by the admin

17. CompAccruedAdjusted: This event is logged when the COMP accured by a user has been manually adjusted 

18. CompReceivableUpdated: This event is emitted when the COMP which can be recieved by a user is updated





## Function

1. getAssetsIn(address _account): 
This is a view function that returns the assets of the address passed to the function.

```solidity 
    function getAssetsIn(address account) external view returns (CToken[] memory) {
        CToken[] memory assetsIn = accountAssets[account];

        return assetsIn;
    }
```

2. checkMembership(address account, CToken cToken): 
This function is a view function which true if the given address *account* is in the given market *cToken*

```solidity 
    function checkMembership(address account, CToken cToken) external view returns (bool) {
        return markets[address(cToken)].accountMembership[account];
    }
```


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

    function addToMarketInternal(CToken cToken, address borrower) internal returns (Error) {
        Market storage marketToJoin = markets[address(cToken)];

        if (!marketToJoin.isListed) { 
            return Error.MARKET_NOT_LISTED;
        }

        if (marketToJoin.accountMembership[borrower] == true) {
            return Error.NO_ERROR;
        }

        marketToJoin.accountMembership[borrower] = true;
        accountAssets[borrower].push(cToken);

        emit MarketEntered(cToken, borrower);

        return Error.NO_ERROR;
    }
    
```

enterMarket: This function is used to add asset that would be used when calculating an account's liquidity. *cToken* is a dynamic array of addressees of cToken market to be enabled. This function runs a for loop on the array and calls the *addToMarketInternal* function passing the cToken address and the current caller *msg.sender* who is same as the borrower in the compound's context. The *addToMarketInternal* function would add the market to the borrower's "assets in" for liquidity calculations and returns Success indicator for whether the market was entered



4. exitMarket(address cTokenAddress):

```solidity 
    function exitMarket(address cTokenAddress) override external returns (uint) {
        CToken cToken = CToken(cTokenAddress);
        (uint oErr, uint tokensHeld, uint amountOwed, ) = cToken.getAccountSnapshot(msg.sender);
        require(oErr == 0, "exitMarket: getAccountSnapshot failed");
        if (amountOwed != 0) {
            return fail(Error.NONZERO_BORROW_BALANCE, FailureInfo.EXIT_MARKET_BALANCE_OWED);
        }
        uint allowed = redeemAllowedInternal(cTokenAddress, msg.sender, tokensHeld);
        if (allowed != 0) {
            return failOpaque(Error.REJECTION, FailureInfo.EXIT_MARKET_REJECTION, allowed);
        }
        Market storage marketToExit = markets[address(cToken)];
        if (!marketToExit.accountMembership[msg.sender]) {
            return uint(Error.NO_ERROR);
        }
        delete marketToExit.accountMembership[msg.sender];
        CToken[] memory userAssetList = accountAssets[msg.sender];
        uint len = userAssetList.length;
        uint assetIndex = len;
        for (uint i = 0; i < len; i++) {
            if (userAssetList[i] == cToken) {
                assetIndex = i;
                break;
            }
        }
        assert(assetIndex < len);
        CToken[] storage storedList = accountAssets[msg.sender];
        storedList[assetIndex] = storedList[storedList.length - 1];
        storedList.pop();
        emit MarketExited(cToken, msg.sender);
        return uint(Error.NO_ERROR);
    }
```

This function is used to exit the market, when an account exit a market, it would not be used as a base for calculating the account's liquidation. When calling this function, the *cTokenAddress* must be provided because this is the address of the cToken whose market is to be exited.


5. mintAllowed(address cToken, address minter, uint mintAmount): 

```solidity 
    function mintAllowed(address cToken, address minter, uint mintAmount) override external returns (uint) {
        require(!mintGuardianPaused[cToken], "mint is paused");
        minter;
        mintAmount;
        if (!markets[cToken].isListed) {
            return uint(Error.MARKET_NOT_LISTED);
        }
        updateCompSupplyIndex(cToken);
        distributeSupplierComp(cToken, minter);
        return uint(Error.NO_ERROR);
    }
```

This function check if the address calling the function is allowed to mint token in a given market 

*cToken* this is the address of the market
*minter* this is the address the token is to be minted to 
*mintAmount* This is the amount that is to be minted


6. mintVerify(address cToken, address minter, uint actualMintAmount, uint mintTokens):

```solidity 
    function mintVerify(address cToken, address minter, uint actualMintAmount, uint mintTokens) override external {
        cToken;
        minter;
        actualMintAmount;
        mintTokens;
        if (false) {
            maxAssets = maxAssets;
        }
    }
```

 Validates mint and reverts on rejection


7. redeemAllowed(address cToken, address redeemer, uint redeemTokens);

```solidity 
    function redeemAllowed(address cToken, address redeemer, uint redeemTokens) override external returns (uint) {
        uint allowed = redeemAllowedInternal(cToken, redeemer, redeemTokens);
        if (allowed != uint(Error.NO_ERROR)) {
            return allowed;
        }
        updateCompSupplyIndex(cToken);
        distributeSupplierComp(cToken, redeemer);
        return uint(Error.NO_ERROR);
    }
```

This function runs a  check on a account *redeemer* relative to *cToken* and *redeemToken* to see if the account can redeem those funds. The function that does all the heavy lifting is the *redeemAllowedInternal*


8.   redeemVerify(address cToken, address redeemer, uint redeemAmount, uint redeemTokens):

```solidity 
    function redeemVerify(address cToken, address redeemer, uint redeemAmount, uint redeemTokens) override external {
        cToken;
        redeemer;
        if (redeemTokens == 0 && redeemAmount > 0) {
            revert("redeemTokens zero");
        }
    }

```

Validate redeem request a and revert on rejection.


9. borrowAllowed(address cToken, address borrower, uint borrowAmount):

```solidity 
    function borrowAllowed(address cToken, address borrower, uint borrowAmount) override external returns (uint) {
        require(!borrowGuardianPaused[cToken], "borrow is paused");
        if (!markets[cToken].isListed) {
            return uint(Error.MARKET_NOT_LISTED);
        }
        if (!markets[cToken].accountMembership[borrower]) {
            require(msg.sender == cToken, "sender must be cToken");
            Error err = addToMarketInternal(CToken(msg.sender), borrower);
            if (err != Error.NO_ERROR) {
                return uint(err);
            }
            assert(markets[cToken].accountMembership[borrower]);
        }
        if (oracle.getUnderlyingPrice(CToken(cToken)) == 0) {
            return uint(Error.PRICE_ERROR);
        }
        uint borrowCap = borrowCaps[cToken];
        if (borrowCap != 0) {
            uint totalBorrows = CToken(cToken).totalBorrows();
            uint nextTotalBorrows = add_(totalBorrows, borrowAmount);
            require(nextTotalBorrows < borrowCap, "market borrow cap reached");
        }
        (Error err, , uint shortfall) = getHypotheticalAccountLiquidityInternal(borrower, CToken(cToken), 0, borrowAmount);
        if (err != Error.NO_ERROR) {
            return uint(err);
        }
        if (shortfall > 0) {
            return uint(Error.INSUFFICIENT_LIQUIDITY);
        }
        Exp memory borrowIndex = Exp({mantissa: CToken(cToken).borrowIndex()});
        updateCompBorrowIndex(cToken, borrowIndex);
        distributeBorrowerComp(cToken, borrower, borrowIndex);
        return uint(Error.NO_ERROR);
    }

```

This function is called to check if an account is allowed to borrow an underlined asset of a given market. This function takes these parameter *cToken*, *borrower* and *borrow*. 

10. repayBorrowAllowed:

```solidity 
      function repayBorrowAllowed(
        address cToken,
        address payer,
        address borrower,
        uint repayAmount) override external returns (uint) {
        payer;
        borrower;
        repayAmount;
        if (!markets[cToken].isListed) {
            return uint(Error.MARKET_NOT_LISTED);
        }
        Exp memory borrowIndex = Exp({mantissa: CToken(cToken).borrowIndex()});
        updateCompBorrowIndex(cToken, borrowIndex);
        distributeBorrowerComp(cToken, borrower, borrowIndex);
        return uint(Error.NO_ERROR);
    }

```

When this function is called, it runs to see that the borrower is allowed to repay a borrow in a given market


11. liquidateBorrowAllowed: 


```solidity
    function liquidateBorrowAllowed(
        address cTokenBorrowed,
        address cTokenCollateral,
        address liquidator,
        address borrower,
        uint repayAmount) override external returns (uint) {
        liquidator;
        if (!markets[cTokenBorrowed].isListed || !markets[cTokenCollateral].isListed) {
            return uint(Error.MARKET_NOT_LISTED);
        }
        uint borrowBalance = CToken(cTokenBorrowed).borrowBalanceStored(borrower);
        if (isDeprecated(CToken(cTokenBorrowed))) {
            require(borrowBalance >= repayAmount, "Can not repay more than the total borrow");
        } else {
            (Error err, , uint shortfall) = getAccountLiquidityInternal(borrower);
            if (err != Error.NO_ERROR) {
                return uint(err);
            }
            if (shortfall == 0) {
                return uint(Error.INSUFFICIENT_SHORTFALL);
            }
            uint maxClose = mul_ScalarTruncate(Exp({mantissa: closeFactorMantissa}), borrowBalance);
            if (repayAmount > maxClose) {
                return uint(Error.TOO_MUCH_REPAY);
            }
        }
        return uint(Error.NO_ERROR);
    }
```

This function is call to validate that a position can be liqudated 


12. seizeAllowed: 

```solidity
    function seizeAllowed(
        address cTokenCollateral,
        address cTokenBorrowed,
        address liquidator,
        address borrower,
        uint seizeTokens) override external returns (uint) {
        require(!seizeGuardianPaused, "seize is paused");
        seizeTokens;
        if (!markets[cTokenCollateral].isListed || !markets[cTokenBorrowed].isListed) {
            return uint(Error.MARKET_NOT_LISTED);
        }
        if (CToken(cTokenCollateral).comptroller() != CToken(cTokenBorrowed).comptroller()) {
            return uint(Error.COMPTROLLER_MISMATCH);
        }
        updateCompSupplyIndex(cTokenCollateral);
        distributeSupplierComp(cTokenCollateral, borrower);
        distributeSupplierComp(cTokenCollateral, liquidator);
        return uint(Error.NO_ERROR);
    }
```

The function is call to see if an asset can be seized or not.


13. transferAllowed:

```solidity
    function transferAllowed(address cToken, address src, address dst, uint transferTokens) override external returns (uint) {
        require(!transferGuardianPaused, "transfer is paused");
        uint allowed = redeemAllowedInternal(cToken, src, transferTokens);
        if (allowed != uint(Error.NO_ERROR)) {
            return allowed;
        }
        updateCompSupplyIndex(cToken);
        distributeSupplierComp(cToken, src);
        distributeSupplierComp(cToken, dst);
        return uint(Error.NO_ERROR);
    }
```

This function when ran, runs a check to see if transfer is allowed at the time of that call


14. getAccountLiquidity: 

```solidity
    function getAccountLiquidity(address account) public view returns (uint, uint, uint) {
        (Error err, uint liquidity, uint shortfall) = getHypotheticalAccountLiquidityInternal(account, CToken(address(0)), 0, 0);

        return (uint(err), liquidity, shortfall);
    }

```

This function calculates the liquidity associated to an account and return it.


15.  getHypotheticalAccountLiquidity:

```solidity

    function getHypotheticalAccountLiquidity(
        address account,
        address cTokenModify,
        uint redeemTokens,
        uint borrowAmount) public view returns (uint, uint, uint) {
        (Error err, uint liquidity, uint shortfall) = getHypotheticalAccountLiquidityInternal(account, CToken(cTokenModify), redeemTokens, borrowAmount);
        return (uint(err), liquidity, shortfall);
    }
```

This is a very important function that predicts the liquidity of an account if the following *amount* was borrowed or redeemed.


16. getHypotheticalAccountLiquidityInternal:

```solidity
    function getHypotheticalAccountLiquidityInternal(
        address account,
        CToken cTokenModify,
        uint redeemTokens,
        uint borrowAmount) internal view returns (Error, uint, uint) {
        AccountLiquidityLocalVars memory vars;
        uint oErr;
        CToken[] memory assets = accountAssets[account];
        for (uint i = 0; i < assets.length; i++) {
            CToken asset = assets[i];
            (oErr, vars.cTokenBalance, vars.borrowBalance, vars.exchangeRateMantissa) = asset.getAccountSnapshot(account);
            if (oErr != 0) {
                upgrades
                return (Error.SNAPSHOT_ERROR, 0, 0);
            }
            vars.collateralFactor = Exp({mantissa: markets[address(asset)].collateralFactorMantissa});
            vars.exchangeRate = Exp({mantissa: vars.exchangeRateMantissa});
            vars.oraclePriceMantissa = oracle.getUnderlyingPrice(asset);
            if (vars.oraclePriceMantissa == 0) {
                return (Error.PRICE_ERROR, 0, 0);
            }
            vars.oraclePrice = Exp({mantissa: vars.oraclePriceMantissa});
            vars.tokensToDenom = mul_(mul_(vars.collateralFactor, vars.exchangeRate), vars.oraclePrice);
            vars.sumCollateral = mul_ScalarTruncateAddUInt(vars.tokensToDenom, vars.cTokenBalance, vars.sumCollateral);
            vars.sumBorrowPlusEffects = mul_ScalarTruncateAddUInt(vars.oraclePrice, vars.borrowBalance, vars.sumBorrowPlusEffects);
            if (asset == cTokenModify) {
                vars.sumBorrowPlusEffects = mul_ScalarTruncateAddUInt(vars.tokensToDenom, redeemTokens, vars.sumBorrowPlusEffects);
                vars.sumBorrowPlusEffects = mul_ScalarTruncateAddUInt(vars.oraclePrice, borrowAmount, vars.sumBorrowPlusEffects);
            }
        }
        if (vars.sumCollateral > vars.sumBorrowPlusEffects) {
            return (Error.NO_ERROR, vars.sumCollateral - vars.sumBorrowPlusEffects, 0);
        } else {
            return (Error.NO_ERROR, 0, vars.sumBorrowPlusEffects - vars.sumCollateral);
        }
    }
```


This function carries the heavy lifting with respect to the *getHypotheticalAccountLiquidityInternal* and it could return possible error code or   hypothetical account liquidity in excess of collateral requirements or hypothetical account shortfall below collateral requirements


17. liquidateCalculateSeizeTokens:

```solidity 
    function liquidateCalculateSeizeTokens(address cTokenBorrowed, address cTokenCollateral, uint actualRepayAmount) override external view returns (uint, uint) {
        uint priceBorrowedMantissa = oracle.getUnderlyingPrice(CToken(cTokenBorrowed));
        uint priceCollateralMantissa = oracle.getUnderlyingPrice(CToken(cTokenCollateral));
        if (priceBorrowedMantissa == 0 || priceCollateralMantissa == 0) {
            return (uint(Error.PRICE_ERROR), 0);
        }
        uint exchangeRateMantissa = CToken(cTokenCollateral).exchangeRateStored();
        uint seizeTokens;
        Exp memory numerator;
        Exp memory denominator;
        Exp memory ratio;
        numerator = mul_(Exp({mantissa: liquidationIncentiveMantissa}), Exp({mantissa: priceBorrowedMantissa}));
        denominator = mul_(Exp({mantissa: priceCollateralMantissa}), Exp({mantissa: exchangeRateMantissa}));
        ratio = div_(numerator, denominator);
        seizeTokens = mul_ScalarTruncate(ratio, actualRepayAmount);
        return (uint(Error.NO_ERROR), seizeTokens);
    }
```

This is a view function that is used to calculate the number of tokens of collateral asset to seize given an underlying amount. This function would return an errCode and number of cTokenColleteral tokens to be seized in a liquidation.


18. _setPriceOracle:

```solidity
    function _setPriceOracle(PriceOracle newOracle) public returns (uint) {
        if (msg.sender != admin) {
            return fail(Error.UNAUTHORIZED, FailureInfo.SET_PRICE_ORACLE_OWNER_CHECK);
        }
        PriceOracle oldOracle = oracle;
        oracle = newOracle;
        emit NewPriceOracle(oldOracle, newOracle);
        return uint(Error.NO_ERROR);
    }
```

This is an important function that can be called only by the admin, the function would be change the address of the current price oracle.


19. _setCloseFactor: 

```solidity 
    function _setCloseFactor(uint newCloseFactorMantissa) external returns (uint) {
    	require(msg.sender == admin, "only admin can set close factor");
        uint oldCloseFactorMantissa = closeFactorMantissa;
        closeFactorMantissa = newCloseFactorMantissa;
        emit NewCloseFactor(oldCloseFactorMantissa, closeFactorMantissa);
        return uint(Error.NO_ERROR);
    }
```

This is a write function which is called by the admin to edit the closeFactor. The closeFactor is a bases on which liquidation can occur.

20. _setCollateralFactor:

```solidity 
    function _setCollateralFactor(CToken cToken, uint newCollateralFactorMantissa) external returns (uint) {
        if (msg.sender != admin) {
            return fail(Error.UNAUTHORIZED, FailureInfo.SET_COLLATERAL_FACTOR_OWNER_CHECK);
        }
        Market storage market = markets[address(cToken)];
        if (!market.isListed) {
            return fail(Error.MARKET_NOT_LISTED, FailureInfo.SET_COLLATERAL_FACTOR_NO_EXISTS);
        }
        Exp memory newCollateralFactorExp = Exp({mantissa: newCollateralFactorMantissa});
        Exp memory highLimit = Exp({mantissa: collateralFactorMaxMantissa});
        if (lessThanExp(highLimit, newCollateralFactorExp)) {
            return fail(Error.INVALID_COLLATERAL_FACTOR, FailureInfo.SET_COLLATERAL_FACTOR_VALIDATION);
        }
        if (newCollateralFactorMantissa != 0 && oracle.getUnderlyingPrice(cToken) == 0) {
            return fail(Error.PRICE_ERROR, FailureInfo.SET_COLLATERAL_FACTOR_WITHOUT_PRICE);
        }
        uint oldCollateralFactorMantissa = market.collateralFactorMantissa;
        market.collateralFactorMantissa = newCollateralFactorMantissa;
        emit NewCollateralFactor(cToken, oldCollateralFactorMantissa, newCollateralFactorMantissa);
        return uint(Error.NO_ERROR);
    }
```

This is a write function that can be called only by the admin the function updates the state variable *market.collateralFactorMantissa* this is the factor of cToken that would be minted to the borrow based on the amount on collaterial provided. 


21. _setLiquidationIncentive:

```solidity 

    function _setLiquidationIncentive(uint newLiquidationIncentiveMantissa) external returns (uint) {
        if (msg.sender != admin) {
            return fail(Error.UNAUTHORIZED, FailureInfo.SET_LIQUIDATION_INCENTIVE_OWNER_CHECK);
        }
        uint oldLiquidationIncentiveMantissa = liquidationIncentiveMantissa;
        liquidationIncentiveMantissa = newLiquidationIncentiveMantissa;
        emit NewLiquidationIncentive(oldLiquidationIncentiveMantissa, newLiquidationIncentiveMantissa);
        return uint(Error.NO_ERROR);
    }

```

This is also another write function gated to be called by only the admin. THis function would change the incentive factor provided when liquidity is added.


22. _supportMarket: 

```solidity 
    function _supportMarket(CToken cToken) external returns (uint) {
        if (msg.sender != admin) {
            return fail(Error.UNAUTHORIZED, FailureInfo.SUPPORT_MARKET_OWNER_CHECK);
        }
        if (markets[address(cToken)].isListed) {
            return fail(Error.MARKET_ALREADY_LISTED, FailureInfo.SUPPORT_MARKET_EXISTS);
        }
        cToken.isCToken();
        Market storage newMarket = markets[address(cToken)];
        newMarket.isListed = true;
        newMarket.isComped = false;
        newMarket.collateralFactorMantissa = 0;
        _addMarketInternal(address(cToken));
        _initializeMarket(address(cToken));
        emit MarketListed(cToken);
        return uint(Error.NO_ERROR);
    }


```

This is a writing function to be called only by an admin. This function would add a new market to the protocol. Recall market in the compound protocol is represented by *cToken*


23. _setMarketBorrowCaps:


```solidity

    function _setMarketBorrowCaps(CToken[] calldata cTokens, uint[] calldata newBorrowCaps) external {
    	require(msg.sender == admin || msg.sender == borrowCapGuardian, "only admin or borrow cap guardian can set borrow caps");

        uint numMarkets = cTokens.length;
        uint numBorrowCaps = newBorrowCaps.length;

        require(numMarkets != 0 && numMarkets == numBorrowCaps, "invalid input");

        for(uint i = 0; i < numMarkets; i++) {
            borrowCaps[address(cTokens[i])] = newBorrowCaps[i];
            emit NewBorrowCap(cTokens[i], newBorrowCaps[i]);
        }
    }


```

Set the given borrow caps for the given cToken markets. Borrowing that brings total borrows to or above borrow cap will revert.



24. _setBorrowCapGuardian:

```solidity
    function _setBorrowCapGuardian(address newBorrowCapGuardian) external {
        require(msg.sender == admin, "only admin can set borrow cap guardian");
        address oldBorrowCapGuardian = borrowCapGuardian;
        borrowCapGuardian = newBorrowCapGuardian;
        emit NewBorrowCapGuardian(oldBorrowCapGuardian, newBorrowCapGuardian);
    }

```

This is a write function that changes the account that can change the borrowCap of the protocol.

25. _setPauseGuardian:

```solidity 
    function _setPauseGuardian(address newPauseGuardian) public returns (uint) {
        if (msg.sender != admin) {
            return fail(Error.UNAUTHORIZED, FailureInfo.SET_PAUSE_GUARDIAN_OWNER_CHECK);
        }
        address oldPauseGuardian = pauseGuardian;
        pauseGuardian = newPauseGuardian;
        emit NewPauseGuardian(oldPauseGuardian, pauseGuardian);
        return uint(Error.NO_ERROR);
    }

```

This is a function to set the account that can call a pause function on this contract.


26. _setMintPaused: 

```solidity 
    function _setMintPaused(CToken cToken, bool state) public returns (bool) {
        require(markets[address(cToken)].isListed, "cannot pause a market that is not listed");
        require(msg.sender == pauseGuardian || msg.sender == admin, "only pause guardian and admin can pause");
        require(msg.sender == admin || state == true, "only admin can unpause");

        mintGuardianPaused[address(cToken)] = state;
        emit ActionPaused(cToken, "Mint", state);
        return state;
    }
```

This is a write function that can be called by only the pause guardian. This function would make the minting operation paused.


27. _setBorrowPaused:

```solidity 
    function _setBorrowPaused(CToken cToken, bool state) public returns (bool) {
        require(markets[address(cToken)].isListed, "cannot pause a market that is not listed");
        require(msg.sender == pauseGuardian || msg.sender == admin, "only pause guardian and admin can pause");
        require(msg.sender == admin || state == true, "only admin can unpause");

        borrowGuardianPaused[address(cToken)] = state;
        emit ActionPaused(cToken, "Borrow", state);
        return state;
    }
```

This write  function can only be called by the pause guardian. This function would pause the borrowing operation.


28. _setTransferPaused:

```solidity 

    function _setTransferPaused(bool state) public returns (bool) {
        require(msg.sender == pauseGuardian || msg.sender == admin, "only pause guardian and admin can pause");
        require(msg.sender == admin || state == true, "only admin can unpause");

        transferGuardianPaused = state;
        emit ActionPaused("Transfer", state);
        return state;
    }


```

This function would halted the transfer operation 


29. _setSeizePaused:

```solidity 

    function _setSeizePaused(bool state) public returns (bool) {
        require(msg.sender == pauseGuardian || msg.sender == admin, "only pause guardian and admin can pause");
        require(msg.sender == admin || state == true, "only admin can unpause");

        seizeGuardianPaused = state;
        emit ActionPaused("Seize", state);
        return state;
    }
```


This function when called by the guardian would halt the seize operation 


30. setCompSpeedInternal:


```solidity 
    function setCompSpeedInternal(CToken cToken, uint supplySpeed, uint borrowSpeed) internal {
        Market storage market = markets[address(cToken)];
        require(market.isListed, "comp market is not listed");
        if (compSupplySpeeds[address(cToken)] != supplySpeed) {
            updateCompSupplyIndex(address(cToken));
            compSupplySpeeds[address(cToken)] = supplySpeed;
            emit CompSupplySpeedUpdated(cToken, supplySpeed);
        }
        if (compBorrowSpeeds[address(cToken)] != borrowSpeed) {
            Exp memory borrowIndex = Exp({mantissa: cToken.borrowIndex()});
            updateCompBorrowIndex(address(cToken), borrowIndex);
            compBorrowSpeeds[address(cToken)] = borrowSpeed;
            emit CompBorrowSpeedUpdated(cToken, borrowSpeed);
        }
    }

```

The function is used to set COMP speed for a single market 

31. updateCompSupplyIndex:

```solidity 
    function updateCompSupplyIndex(address cToken) internal {
        CompMarketState storage supplyState = compSupplyState[cToken];
        uint supplySpeed = compSupplySpeeds[cToken];
        uint32 blockNumber = safe32(getBlockNumber(), "block number exceeds 32 bits");
        uint deltaBlocks = sub_(uint(blockNumber), uint(supplyState.block));
        if (deltaBlocks > 0 && supplySpeed > 0) {
            uint supplyTokens = CToken(cToken).totalSupply();
            uint compAccrued = mul_(deltaBlocks, supplySpeed);
            Double memory ratio = supplyTokens > 0 ? fraction(compAccrued, supplyTokens) : Double({mantissa: 0});
            supplyState.index = safe224(add_(Double({mantissa: supplyState.index}), ratio).mantissa, "new index exceeds 224 bits");
            supplyState.block = blockNumber;
        } else if (deltaBlocks > 0) {
            supplyState.block = blockNumber;
        }
    }
```

Accrue COMP to the market by updating the supply index


32. updateCompBorrowIndex: 

```solidity 
    function updateCompBorrowIndex(address cToken, Exp memory marketBorrowIndex) internal {
        CompMarketState storage borrowState = compBorrowState[cToken];
        uint borrowSpeed = compBorrowSpeeds[cToken];
        uint32 blockNumber = safe32(getBlockNumber(), "block number exceeds 32 bits");
        uint deltaBlocks = sub_(uint(blockNumber), uint(borrowState.block));
        if (deltaBlocks > 0 && borrowSpeed > 0) {
            uint borrowAmount = div_(CToken(cToken).totalBorrows(), marketBorrowIndex);
            uint compAccrued = mul_(deltaBlocks, borrowSpeed);
            Double memory ratio = borrowAmount > 0 ? fraction(compAccrued, borrowAmount) : Double({mantissa: 0});
            borrowState.index = safe224(add_(Double({mantissa: borrowState.index}), ratio).mantissa, "new index exceeds 224 bits");
            borrowState.block = blockNumber;
        } else if (deltaBlocks > 0) {
            borrowState.block = blockNumber;
        }
    }
```

 Accrue COMP to the market by updating the borrow index


 33. distributeSupplierComp:

 ```solidity 
    function distributeSupplierComp(address cToken, address supplier) internal {
        CompMarketState storage supplyState = compSupplyState[cToken];
        uint supplyIndex = supplyState.index;
        uint supplierIndex = compSupplierIndex[cToken][supplier];
        compSupplierIndex[cToken][supplier] = supplyIndex;
        if (supplierIndex == 0 && supplyIndex >= compInitialIndex) {
            supplierIndex = compInitialIndex;
        }
        Double memory deltaIndex = Double({mantissa: sub_(supplyIndex, supplierIndex)});
        uint supplierTokens = CToken(cToken).balanceOf(supplier);
        uint supplierDelta = mul_(supplierTokens, deltaIndex);
        uint supplierAccrued = add_(compAccrued[supplier], supplierDelta);
        compAccrued[supplier] = supplierAccrued;
        emit DistributedSupplierComp(CToken(cToken), supplier, supplierDelta, supplyIndex);
    }
 ```
Calculate COMP accrued by a supplier and possibly transfer it to them

34. distributeBorrowerComp:

```solidity 
    function distributeBorrowerComp(address cToken, address borrower, Exp memory marketBorrowIndex) internal {
        CompMarketState storage borrowState = compBorrowState[cToken];
        uint borrowIndex = borrowState.index;
        uint borrowerIndex = compBorrowerIndex[cToken][borrower];
        compBorrowerIndex[cToken][borrower] = borrowIndex;
        if (borrowerIndex == 0 && borrowIndex >= compInitialIndex) {
            borrowerIndex = compInitialIndex;
        }
        Double memory deltaIndex = Double({mantissa: sub_(borrowIndex, borrowerIndex)});
        uint borrowerAmount = div_(CToken(cToken).borrowBalanceStored(borrower), marketBorrowIndex);
        uint borrowerDelta = mul_(borrowerAmount, deltaIndex);
        uint borrowerAccrued = add_(compAccrued[borrower], borrowerDelta);
        compAccrued[borrower] = borrowerAccrued;
        emit DistributedBorrowerComp(CToken(cToken), borrower, borrowerDelta, borrowIndex);
    }
```

Calculate COMP accrued by a borrower and possibly transfer it to them

35. updateContributorRewards:

```solidity 
    function updateContributorRewards(address contributor) public {
        uint compSpeed = compContributorSpeeds[contributor];
        uint blockNumber = getBlockNumber();
        uint deltaBlocks = sub_(blockNumber, lastContributorBlock[contributor]);
        if (deltaBlocks > 0 && compSpeed > 0) {
            uint newAccrued = mul_(deltaBlocks, compSpeed);
            uint contributorAccrued = add_(compAccrued[contributor], newAccrued);

            compAccrued[contributor] = contributorAccrued;
            lastContributorBlock[contributor] = blockNumber;
        }
    }

```

This function when ran updates the reward been given to the contributor.


36. claimComp;

```solidity 
    function claimComp(address holder) public {
        return claimComp(holder, allMarkets);
    }
```

The function when ran would claim an account's COMP reward from all market 


37. grantCompInternal:

```solidity 

    function grantCompInternal(address user, uint amount) internal returns (uint) {
        Comp comp = Comp(getCompAddress());
        uint compRemaining = comp.balanceOf(address(this));
        if (amount > 0 && amount <= compRemaining) {
            comp.transfer(user, amount);
            return 0;
        }
        return amount;
    }

```
The function is used to transfer COMP token to the user specified 


38. getAllMarkets:

```solidity 

    function getAllMarkets() public view returns (CToken[] memory) {
        return allMarkets;
    }

```


This function when ran would return all the market supported by the protocol


39. isDeprecated:


```solidity 
    function isDeprecated(CToken cToken) public view returns (bool) {
        return
            markets[address(cToken)].collateralFactorMantissa == 0 &&
            borrowGuardianPaused[address(cToken)] == true &&
            cToken.reserveFactorMantissa() == 1e18
        ;
    }

```

THis function is used to see if a market *cToken* is deprecated.


40. getCompAddress:

```solidity 

    function getCompAddress() virtual public view returns (address) {
        return 0xc00e94Cb662C3520282E6f5717214004A7f26888;
    }
    
```


This function would return the address of the COMP token.