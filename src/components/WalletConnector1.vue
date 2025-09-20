<template>
  <div class="wallet-container">
    <!-- 连接钱包 -->
    <button @click="connectWallet" :disabled="isConnecting">
      {{ isConnecting ? '连接中...' : '连接 OKX 钱包' }}
    </button>
    <p v-if="account">当前账户: {{ account }}</p>
    <p v-if="error" class="error">{{ error }}</p>

    <!-- 转账表单 -->
    <div v-if="account" class="transfer-form">
      <h3>转账 ERC20 代币 (USDT)</h3>
      <input v-model="toAddress" placeholder="接收地址 (0x...)" />
      <input v-model.number="amount" type="number" placeholder="转账金额 (如 100)" />
      <button @click="transferToken" :disabled="isTransferring">
        {{ isTransferring ? '转账中...' : '执行转账' }}
      </button>
      <p v-if="txHash">交易哈希: <a :href="explorerUrl" target="_blank">{{ txHash }}</a></p>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'
import Web3 from 'web3'

// Define ERC20 ABI
const erc20Abi = [
  {
    constant: false,
    inputs: [
      { name: '_to', type: 'address' },
      { name: '_value', type: 'uint256' }
    ],
    name: 'transfer',
    outputs: [{ name: '', type: 'bool' }],
    type: 'function'
  }
]

// 状态管理
const account = ref('')
const isConnecting = ref(false)
const error = ref('')
const toAddress = ref('')
const amount = ref(0)
const isTransferring = ref(false)
const txHash = ref('')
let web3 = null

// Etherscan 交易链接
const explorerUrl = computed(() => `https://etherscan.io/tx/${txHash.value}`)

// 连接 OKX 钱包
const connectWallet = async () => {
  if (!window.okxwallet) {
    error.value = '请安装 OKX Wallet 浏览器扩展！'
    return
  }

  try {
    isConnecting.value = true
    web3 = new Web3(window.okxwallet)

    // 请求连接账户
    const accounts = await window.okxwallet.request({
      method: 'eth_requestAccounts'
    })
    account.value = accounts[0]
    error.value = ''
    console.log('连接成功，账户:', account.value)

    // 切换到 Ethereum 主网 (chainId: 0x1)
    try {
      await window.okxwallet.request({
        method: 'wallet_switchEthereumChain',
        params: [{ chainId: '0x1' }]
      })
    } catch (switchError) {
      if (switchError.code === 4902) {
        await window.okxwallet.request({
          method: 'wallet_addEthereumChain',
          params: [{
            chainId: '0x1',
            chainName: 'Ethereum Mainnet',
            rpcUrls: ['https://mainnet.infura.io/v3/YOUR_INFURA_KEY'], // 替换为你的 Infura RPC
            nativeCurrency: { name: 'Ether', symbol: 'ETH', decimals: 18 },
            blockExplorerUrls: ['https://etherscan.io']
          }]
        })
      } else {
        throw switchError
      }
    }
  } catch (err) {
    error.value = `连接失败: ${err.message}`
    console.error(err)
  } finally {
    isConnecting.value = false
  }
}

// 执行 ERC20 转账
const transferToken = async () => {
  if (!web3 || !account.value) {
    error.value = '请先连接钱包'
    return
  }

  if (!web3.utils.isAddress(toAddress.value) || amount.value <= 0) {
    error.value = '请输入有效地址和金额'
    return
  }

  try {
    isTransferring.value = true
    const contractAddress = '0xdAC17F958D2ee523a2206206994597C13D831ec7' // USDT 合约地址
    const contract = new web3.eth.Contract(erc20Abi, contractAddress)

    // 金额转换（USDT decimals=6）
    const decimals = 6
    const value = web3.utils.toBN(amount.value * 10 ** decimals)

    // 估算 gas
    const gasEstimate = await contract.methods.transfer(toAddress.value, value).estimateGas({
      from: account.value
    })

    // 发送交易
    const tx = await contract.methods.transfer(toAddress.value, value).send({
      from: account.value,
      gas: Math.round(gasEstimate * 1.2) // 增加 20% gas 防止波动
    })

    txHash.value = tx.transactionHash
    error.value = ''
    console.log('转账成功，Tx Hash:', txHash.value)
  } catch (err) {
    error.value = `转账失败: ${err.message}`
    console.error(err)
  } finally {
    isTransferring.value = false
  }
}
</script>

<style scoped>
.wallet-container {
  max-width: 500px;
  margin: 0 auto;
  padding: 20px;
  text-align: center;
}
.transfer-form {
  margin-top: 20px;
}
input {
  display: block;
  width: 100%;
  margin: 10px 0;
  padding: 8px;
}
button {
  padding: 10px 20px;
  background-color: #007bff;
  color: white;
  border: none;
  cursor: pointer;
}
button:disabled {
  background-color: #6c757d;
  cursor: not-allowed;
}
.error {
  color: red;
}
a {
  color: #007bff;
  text-decoration: none;
}
</style>
