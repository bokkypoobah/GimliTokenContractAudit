# GimliCrowdsale

Source file [../../sol/GimliCrowdsale.sol](../../sol/GimliCrowdsale.sol).

<br />

<hr />

```javascript
// BK NOTE - Consider using ^0.4.16
pragma solidity ^0.4.11;

// BK Next 2 Ok
import 'SafeMath.sol';
import "GimliToken.sol";

/// @title Gimli Crowdsale Contract.
contract GimliCrowdsale is SafeMath, GimliToken {

    address public constant MULTISIG_WALLET_ADDRESS = 0xd889caA9847F64C77118AD5Ec60291525A3d3939;
    address public constant LOCKED_ADDRESS = 0xabcdefabcdefabcdefabcdefabcdefabcdefabcd;

    // crowdsale
    uint256 public constant CROWDSALE_AMOUNT = 80 * MILLION_GML; // Should not include vested amount
    uint public constant START_DATE = 1400000000; // TODO (epoch timestamp)
    uint public constant END_DATE = 1500000000; // TODO (epoch timestamp)
    uint256 public constant CROWDSALE_PRICE = 700; // 700 GML / ETH
    uint256 public constant VESTING_1_AMOUNT = 15 * MILLION_GML; // TODO
    uint256 public constant VESTING_1_DATE = 1600000000; // TODO (epoch timestamp)
    uint256 public constant VESTING_2_AMOUNT = 15 * MILLION_GML; // TODO
    uint256 public constant VESTING_2_DATE = 1700000000; // TODO (epoch timestamp)
    bool public vesting1Withdrawn = false;
    bool public vesting2Withdrawn = false;
    bool public crowdsaleCanceled = false;
    uint256 public soldAmount;

    /// @notice `msg.sender` invest `msg.value`
    // BK Ok - Default function, payable, receives the participant's contributions
    function() payable {
        // BK Ok
        require(!crowdsaleCanceled);

        // BK Ok - Participant must send non-zero value
        require(msg.value > 0);
        // check date
        // BK Ok 
        require(block.timestamp >= START_DATE && block.timestamp <= END_DATE);

        // calculate and check quantity
        uint256 quantity = safeDiv(safeMul(msg.value, CROWDSALE_PRICE), 10**(18-uint256(decimals)));
        if (safeSub(balances[this], quantity) < 0)
            return;

        // BK Ok
        require(MULTISIG_WALLET_ADDRESS.send(msg.value));

        // update balances
        balances[this] = safeSub(balances[this], quantity);
        balances[msg.sender] = safeAdd(balances[msg.sender], quantity);
        soldAmount = safeAdd(soldAmount, quantity);

        // BK Ok - Log event
        Transfer(this, msg.sender, quantity);
        // BK NOTE - Transfer `msg.value` directly to the wallet, e.g. `crowdsaleWallet.transfer(msg.value);`
    }

    /// @notice returns non-sold tokens to owner
    // BK Ok
    function  closeCrowdsale() onlyOwner {
        // check if closable
        require(block.timestamp > END_DATE || crowdsaleCanceled || balances[this] == 0);

        // enable token transfer
        transferable = true;

        // update balances
        if (balances[this] > 0) {
            // BK Ok
            uint256 amount = balances[this];
            // BK Ok - Transfer unsold tokens to the owner
            balances[owner] = safeAdd(balances[owner], amount);
            // BK Ok
            balances[this] = 0;

            // BK Ok
            Transfer(this, owner, amount);
        }
    }

    /// @notice Terminate the crowdsale before END_DATE
    function cancelCrowdsale() onlyOwner {
        crowdsaleCanceled = true;
    }

    /// @notice Pre-allocate tokens to advisor or partner
    /// @param _to The pre-allocation destination
    /// @param _value The amount of token to be allocated
    function preAllocate(address _to, uint256 _value) onlyOwner {
        require(block.timestamp < START_DATE);

        balances[_to] = safeAdd(balances[_to], _value);
        balances[this] = safeSub(balances[this], _value);
        soldAmount = safeAdd(soldAmount, _value);

        Transfer(this, _to, _value);
    }

    /// @notice Send vested amount to _destination
    /// @param _destination The address of the recipient
    /// @return Whether the release was successful or not
    function releaseVesting(address _destination) onlyOwner returns (bool success) {
        if (block.timestamp > VESTING_1_DATE && vesting1Withdrawn == false) {
            balances[_destination] = safeAdd(balances[_destination], VESTING_1_AMOUNT);
            balances[LOCKED_ADDRESS] = safeSub(balances[LOCKED_ADDRESS], VESTING_1_AMOUNT);
            vesting1Withdrawn = true;
            Transfer(LOCKED_ADDRESS, _destination, VESTING_1_AMOUNT);
            return true;
        }
        if (block.timestamp > VESTING_2_DATE && vesting2Withdrawn == false) {
            balances[_destination] = safeAdd(balances[_destination], VESTING_2_AMOUNT);
            balances[LOCKED_ADDRESS] = safeSub(balances[LOCKED_ADDRESS], VESTING_2_AMOUNT);
            vesting2Withdrawn = true;
            Transfer(LOCKED_ADDRESS, _destination, VESTING_2_AMOUNT);
            return true;
        }
        return false;
    }

    /// @notice transfer out any accidentally sent ERC20 tokens
    /// @param tokenAddress Address of the ERC20 contract
    /// @param amount The amount of token to be transfered
    function transferOtherERC20Token(address tokenAddress, uint amount)
      onlyOwner returns (bool success)
    {
        // can't be used for GIM token
        require(tokenAddress != address(this));
        return ERC20(tokenAddress).transfer(owner, amount);
    }
}

```