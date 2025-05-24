# bloountech-auditoria
Auditoria comunitária do contrato BloounTech (BEP-20)
# BloounTech (BLOOUN)
Contrato inteligente da criptomoeda BloounTech implantado na BNB Smart Chain.

## Descrição
Token com mecanismos de:
- Taxa de transação
- Queima automática
- Bloqueio inicial de transferências
- Reservas para pré-venda, liquidez, staking, etc.

## Auditoria
Auditoria comunitária em andamento.

## Contato
https://discord.gg/mRqMYsu2

## Código
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract BloounTech is ERC20, ReentrancyGuard, Ownable {
    uint256 public constant INITIAL_SUPPLY = 250000000 * (10 ** 18); // 250 million tokens

    uint256 public transactionTax = 1; // Transaction tax (%)
    uint256 public burnTax = 1; // Burn tax (%)

    uint256 public airdropReserve = (INITIAL_SUPPLY * 20) / 100; // 20% for airdrops
    uint256 public liquidityReserve = (INITIAL_SUPPLY * 25) / 100; // 25% for liquidity
    uint256 public partnershipReserve = (INITIAL_SUPPLY * 15) / 100; // 15% for partnerships
    uint256 public presaleReserve = (INITIAL_SUPPLY * 15) / 100; // 15% for presale
    uint256 public stakingRewardsReserve = (INITIAL_SUPPLY * 25) / 100; // 25% for staking and rewards

    // Wallet addresses for reserves
    address public airdropWallet;
    address public liquidityWallet;
    address public partnershipWallet;
    address public presaleWallet;
    address public stakingWallet;
    address public taxWallet;

    mapping(address => bool) public taxExempt; // Tax exemptions for addresses
    mapping(address => bool) private exemptedAddresses; 

    bool public transfersBlocked = true; // Initial transfer blocking
    uint256 public totalTokensSold; // Tokens sold during presale

    event PresaleFinalized(uint256 totalRaised);
    event TokensBurned(uint256 amount);

    constructor(
        address _airdropWallet, 
        address _liquidityWallet, 
        address _partnershipWallet, 
        address _presaleWallet, 
        address _stakingWallet, 
        address _taxWallet, 
        address initialOwner
    ) ERC20("BloounTech", "BLOOUN") Ownable(initialOwner) {
        require(_airdropWallet != address(0), "Invalid airdrop wallet");
        require(_liquidityWallet != address(0), "Invalid liquidity wallet");
        require(_partnershipWallet != address(0), "Invalid partnership wallet");
        require(_presaleWallet != address(0), "Invalid presale wallet");
        require(_stakingWallet != address(0), "Invalid staking wallet");
        require(_taxWallet != address(0), "Invalid tax wallet"); // 

        airdropWallet = _airdropWallet;
        liquidityWallet = _liquidityWallet;
        partnershipWallet = _partnershipWallet;
        presaleWallet = _presaleWallet;
        stakingWallet = _stakingWallet;
        taxWallet = _taxWallet;

        _mint(airdropWallet, airdropReserve);
        _mint(liquidityWallet, liquidityReserve);
        _mint(partnershipWallet, partnershipReserve);
        _mint(presaleWallet, presaleReserve);
        _mint(stakingWallet, stakingRewardsReserve);
    }

    // Mark an address as exempt from fees
    function setExemptedAddress(address _address, bool _status) external onlyOwner {
        exemptedAddresses[_address] = _status;
    }

    // Finalize presale and transfer unsold tokens to liquidity wallet
    function finalizePresale() external onlyOwner {
        uint256 unsoldTokens = presaleReserve - totalTokensSold;
        if (unsoldTokens > 0) {
            _transfer(presaleWallet, liquidityWallet, unsoldTokens);
        }
        presaleReserve = totalTokensSold;
        emit PresaleFinalized(totalTokensSold);
    }

    // Unlock transfers after presale
    function unlockTransfers() external onlyOwner {
        transfersBlocked = false;
    }

    // Set transaction tax
    function setTransactionTax(uint256 newTax) external onlyOwner {
        require(newTax <= 10, "Transaction tax too high");
        transactionTax = newTax;
    }

    // Set burn tax
    function setBurnTax(uint256 newTax) external onlyOwner {
        require(newTax <= 10, "Burn tax too high");
        burnTax = newTax;
    }

    // Set tax exemption for an address
    function setTaxExempt(address account, bool exempt) external onlyOwner {
        taxExempt[account] = exempt;
    }

    // Override _beforeTokenTransfer to apply tax and burn
    function _beforeTokenTransfer(
        address from,
        address to,
        uint256 amount
    ) internal {
        require(!transfersBlocked || from == owner(), "Transfers are currently blocked");

        uint256 fee = 0;
        if (!exemptedAddresses[from] && !exemptedAddresses[to]) {
            fee = (amount * transactionTax) / 100;
        }

        uint256 amountAfterFee = amount - fee;

        if (fee > 0) {
            _burn(from, fee);
            emit TokensBurned(fee);
        }

        super._transfer(from, to, amountAfterFee);
    }
}
