# OneWinnerBNB

<p align="center">
  <img src="onewinnerbnb.png" alt="OneWinnerBNB" width="400"/>
</p>

<p align="center">
  <strong>确定性链上分配协议</strong><br/>
  一个持有者 · 一个区块 · 全部开发份额
</p>

---

## 概述

OneWinnerBNB 是部署在 BNB Chain 上的确定性分配系统。通过链上随机选择的未来区块，使用区块哈希和持有者余额数据,以完全可验证的方式选出单一获胜者,并自动转移 100% 的开发供应。

**核心原则:**
- 无人工干预
- 无后门权限
- 纯链上执行
- 完全透明可验证

---

## 技术架构

### 合约结构

```
OneWinnerBNB
├── 快照机制
│   ├── 区块高度选择
│   └── 持有者余额记录
├── 随机性来源
│   ├── 区块哈希熵值
│   └── 组合式随机数生成
└── 分配执行器
    ├── 获胜者计算
    └── 代币自动转移
```

### 选择算法

获胜者通过以下确定性过程选出:

1. **快照区块** (`B_snapshot`)  
   在合约部署时或通过链上随机性选择的未来区块高度

2. **熵值提取**  
   ```
   entropy = keccak256(blockhash(B_snapshot))
   ```

3. **加权选择**  
   每个持有者的中奖概率与其持有量成正比:
   ```
   P(holder_i) = balance_i / total_supply
   ```

4. **确定性映射**  
   ```
   winner_index = entropy mod Σ(balances)
   cumulative_sum = 0
   for each holder:
       cumulative_sum += holder.balance
       if cumulative_sum >= winner_index:
           return holder.address
   ```

---

## 数学验证

### 公平性证明

给定 `n` 个持有者,余额为 `b_1, b_2, ..., b_n`:

```
总供应量 T = Σ b_i

每个持有者的获胜概率:
P(winner = i) = b_i / T

期望值:
E[winner] = Σ (b_i / T) × i
```

该系统保证加权公平性,无需信任中心化随机源。

### 熵值来源

BNB Chain 区块哈希提供 256 位熵值:
- 不可预测(矿工无法控制未来区块哈希)
- 不可逆(哈希函数单向性)
- 可验证(所有节点共识)

---

## 合约规范

### 核心函数

```solidity
// 伪代码示例
contract OneWinnerBNB {
    
    uint256 public snapshotBlock;
    uint256 public devSupply;
    address public winner;
    
    mapping(address => uint256) public balancesAtSnapshot;
    address[] public holders;
    
    function captureSnapshot() internal {
        // 记录所有持有者余额
        // 锁定快照区块高度
    }
    
    function selectWinner() public returns (address) {
        require(block.number >= snapshotBlock);
        
        bytes32 entropy = blockhash(snapshotBlock);
        uint256 randomIndex = uint256(entropy) % totalSupply;
        
        return deterministicSelection(randomIndex);
    }
    
    function deterministicSelection(uint256 index) 
        internal view returns (address) 
    {
        uint256 cumulative = 0;
        for (uint i = 0; i < holders.length; i++) {
            cumulative += balancesAtSnapshot[holders[i]];
            if (cumulative >= index) {
                return holders[i];
            }
        }
    }
    
    function executeTransfer() public {
        require(winner != address(0));
        _transfer(address(this), winner, devSupply);
    }
}
```

### 安全特性

| 特性 | 实现 |
|------|------|
| 防重入攻击 | ReentrancyGuard |
| 访问控制 | 部署后无管理员权限 |
| 时间锁 | 快照区块前禁止执行 |
| 溢出保护 | Solidity ^0.8.x |
| 审计状态 | 待第三方审计 |

---

## 部署参数

```json
{
  "network": "BNB Chain (BSC)",
  "compiler": "Solidity ^0.8.19",
  "optimization": true,
  "runs": 200,
  "snapshotDelay": "随机未来区块 (7-30天)",
  "devSupply": "合约部署时定义",
  "minHoldersRequired": 100
}
```

---

## 验证方法

任何人都可以独立验证结果:

### 链上验证

```bash
# 1. 获取快照区块哈希
cast block <SNAPSHOT_BLOCK> --rpc-url <BNB_RPC>

# 2. 提取熵值
cast call <CONTRACT> "getEntropy()" --rpc-url <BNB_RPC>

# 3. 重构选择算法
# 使用相同输入运行本地脚本

# 4. 对比获胜地址
cast call <CONTRACT> "winner()" --rpc-url <BNB_RPC>
```

### 离线验证

```python
# 示例验证脚本
import hashlib

def verify_winner(block_hash, holders_balances):
    entropy = int(block_hash, 16)
    total_supply = sum(holders_balances.values())
    random_index = entropy % total_supply
    
    cumulative = 0
    for address, balance in holders_balances.items():
        cumulative += balance
        if cumulative >= random_index:
            return address
```

---

## 风险声明

> **实验性协议**  
> OneWinnerBNB 是一个实验性的链上分配机制。不保证代币价值,不构成投资建议。参与者应充分理解智能合约风险。

### 已知限制

- **区块哈希依赖性**: 虽然矿工无法预测未来区块哈希,但理论上存在极小概率的操纵可能性
- **快照时效性**: 持有者可能在快照区块前转移代币
- **Gas 成本**: 大量持有者可能导致选择函数 Gas 消耗较高

---

## 技术文档

### 状态机

```
[已部署] 
    ↓
[等待快照区块]
    ↓
[快照已捕获]
    ↓
[计算获胜者]
    ↓
[执行转移]
    ↓
[已完成]
```

### 事件日志

```solidity
event SnapshotCaptured(uint256 blockNumber, uint256 totalHolders);
event WinnerSelected(address indexed winner, uint256 amount);
event DevSupplyTransferred(address indexed winner, uint256 timestamp);
```

---

## 开发指南

### 本地测试

```bash
# 安装依赖
npm install

# 编译合约
npx hardhat compile

# 运行测试套件
npx hardhat test

# 部署到测试网
npx hardhat run scripts/deploy.js --network bscTestnet
```

### 环境变量

```env
BSC_RPC_URL=https://bsc-dataseed.binance.org
PRIVATE_KEY=your_private_key
BSCSCAN_API_KEY=your_api_key
```

---

## 资源链接

- **合约地址**: `待部署`
- **BscScan**: `待更新`
- **审计报告**: `进行中`
- **技术白皮书**: `docs/whitepaper.pdf`

---

## 许可证

MIT License

---

<p align="center">
  <sub>确定性 · 透明 · 不可篡改</sub><br/>
  <sub>OneWinnerBNB - 链上公平分配协议</sub>
</p>