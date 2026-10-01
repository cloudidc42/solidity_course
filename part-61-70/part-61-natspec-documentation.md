# Part 61: NatSpec & Protocol Documentation

## บทนำ

เมื่อพัฒนา Smart Contract สำหรับ Production การเขียน Documentation ที่ดีเป็นสิ่งสำคัญไม่แพ้ตัว Code เอง NatSpec (Natural Language Specification) คือมาตรฐาน Documentation Format ที่ Ethereum ใช้สำหรับ Smart Contracts ช่วยให้ Developer, Auditor, และผู้ใช้ทั่วไปเข้าใจ Contract ได้ง่ายขึ้น

## 1. NatSpec Format ทั้งหมด

### 1.1 Tags ที่ใช้ได้ใน NatSpec

| Tag | ใช้ที่ | ความหมาย |
|-----|--------|-----------|
| `@title` | Contract/Interface | ชื่อ Contract |
| `@author` | Contract/Interface | ชื่อผู้เขียน |
| `@notice` | ทุกที่ | อธิบายสำหรับผู้ใช้ทั่วไป |
| `@dev` | ทุกที่ | อธิบาย Technical Detail สำหรับ Developer |
| `@param` | Function/Event/Error | อธิบาย Parameter |
| `@return` | Function | อธิบาย Return Value |
| `@inheritdoc` | Function | สืบทอด Doc จาก Interface/Base Contract |
| `@custom:tag` | ทุกที่ | Custom Tag ที่กำหนดเอง |

### 1.2 รูปแบบการเขียน

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title ชื่อ Contract
/// @author ชื่อผู้เขียน
/// @notice อธิบายสำหรับ End User
/// @dev อธิบาย Technical Detail
contract MyContract {
    /// @notice อธิบาย State Variable
    /// @dev ใช้ packed storage เพื่อประหยัด gas
    uint256 public myVar;

    /// @notice ทำ X และ Y
    /// @dev เรียกใช้ internal function Z
    /// @param amount จำนวนที่ต้องการ
    /// @param recipient ที่อยู่ผู้รับ
    /// @return success true ถ้าสำเร็จ
    function myFunction(uint256 amount, address recipient) 
        external 
        returns (bool success) 
    {
        // implementation
    }
}
```

## 2. DocumentedVault: ERC-4626 Vault พร้อม NatSpec ครบถ้วน

ERC-4626 เป็นมาตรฐาน Tokenized Vault ที่ใช้กันอย่างแพร่หลาย เราจะเขียน Vault นี้พร้อม NatSpec ทุก Function

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {IERC20Metadata} from "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {Math} from "@openzeppelin/contracts/utils/math/Math.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";

/// @title DocumentedVault
/// @author Solidity Course Team
/// @notice ERC-4626 Tokenized Vault สำหรับการฝากและถอน ERC-20 Token
/// @dev Implements ERC-4626 standard with additional fee mechanism
/// @custom:security-contact security@example.com
/// @custom:version 1.0.0
/// @custom:audit-status unaudited
contract DocumentedVault is ERC20, Ownable {
    using SafeERC20 for IERC20;
    using Math for uint256;

    // ─────────────────────────────────────────────
    // Custom Errors
    // ─────────────────────────────────────────────

    /// @notice เกิดเมื่อพยายาม deposit หรือ mint 0
    error ZeroAssets();

    /// @notice เกิดเมื่อพยายาย withdraw หรือ redeem 0 shares
    error ZeroShares();

    /// @notice เกิดเมื่อ shares เกิน maximum ที่อนุญาต
    /// @param shares จำนวน shares ที่ร้องขอ
    /// @param maxShares จำนวน shares สูงสุดที่อนุญาต
    error ExceedsMaxDeposit(uint256 shares, uint256 maxShares);

    /// @notice เกิดเมื่อ assets เกิน maximum ที่อนุญาต
    /// @param assets จำนวน assets ที่ร้องขอ
    /// @param maxAssets จำนวน assets สูงสุดที่อนุญาต
    error ExceedsMaxWithdraw(uint256 assets, uint256 maxAssets);

    /// @notice เกิดเมื่อ fee rate ไม่ถูกต้อง (เกิน 10000 basis points)
    /// @param fee fee ที่ถูกส่งมา
    error InvalidFeeRate(uint256 fee);

    // ─────────────────────────────────────────────
    // Events
    // ─────────────────────────────────────────────

    /// @notice เกิดเมื่อมีการ deposit assets เข้า vault
    /// @param caller ผู้เรียก function
    /// @param owner เจ้าของ shares ที่ mint
    /// @param assets จำนวน assets ที่ deposit
    /// @param shares จำนวน shares ที่ mint
    event Deposit(
        address indexed caller,
        address indexed owner,
        uint256 assets,
        uint256 shares
    );

    /// @notice เกิดเมื่อมีการ withdraw assets จาก vault
    /// @param caller ผู้เรียก function
    /// @param receiver ผู้รับ assets
    /// @param owner เจ้าของ shares ที่ burn
    /// @param assets จำนวน assets ที่ withdraw
    /// @param shares จำนวน shares ที่ burn
    event Withdraw(
        address indexed caller,
        address indexed receiver,
        address indexed owner,
        uint256 assets,
        uint256 shares
    );

    /// @notice เกิดเมื่อ fee rate เปลี่ยนแปลง
    /// @param oldFee fee เดิม (basis points)
    /// @param newFee fee ใหม่ (basis points)
    event FeeRateUpdated(uint256 oldFee, uint256 newFee);

    /// @notice เกิดเมื่อ fee สะสมถูก harvest
    /// @param recipient ผู้รับ fee
    /// @param amount จำนวน shares ที่ส่งเป็น fee
    event FeeHarvested(address indexed recipient, uint256 amount);

    // ─────────────────────────────────────────────
    // State Variables
    // ─────────────────────────────────────────────

    /// @notice Underlying asset token ที่ vault จัดการ
    /// @dev ถูกกำหนดตอน deploy และไม่สามารถเปลี่ยนได้
    IERC20 public immutable asset;

    /// @notice จำนวน decimal ของ underlying asset
    /// @dev ใช้สำหรับคำนวณ share/asset ratio
    uint8 private immutable _assetDecimals;

    /// @notice Fee rate ใน basis points (1 bp = 0.01%)
    /// @dev Maximum 10000 bp = 100%
    uint256 public feeRate;

    /// @notice จำนวน fee shares ที่สะสม ยังไม่ได้ harvest
    /// @dev เพิ่มขึ้นทุกครั้งที่มี deposit
    uint256 public accumulatedFeeShares;

    /// @notice Constant สำหรับ virtual shares เพื่อป้องกัน inflation attack
    /// @dev ดู https://github.com/OpenZeppelin/openzeppelin-contracts/issues/3706
    uint256 private constant VIRTUAL_SHARES = 1e3;

    /// @notice Constant สำหรับ virtual assets
    uint256 private constant VIRTUAL_ASSETS = 1;

    /// @notice Maximum basis points (100%)
    uint256 private constant MAX_BPS = 10_000;

    // ─────────────────────────────────────────────
    // Constructor
    // ─────────────────────────────────────────────

    /// @notice สร้าง Vault ใหม่
    /// @dev Deploy vault พร้อมกำหนด underlying asset และ initial fee
    /// @param _asset Token ที่ vault จะรับฝาก
    /// @param _name ชื่อ share token (เช่น "Vault USDC")
    /// @param _symbol Symbol ของ share token (เช่น "vUSDC")
    /// @param _feeRate Fee ใน basis points (0-1000 = 0-10%)
    /// @param _owner เจ้าของ vault (ผู้มีสิทธิ์ harvest fee)
    constructor(
        address _asset,
        string memory _name,
        string memory _symbol,
        uint256 _feeRate,
        address _owner
    ) ERC20(_name, _symbol) Ownable(_owner) {
        if (_feeRate > MAX_BPS) revert InvalidFeeRate(_feeRate);
        asset = IERC20(_asset);
        _assetDecimals = IERC20Metadata(_asset).decimals();
        feeRate = _feeRate;
    }

    // ─────────────────────────────────────────────
    // ERC-4626 Core Functions
    // ─────────────────────────────────────────────

    /// @notice คืน decimal ของ share token
    /// @dev เท่ากับ decimal ของ underlying asset
    /// @return จำนวน decimal
    function decimals() public view override returns (uint8) {
        return _assetDecimals;
    }

    /// @notice คืนจำนวน assets ทั้งหมดที่ vault จัดการ
    /// @dev รวม fee ที่ยังไม่ได้ harvest ด้วย
    /// @return totalManagedAssets จำนวน assets ทั้งหมด
    function totalAssets() public view returns (uint256 totalManagedAssets) {
        return asset.balanceOf(address(this));
    }

    /// @notice แปลง assets เป็น shares
    /// @dev ใช้ virtual shares เพื่อป้องกัน inflation attack
    /// @param assets จำนวน assets ที่ต้องการแปลง
    /// @return shares จำนวน shares ที่ได้
    function convertToShares(uint256 assets) public view returns (uint256 shares) {
        return assets.mulDiv(
            totalSupply() + VIRTUAL_SHARES,
            totalAssets() + VIRTUAL_ASSETS,
            Math.Rounding.Floor
        );
    }

    /// @notice แปลง shares เป็น assets
    /// @dev ใช้ virtual shares เพื่อป้องกัน inflation attack
    /// @param shares จำนวน shares ที่ต้องการแปลง
    /// @return assets จำนวน assets ที่ได้
    function convertToAssets(uint256 shares) public view returns (uint256 assets) {
        return shares.mulDiv(
            totalAssets() + VIRTUAL_ASSETS,
            totalSupply() + VIRTUAL_SHARES,
            Math.Rounding.Floor
        );
    }

    /// @notice คืนจำนวน assets สูงสุดที่ address สามารถ deposit ได้
    /// @dev ไม่มีการจำกัด เว้นแต่ overflow
    /// @param /* receiver */ ผู้รับ shares (ไม่ใช้ใน implementation นี้)
    /// @return maxAssets จำนวน assets สูงสุด
    function maxDeposit(address /* receiver */) public pure returns (uint256 maxAssets) {
        return type(uint256).max;
    }

    /// @notice คืนจำนวน shares สูงสุดที่ address สามารถ mint ได้
    /// @param /* receiver */ ผู้รับ shares (ไม่ใช้ใน implementation นี้)
    /// @return maxShares จำนวน shares สูงสุด
    function maxMint(address /* receiver */) public pure returns (uint256 maxShares) {
        return type(uint256).max;
    }

    /// @notice คืนจำนวน assets สูงสุดที่ owner สามารถ withdraw ได้
    /// @param owner ที่อยู่ที่ต้องการตรวจสอบ
    /// @return maxAssets จำนวน assets สูงสุดที่ถอนได้
    function maxWithdraw(address owner) public view returns (uint256 maxAssets) {
        return convertToAssets(balanceOf(owner));
    }

    /// @notice คืนจำนวน shares สูงสุดที่ owner สามารถ redeem ได้
    /// @param owner ที่อยู่ที่ต้องการตรวจสอบ
    /// @return maxShares จำนวน shares สูงสุดที่ redeem ได้
    function maxRedeem(address owner) public view returns (uint256 maxShares) {
        return balanceOf(owner);
    }

    /// @notice คาดการณ์ shares ที่จะได้รับจาก deposit
    /// @dev หักค่า fee ก่อนคำนวณ
    /// @param assets จำนวน assets ที่จะ deposit
    /// @return shares จำนวน shares ที่จะได้รับ (หลังหัก fee)
    function previewDeposit(uint256 assets) public view returns (uint256 shares) {
        uint256 fee = _calculateFee(assets);
        return convertToShares(assets - fee);
    }

    /// @notice คาดการณ์ assets ที่ต้องใช้เพื่อ mint shares จำนวนหนึ่ง
    /// @dev บวก fee เข้าไปใน assets ที่ต้องใช้
    /// @param shares จำนวน shares ที่ต้องการ mint
    /// @return assets จำนวน assets ที่ต้องใช้ (รวม fee)
    function previewMint(uint256 shares) public view returns (uint256 assets) {
        uint256 baseAssets = convertToAssets(shares);
        return baseAssets + _calculateFee(baseAssets);
    }

    /// @notice คาดการณ์ shares ที่ต้อง burn เพื่อ withdraw assets จำนวนหนึ่ง
    /// @param assets จำนวน assets ที่ต้องการ withdraw
    /// @return shares จำนวน shares ที่ต้อง burn
    function previewWithdraw(uint256 assets) public view returns (uint256 shares) {
        return convertToShares(assets);
    }

    /// @notice คาดการณ์ assets ที่จะได้รับจาก redeem shares จำนวนหนึ่ง
    /// @param shares จำนวน shares ที่ต้องการ redeem
    /// @return assets จำนวน assets ที่จะได้รับ
    function previewRedeem(uint256 shares) public view returns (uint256 assets) {
        return convertToAssets(shares);
    }

    // ─────────────────────────────────────────────
    // Deposit/Withdraw Functions
    // ─────────────────────────────────────────────

    /// @notice Deposit assets เข้า vault เพื่อรับ shares
    /// @dev ดึง assets จาก msg.sender, mint shares ให้ receiver
    ///      ค่า fee ถูกหักก่อน mint shares
    /// @param assets จำนวน underlying assets ที่จะ deposit
    /// @param receiver ที่อยู่ที่จะรับ shares
    /// @return shares จำนวน shares ที่ mint ให้ receiver
    function deposit(uint256 assets, address receiver) external returns (uint256 shares) {
        if (assets == 0) revert ZeroAssets();

        // คำนวณ fee
        uint256 fee = _calculateFee(assets);
        uint256 assetsAfterFee = assets - fee;

        // คำนวณ shares ที่จะ mint
        shares = convertToShares(assetsAfterFee);
        if (shares == 0) revert ZeroShares();

        // ดึง assets จาก caller
        asset.safeTransferFrom(msg.sender, address(this), assets);

        // สะสม fee shares
        if (fee > 0) {
            uint256 feeShares = convertToShares(fee);
            accumulatedFeeShares += feeShares;
        }

        // Mint shares ให้ receiver
        _mint(receiver, shares);

        emit Deposit(msg.sender, receiver, assets, shares);
    }

    /// @notice Mint shares จำนวนที่กำหนด (ดึง assets เพิ่มตาม ratio)
    /// @dev คำนวณ assets ที่ต้องใช้รวม fee แล้วดึงจาก msg.sender
    /// @param shares จำนวน shares ที่ต้องการ mint
    /// @param receiver ที่อยู่ที่จะรับ shares
    /// @return assets จำนวน assets ที่ถูกดึงจาก msg.sender
    function mint(uint256 shares, address receiver) external returns (uint256 assets) {
        if (shares == 0) revert ZeroShares();

        assets = previewMint(shares);

        // ดึง assets จาก caller (รวม fee)
        asset.safeTransferFrom(msg.sender, address(this), assets);

        // สะสม fee shares
        uint256 fee = _calculateFee(convertToAssets(shares));
        if (fee > 0) {
            uint256 feeShares = convertToShares(fee);
            accumulatedFeeShares += feeShares;
        }

        _mint(receiver, shares);

        emit Deposit(msg.sender, receiver, assets, shares);
    }

    /// @notice Withdraw assets จำนวนที่กำหนด โดย burn shares ตาม ratio
    /// @dev Owner ต้องมี shares เพียงพอ
    ///      ถ้า caller ไม่ใช่ owner ต้องมี allowance
    /// @param assets จำนวน assets ที่ต้องการถอน
    /// @param receiver ที่อยู่ที่จะรับ assets
    /// @param owner ที่อยู่ที่เป็นเจ้าของ shares
    /// @return shares จำนวน shares ที่ถูก burn
    function withdraw(
        uint256 assets,
        address receiver,
        address owner
    ) external returns (uint256 shares) {
        if (assets == 0) revert ZeroAssets();

        shares = previewWithdraw(assets);
        if (shares == 0) revert ZeroShares();

        uint256 maxShares = maxRedeem(owner);
        if (shares > maxShares) revert ExceedsMaxWithdraw(assets, convertToAssets(maxShares));

        // ตรวจสอบ allowance ถ้าไม่ใช่เจ้าของ
        if (msg.sender != owner) {
            _spendAllowance(owner, msg.sender, shares);
        }

        _burn(owner, shares);
        asset.safeTransfer(receiver, assets);

        emit Withdraw(msg.sender, receiver, owner, assets, shares);
    }

    /// @notice Redeem shares จำนวนที่กำหนด เพื่อรับ assets ตาม ratio
    /// @param shares จำนวน shares ที่ต้องการ redeem
    /// @param receiver ที่อยู่ที่จะรับ assets
    /// @param owner ที่อยู่ที่เป็นเจ้าของ shares
    /// @return assets จำนวน assets ที่ส่งให้ receiver
    function redeem(
        uint256 shares,
        address receiver,
        address owner
    ) external returns (uint256 assets) {
        if (shares == 0) revert ZeroShares();

        assets = previewRedeem(shares);
        if (assets == 0) revert ZeroAssets();

        uint256 maxShares = maxRedeem(owner);
        if (shares > maxShares) revert ExceedsMaxWithdraw(assets, maxWithdraw(owner));

        if (msg.sender != owner) {
            _spendAllowance(owner, msg.sender, shares);
        }

        _burn(owner, shares);
        asset.safeTransfer(receiver, assets);

        emit Withdraw(msg.sender, receiver, owner, assets, shares);
    }

    // ─────────────────────────────────────────────
    // Fee Management
    // ─────────────────────────────────────────────

    /// @notice อัปเดต fee rate
    /// @dev ต้องเป็น owner เท่านั้น, fee ต้องไม่เกิน 10%
    /// @param newFeeRate Fee rate ใหม่ใน basis points (0-1000)
    function setFeeRate(uint256 newFeeRate) external onlyOwner {
        if (newFeeRate > 1000) revert InvalidFeeRate(newFeeRate); // max 10%
        uint256 oldFee = feeRate;
        feeRate = newFeeRate;
        emit FeeRateUpdated(oldFee, newFeeRate);
    }

    /// @notice Harvest fee shares ที่สะสมไว้
    /// @dev Mint shares ให้ recipient แทนการโอน assets
    ///      การ mint shares จะ dilute holder ทุกคน
    /// @param recipient ที่อยู่ที่จะรับ fee shares
    function harvestFees(address recipient) external onlyOwner {
        uint256 feeShares = accumulatedFeeShares;
        if (feeShares == 0) return;

        accumulatedFeeShares = 0;
        _mint(recipient, feeShares);

        emit FeeHarvested(recipient, feeShares);
    }

    // ─────────────────────────────────────────────
    // Internal Functions
    // ─────────────────────────────────────────────

    /// @notice คำนวณ fee จาก assets amount
    /// @dev ใช้ basis points (1 bp = 0.01%)
    /// @param assets จำนวน assets ที่จะคำนวณ fee
    /// @return fee จำนวน fee ที่คำนวณได้
    function _calculateFee(uint256 assets) internal view returns (uint256 fee) {
        return assets.mulDiv(feeRate, MAX_BPS, Math.Rounding.Floor);
    }
}
```

### 2.1 Interface พร้อม NatSpec

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title IDocumentedVault
/// @notice Interface สำหรับ ERC-4626 Vault
/// @dev ทุก function มี NatSpec ครบถ้วนตาม ERC-4626 standard
interface IDocumentedVault {
    // ─────────────────────────────────────────────
    // View Functions
    // ─────────────────────────────────────────────

    /// @notice คืน underlying asset ของ vault
    /// @return assetTokenAddress ที่อยู่ของ asset token
    function asset() external view returns (address assetTokenAddress);

    /// @notice คืน assets ทั้งหมดที่ vault จัดการ
    /// @return totalManagedAssets จำนวน assets รวม
    function totalAssets() external view returns (uint256 totalManagedAssets);

    /// @notice แปลง assets เป็น shares (ปัดลง)
    /// @param assets จำนวน assets
    /// @return shares จำนวน shares ที่ได้
    function convertToShares(uint256 assets) external view returns (uint256 shares);

    /// @notice แปลง shares เป็น assets (ปัดลง)
    /// @param shares จำนวน shares
    /// @return assets จำนวน assets ที่ได้
    function convertToAssets(uint256 shares) external view returns (uint256 assets);
}
```

## 3. การ Generate Docs ด้วย Forge และ Hardhat

### 3.1 Forge Doc

`forge doc` สร้าง Documentation จาก NatSpec อัตโนมัติ

```bash
# ติดตั้ง Foundry
curl -L https://foundry.paradigm.xyz | bash
foundryup

# โครงสร้างโปรเจกต์
my-protocol/
├── src/
│   ├── DocumentedVault.sol
│   └── interfaces/
│       └── IDocumentedVault.sol
├── test/
├── foundry.toml
└── docs/           # จะถูกสร้างโดย forge doc
```

```toml
# foundry.toml
[profile.default]
src = "src"
out = "out"
libs = ["lib"]

[doc]
out = "docs"
title = "My Protocol Documentation"
repository = "https://github.com/my-org/my-protocol"
```

```bash
# Generate documentation
forge doc

# Build และ serve locally
forge doc --serve

# ดูที่ http://localhost:3000
```

### 3.2 Hardhat Docgen

```bash
# ติดตั้ง plugin
npm install --save-dev @solidity-docgen/hardhat
```

```javascript
// hardhat.config.js
require("@nomicfoundation/hardhat-toolbox");
require("@solidity-docgen/hardhat");

module.exports = {
  solidity: "0.8.24",
  docgen: {
    output: "docs",
    pages: "files",         // สร้างหน้าแยกต่างหาก
    exclude: ["mocks/"],    // ไม่รวม mock contracts
    theme: "markdown",      // ใช้ markdown theme
    templates: "docgen-templates",  // custom templates
  }
};
```

```bash
# Generate docs
npx hardhat docgen
```

### 3.3 Custom Template (Handlebars)

```handlebars
{{! docgen-templates/contract.hbs }}
# {{name}}

{{#if natspec.title}}
## {{natspec.title}}
{{/if}}

{{#if natspec.notice}}
> {{natspec.notice}}
{{/if}}

{{#if natspec.dev}}
**Developer Note:** {{natspec.dev}}
{{/if}}

## Functions

{{#each functions}}
### `{{name}}`

{{#if natspec.notice}}
{{natspec.notice}}
{{/if}}

**Parameters:**
{{#each natspec.params}}
- `{{name}}`: {{description}}
{{/each}}

{{#if natspec.returns}}
**Returns:**
{{#each natspec.returns}}
- `{{name}}`: {{description}}
{{/each}}
{{/if}}

---
{{/each}}
```

## 4. Protocol-Level Architecture Documentation

### 4.1 โครงสร้าง README.md ที่ดี

```markdown
# Protocol Name

> One-line description of what the protocol does

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Tests](https://github.com/org/repo/actions/workflows/test.yml/badge.svg)](...)
[![Coverage](https://codecov.io/gh/org/repo/badge.svg)](...)

## Overview

Brief explanation (2-3 paragraphs):
- What problem does it solve?
- How does it work at a high level?
- Who is it for?

## Architecture

### Core Contracts

| Contract | Description | Deployments |
|----------|-------------|-------------|
| `DocumentedVault` | ERC-4626 vault with fee | [Mainnet](#), [Goerli](#) |
| `FeeCollector` | Collects and distributes fees | [Mainnet](#), [Goerli](#) |
| `VaultFactory` | Deploys new vaults | [Mainnet](#), [Goerli](#) |

### System Diagram

```
User
  │
  ├──deposit()──► DocumentedVault ◄──► ERC-20 Asset
  │                    │
  │                    ├──fee──► FeeCollector ──► Treasury
  │                    │                    └──► Stakers
  └──redeem()◄──┘
```

## Deployments

### Mainnet (Ethereum)
- `DocumentedVault (USDC)`: `0x1234...`
- `FeeCollector`: `0x5678...`

### Testnet (Sepolia)
- `DocumentedVault (USDC)`: `0xabcd...`
- `FeeCollector`: `0xef01...`

## Installation

```bash
# Using Foundry
forge install org/protocol

# Using npm
npm install @org/protocol
```

## Quick Start

```solidity
import "@org/protocol/src/DocumentedVault.sol";

contract MyIntegration {
    DocumentedVault vault = DocumentedVault(VAULT_ADDRESS);
    
    function depositToVault(uint256 amount) external {
        USDC.approve(address(vault), amount);
        vault.deposit(amount, msg.sender);
    }
}
```

## Security

### Audits
- [Audit by Firm X](link) - Date
- [Audit by Firm Y](link) - Date

### Bug Bounty
Report vulnerabilities to: security@protocol.xyz
Bounty: Up to $100,000 USDC

### Known Risks
- Smart contract risk
- Oracle manipulation risk
- Governance attack risk

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md)

## License

MIT License - see [LICENSE](LICENSE)
```

### 4.2 ARCHITECTURE.md แยกต่างหาก

```markdown
# Protocol Architecture

## Design Principles

1. **Simplicity**: ทุก Contract ทำหน้าที่เดียว
2. **Upgradeability**: ใช้ proxy pattern สำหรับ critical components
3. **Security**: Defense in depth, multiple audits
4. **Gas efficiency**: Packed storage, batch operations

## Data Flow

### Deposit Flow
1. User calls `vault.deposit(assets, receiver)`
2. Vault calculates fee: `fee = assets * feeRate / 10000`
3. Vault pulls assets from user via `safeTransferFrom`
4. Vault mints shares: `shares = (assets - fee) * totalSupply / totalAssets`
5. Fee shares accumulated in `accumulatedFeeShares`

### Fee Collection Flow
1. Owner calls `vault.harvestFees(treasury)`
2. Fee shares minted to treasury
3. Treasury can redeem shares for assets

## Storage Layout

```
slot 0: [ERC20 name/symbol via OZ]
slot 1: [asset address - immutable]
slot 2: [feeRate - uint256]
slot 3: [accumulatedFeeShares - uint256]
```

## Upgrade Path

The protocol uses transparent proxy pattern:
- ProxyAdmin: controlled by multisig
- Implementation: can be upgraded with 48h timelock
- Storage: never modified, only appended
```

## 5. Changelog และ Versioning

### 5.1 CHANGELOG.md Format (Keep a Changelog)

```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- New `batchDeposit` function for gas savings

## [1.2.0] - 2024-03-15

### Added
- `@custom:audit-status` NatSpec tag to all contracts
- Emergency pause functionality
- Event for fee rate changes (`FeeRateUpdated`)

### Changed
- Maximum fee rate reduced from 20% to 10%
- `previewMint` now rounds up instead of down

### Fixed
- Integer overflow in `convertToShares` with very large amounts
- Missing event emission in `harvestFees`

### Security
- Added reentrancy guard to `withdraw` and `redeem`

## [1.1.0] - 2024-02-01

### Added
- Multi-asset support via `VaultFactory`
- Fee accumulation system

### Changed
- Upgraded to OpenZeppelin 5.0

### Deprecated
- `singleDeposit` (use `deposit` instead, will be removed in v2.0)

## [1.0.0] - 2024-01-01

### Added
- Initial release
- ERC-4626 compliant vault
- NatSpec documentation for all functions

[Unreleased]: https://github.com/org/repo/compare/v1.2.0...HEAD
[1.2.0]: https://github.com/org/repo/compare/v1.1.0...v1.2.0
[1.1.0]: https://github.com/org/repo/compare/v1.0.0...v1.1.0
[1.0.0]: https://github.com/org/repo/releases/tag/v1.0.0
```

### 5.2 Semantic Versioning สำหรับ Smart Contracts

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title Versioned
/// @notice Mixin สำหรับ Contract versioning
/// @dev ใช้ semantic versioning (major.minor.patch)
abstract contract Versioned {
    /// @notice Version ของ contract
    /// @dev เปลี่ยน major เมื่อ breaking change, minor เมื่อเพิ่ม feature, patch เมื่อ bugfix
    string public constant VERSION = "1.2.0";

    /// @notice Major version
    /// @dev เพิ่มเมื่อมี breaking changes ที่ไม่ compatible กับ version เดิม
    uint256 public constant MAJOR_VERSION = 1;

    /// @notice Minor version
    /// @dev เพิ่มเมื่อเพิ่ม functionality ใหม่ที่ backward compatible
    uint256 public constant MINOR_VERSION = 2;

    /// @notice Patch version
    /// @dev เพิ่มเมื่อ bug fixes ที่ backward compatible
    uint256 public constant PATCH_VERSION = 0;
}
```

### 5.3 Contract Deployment Registry

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title DeploymentRegistry
/// @notice บันทึกประวัติการ deploy contracts ทั้งหมด
/// @dev ใช้สำหรับ tracking upgrades และ versions
/// @custom:deployment-date 2024-01-01
contract DeploymentRegistry {
    /// @notice ข้อมูล deployment
    /// @param contractAddress ที่อยู่ของ contract
    /// @param version version string
    /// @param deployedAt timestamp ของการ deploy
    /// @param deployedBy ผู้ deploy
    struct DeploymentInfo {
        address contractAddress;
        string version;
        uint256 deployedAt;
        address deployedBy;
        string changelog;
    }

    /// @notice ประวัติ deployment ทั้งหมด
    mapping(string => DeploymentInfo[]) public deploymentHistory;

    /// @notice บันทึก deployment ใหม่
    /// @param contractName ชื่อ contract
    /// @param contractAddress ที่อยู่ที่ deploy
    /// @param version version ที่ deploy
    /// @param changelog สรุปการเปลี่ยนแปลง
    function recordDeployment(
        string calldata contractName,
        address contractAddress,
        string calldata version,
        string calldata changelog
    ) external {
        deploymentHistory[contractName].push(DeploymentInfo({
            contractAddress: contractAddress,
            version: version,
            deployedAt: block.timestamp,
            deployedBy: msg.sender,
            changelog: changelog
        }));
    }

    /// @notice ดู deployment ล่าสุดของ contract
    /// @param contractName ชื่อ contract
    /// @return latest ข้อมูล deployment ล่าสุด
    function getLatestDeployment(string calldata contractName) 
        external 
        view 
        returns (DeploymentInfo memory latest) 
    {
        DeploymentInfo[] storage history = deploymentHistory[contractName];
        require(history.length > 0, "No deployments found");
        return history[history.length - 1];
    }
}
```

## 6. Workshop: เขียน NatSpec ให้ครบถ้วน

### Workshop 1: Complete NatSpec สำหรับ Staking Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/// @title StakingRewards
/// @author Protocol Team <dev@protocol.xyz>
/// @notice Stake TOKEN เพื่อรับ REWARD tokens
/// @dev ใช้ Synthetix StakingRewards pattern
///      Reward rate คำนวณแบบ per-second
///      Reference: https://github.com/Synthetixio/synthetix/blob/develop/contracts/StakingRewards.sol
/// @custom:security-contact security@protocol.xyz
/// @custom:audit-status audited-by-certik-2024-01
contract StakingRewards is ReentrancyGuard {
    using SafeERC20 for IERC20;

    // ─────────────────────────────────────────────
    // Type Declarations
    // ─────────────────────────────────────────────

    /// @notice เก็บข้อมูล staking ของ user แต่ละคน
    /// @dev packed เพื่อประหยัด gas (2 slots แทน 3)
    struct UserInfo {
        /// @dev จำนวน tokens ที่ stake อยู่
        uint128 staked;
        /// @dev rewards ที่คำนวณแล้วแต่ยังไม่ได้ claim
        uint128 pendingRewards;
        /// @dev rewardPerToken snapshot ตอน stake ครั้งล่าสุด
        uint256 rewardDebt;
    }

    // ─────────────────────────────────────────────
    // Constants
    // ─────────────────────────────────────────────

    /// @notice Precision multiplier สำหรับหลีกเลี่ยง integer division errors
    /// @dev 1e18 precision เพียงพอสำหรับ token amounts
    uint256 private constant PRECISION = 1e18;

    // ─────────────────────────────────────────────
    // Immutables
    // ─────────────────────────────────────────────

    /// @notice Token ที่ใช้ stake
    /// @dev ต้องเป็น ERC-20 standard token
    IERC20 public immutable stakingToken;

    /// @notice Token ที่ใช้จ่าย rewards
    /// @dev อาจเป็น token เดียวกับ stakingToken หรือไม่ก็ได้
    IERC20 public immutable rewardToken;

    // ─────────────────────────────────────────────
    // State Variables
    // ─────────────────────────────────────────────

    /// @notice เวลาที่ reward period สิ้นสุด
    /// @dev ถ้า block.timestamp > periodFinish ไม่มี reward แล้ว
    uint256 public periodFinish;

    /// @notice อัตรา reward ต่อวินาที
    /// @dev = totalReward / duration
    uint256 public rewardRate;

    /// @notice ระยะเวลาของ reward period (วินาที)
    uint256 public rewardsDuration = 7 days;

    /// @notice timestamp ล่าสุดที่ rewardPerTokenStored อัปเดต
    uint256 public lastUpdateTime;

    /// @notice Accumulated reward per token stake
    /// @dev เพิ่มขึ้นทุกวินาทีที่มี stake อยู่
    uint256 public rewardPerTokenStored;

    /// @notice Total tokens ที่ stake อยู่ใน contract
    uint256 public totalStaked;

    /// @notice ข้อมูล staking ของแต่ละ user
    mapping(address => UserInfo) public userInfo;

    // ─────────────────────────────────────────────
    // Events
    // ─────────────────────────────────────────────

    /// @notice เกิดเมื่อ user stake tokens
    /// @param user ที่อยู่ของ user
    /// @param amount จำนวนที่ stake
    event Staked(address indexed user, uint256 amount);

    /// @notice เกิดเมื่อ user unstake tokens
    /// @param user ที่อยู่ของ user
    /// @param amount จำนวนที่ unstake
    event Withdrawn(address indexed user, uint256 amount);

    /// @notice เกิดเมื่อ user claim rewards
    /// @param user ที่อยู่ของ user
    /// @param reward จำนวน reward ที่ได้รับ
    event RewardPaid(address indexed user, uint256 reward);

    /// @notice เกิดเมื่อ reward period ใหม่เริ่มต้น
    /// @param reward จำนวน rewards ทั้งหมดในรอบนี้
    event RewardAdded(uint256 reward);

    // ─────────────────────────────────────────────
    // Constructor
    // ─────────────────────────────────────────────

    /// @notice Deploy StakingRewards contract
    /// @dev ตั้งค่า immutable addresses ที่ไม่สามารถเปลี่ยนได้
    /// @param _stakingToken ที่อยู่ของ token ที่ใช้ stake
    /// @param _rewardToken ที่อยู่ของ token ที่ใช้จ่าย reward
    constructor(address _stakingToken, address _rewardToken) {
        stakingToken = IERC20(_stakingToken);
        rewardToken = IERC20(_rewardToken);
    }

    // ─────────────────────────────────────────────
    // View Functions
    // ─────────────────────────────────────────────

    /// @notice คืน timestamp ล่าสุดที่ยัง applicable (ไม่เกิน periodFinish)
    /// @dev ใช้ใน rewardPerToken calculation
    /// @return lastApplicableTime timestamp ที่ใช้คำนวณ
    function lastTimeRewardApplicable() public view returns (uint256 lastApplicableTime) {
        return block.timestamp < periodFinish ? block.timestamp : periodFinish;
    }

    /// @notice คำนวณ accumulated reward per token stake จนถึงปัจจุบัน
    /// @dev สูตร: stored + (elapsed * rate * precision) / totalStaked
    /// @return rewardPerToken จำนวน reward ต่อ 1 token stake
    function rewardPerToken() public view returns (uint256) {
        if (totalStaked == 0) return rewardPerTokenStored;
        return rewardPerTokenStored + (
            (lastTimeRewardApplicable() - lastUpdateTime) * rewardRate * PRECISION / totalStaked
        );
    }

    /// @notice คำนวณ rewards ที่ user จะได้รับถ้า claim ตอนนี้
    /// @param account ที่อยู่ของ user
    /// @return earnedAmount จำนวน rewards ที่ earn ได้
    function earned(address account) public view returns (uint256 earnedAmount) {
        UserInfo storage user = userInfo[account];
        return (uint256(user.staked) * (rewardPerToken() - user.rewardDebt) / PRECISION) 
               + uint256(user.pendingRewards);
    }

    // ─────────────────────────────────────────────
    // User Functions
    // ─────────────────────────────────────────────

    /// @notice Stake tokens เพื่อรับ rewards
    /// @dev อัปเดต reward state ก่อน stake
    ///      ต้อง approve token ก่อนเรียก function นี้
    /// @param amount จำนวน tokens ที่ต้องการ stake (ต้องมากกว่า 0)
    function stake(uint256 amount) external nonReentrant {
        require(amount > 0, "Cannot stake 0");
        _updateReward(msg.sender);

        totalStaked += amount;
        userInfo[msg.sender].staked += uint128(amount);
        stakingToken.safeTransferFrom(msg.sender, address(this), amount);

        emit Staked(msg.sender, amount);
    }

    /// @notice Unstake tokens
    /// @dev Claim rewards อัตโนมัติเมื่อ unstake
    /// @param amount จำนวน tokens ที่ต้องการ unstake
    function withdraw(uint256 amount) external nonReentrant {
        require(amount > 0, "Cannot withdraw 0");
        require(userInfo[msg.sender].staked >= amount, "Insufficient staked amount");
        _updateReward(msg.sender);

        totalStaked -= amount;
        userInfo[msg.sender].staked -= uint128(amount);
        stakingToken.safeTransfer(msg.sender, amount);

        emit Withdrawn(msg.sender, amount);
    }

    /// @notice Claim rewards ที่ earn ได้
    /// @dev ใช้ CEI pattern (Checks-Effects-Interactions)
    function getReward() external nonReentrant {
        _updateReward(msg.sender);
        uint256 reward = uint256(userInfo[msg.sender].pendingRewards);
        if (reward > 0) {
            userInfo[msg.sender].pendingRewards = 0;
            rewardToken.safeTransfer(msg.sender, reward);
            emit RewardPaid(msg.sender, reward);
        }
    }

    // ─────────────────────────────────────────────
    // Internal Functions
    // ─────────────────────────────────────────────

    /// @notice อัปเดต reward state สำหรับ account
    /// @dev เรียกก่อนทุก state-changing operation
    /// @param account ที่อยู่ที่ต้องการอัปเดต (address(0) = global only)
    function _updateReward(address account) internal {
        rewardPerTokenStored = rewardPerToken();
        lastUpdateTime = lastTimeRewardApplicable();
        if (account != address(0)) {
            UserInfo storage user = userInfo[account];
            user.pendingRewards = uint128(earned(account));
            user.rewardDebt = rewardPerTokenStored;
        }
    }
}
```

### Workshop 2: ตรวจสอบ NatSpec Completeness

```bash
#!/bin/bash
# check-natspec.sh - ตรวจสอบว่า Contract มี NatSpec ครบ

CONTRACT_FILE="$1"

echo "Checking NatSpec completeness for: $CONTRACT_FILE"
echo "================================================="

# Check @title
if grep -q "@title" "$CONTRACT_FILE"; then
    echo "✓ @title found"
else
    echo "✗ @title MISSING"
fi

# Check @notice on public functions
FUNCTIONS=$(grep -n "function.*public\|function.*external" "$CONTRACT_FILE" | grep -v "//")
echo ""
echo "Public/External Functions:"
while IFS= read -r line; do
    LINE_NUM=$(echo "$line" | cut -d: -f1)
    FUNC_NAME=$(echo "$line" | grep -o "function [a-zA-Z]*" | head -1)
    
    # Check if line before has @notice
    PREV_LINES=$(sed -n "$((LINE_NUM-5)),$((LINE_NUM-1))p" "$CONTRACT_FILE")
    if echo "$PREV_LINES" | grep -q "@notice"; then
        echo "  ✓ $FUNC_NAME has @notice"
    else
        echo "  ✗ $FUNC_NAME MISSING @notice"
    fi
done <<< "$FUNCTIONS"
```

## 7. Best Practices สรุป

### 7.1 Do's และ Don'ts

```solidity
// ❌ BAD: ไม่มี NatSpec
function transfer(address to, uint256 amount) external returns (bool) {
    balances[msg.sender] -= amount;
    balances[to] += amount;
    return true;
}

// ✅ GOOD: มี NatSpec ครบ
/// @notice โอน tokens ไปยัง address อื่น
/// @dev ใช้ checked arithmetic (reverts on overflow)
/// @param to ที่อยู่ผู้รับ (ต้องไม่ใช่ address(0))
/// @param amount จำนวนที่โอน (ต้องไม่เกิน balance)
/// @return success true ถ้าโอนสำเร็จ
function transfer(address to, uint256 amount) external returns (bool success) {
    require(to != address(0), "Transfer to zero address");
    balances[msg.sender] -= amount; // reverts if insufficient
    balances[to] += amount;
    success = true;
}
```

### 7.2 @inheritdoc Usage

```solidity
// Interface
interface IToken {
    /// @notice โอน tokens ไปยัง address อื่น
    /// @param to ผู้รับ
    /// @param amount จำนวน
    /// @return success ผลลัพธ์
    function transfer(address to, uint256 amount) external returns (bool success);
}

// Implementation - ใช้ @inheritdoc แทนการเขียนซ้ำ
contract Token is IToken {
    /// @inheritdoc IToken
    /// @dev เพิ่มเติม: emit Transfer event
    function transfer(address to, uint256 amount) external override returns (bool success) {
        // implementation
    }
}
```

### 7.3 Custom Tags สำหรับ Security

```solidity
/// @custom:security-invariant totalSupply == sum(balances)
/// @custom:security-contact security@protocol.xyz
/// @custom:audit-status audited-2024-01-certik
/// @custom:oz-upgrades-unsafe-allow constructor
/// @custom:experimental This feature is experimental
contract SecureToken {
    // ...
}
```

## สรุป Part 61

- **NatSpec** คือมาตรฐาน Documentation สำหรับ Solidity ที่ใช้ tags เช่น `@title`, `@notice`, `@dev`, `@param`, `@return`
- **DocumentedVault** แสดงให้เห็นการใช้ NatSpec ครบถ้วนใน ERC-4626 Vault จริง
- **forge doc** และ **hardhat-docgen** สามารถ generate documentation จาก NatSpec อัตโนมัติ
- **README.md** ที่ดีควรมี: overview, architecture diagram, deployments, security info
- **CHANGELOG.md** ควรใช้ Keep a Changelog format + Semantic Versioning
- **@inheritdoc** ใช้เพื่อสืบทอด documentation จาก Interface ป้องกันการเขียนซ้ำ
- **Custom tags** เช่น `@custom:security-contact` เพิ่ม metadata ที่มีประโยชน์

## Next: Part 62 - Code Review Best Practices
