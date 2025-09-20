<template>
  <div class="wallet-container">
    <div class="card">
      <h1>OKX 智能钱包 (Sepolia 测试网)</h1>

      <!-- 连接钱包 -->
      <div v-if="!isConnected" class="section">
        <button
            @click="connectWallet"
            :disabled="isConnecting"
            class="btn btn-primary"
        >
          {{ isConnecting ? '连接中...' : '连接 OKX 钱包' }}
        </button>
      </div>

      <!-- 已连接状态 -->
      <div v-else>
        <p class="address">已连接：{{ userAddress.slice(0, 6) }}...{{ userAddress.slice(-4) }}</p>

        <!-- 代币选择 -->
        <div class="section">
          <h2>选择代币</h2>
          <select v-model="selectedToken" class="input" @change="onTokenChange">
            <option v-for="token in tokens" :key="token.address" :value="token">
              {{ token.symbol }}
            </option>
          </select>
          <p v-if="selectedToken">当前代币: {{ selectedToken.symbol }} ({{ selectedToken.address }})</p>
        </div>

        <!-- 余额显示 -->
        <div class="section">
          <p>钱包余额：{{ balance }} {{ selectedToken.symbol }}</p>
          <p>智能钱包余额：{{ walletBalance }} {{ selectedToken.symbol }}</p>
          <p v-if="lastWithdrawTime">上次提取时间：{{ formatTime(lastWithdrawTime) }}</p>
          <div class="info-box" v-if="withdrawLimitInfo">
            {{ withdrawLimitInfo }}
          </div>
        </div>

        <!-- 存款 -->
        <div class="section">
          <h2>存款 {{ selectedToken.symbol }}</h2>
          <input
              v-model="depositAmount"
              type="number"
              placeholder="请输入存款金额"
              class="input"
          />
          <button
              @click="deposit"
              :disabled="isProcessing"
              class="btn btn-success"
          >
            {{ isProcessing ? '处理中...' : '存款' }}
          </button>
        </div>

        <!-- 提取 -->
        <div class="section">
          <h2>提取 {{ selectedToken.symbol }}</h2>
          <input
              v-model="withdrawAmount"
              type="number"
              placeholder="请输入提取金额"
              class="input"
          />
          <button
              @click="withdraw"
              :disabled="isProcessing"
              class="btn btn-danger"
          >
            {{ isProcessing ? '处理中...' : '提取' }}
          </button>
        </div>

        <!-- 直接转账 -->
        <div class="section">
          <h2>直接转账 {{ selectedToken.symbol }}</h2>
          <input
              v-model="toAddress"
              placeholder="接收地址 (0x...)"
              class="input"
          />
          <input
              v-model.number="transferAmount"
              type="number"
              placeholder="转账金额"
              class="input"
          />
          <button
              @click="transferToken"
              :disabled="isProcessing"
              class="btn btn-primary"
          >
            {{ isProcessing ? '转账中...' : '执行转账' }}
          </button>
          <p v-if="txHash" class="tx-link">
            交易哈希: <a :href="explorerUrl" target="_blank">{{ txHash.slice(0, 10) }}...{{ txHash.slice(-8) }}</a>
          </p>
        </div>

        <!-- 消息提示 -->
        <p v-if="successMessage" class="success">{{ successMessage }}</p>
        <p v-if="errorMessage" class="error">{{ errorMessage }}</p>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, markRaw, computed, onMounted } from 'vue'
import { ethers } from 'ethers'

// 验证 ethers.providers 是否存在
if (!ethers.providers) {
  console.error('ethers.providers 未定义，请检查 ethers.js 版本和导入')
  throw new Error('ethers.js 初始化失败')
}

// 代币列表
const tokens = [
  {
    symbol: 'USDT',
    address: '0x1ba13513dbfebaf1b588960119b5f04696ebd7d5',
    decimals: 18
  },
  {
    symbol: 'DAI',
    address: '0x73967c6a0904aa032c103b4104747e88c566b1a2',
    decimals: 18
  },
  {
    symbol: 'LINK',
    address: '0x779877A7B0D9E8603169DdbD7836e478b4624789',
    decimals: 18
  }
]

// 合约 ABI - 简化版本，只包含必要函数
const tokenABI = [
  "function balanceOf(address owner) view returns (uint256)",
  "function transfer(address to, uint256 amount) returns (bool)",
  "function approve(address spender, uint256 amount) returns (bool)",
  "function allowance(address owner, address spender) view returns (uint256)",
  "function decimals() view returns (uint8)",
  "event Transfer(address indexed from, address indexed to, uint256 value)"
]

const walletABI = [
  {
    "inputs": [
      {
        "internalType": "address",
        "name": "tokenAddress",
        "type": "address"
      },
      {
        "internalType": "uint256",
        "name": "amount",
        "type": "uint256"
      }
    ],
    "name": "deposit",
    "outputs": [],
    "stateMutability": "nonpayable",
    "type": "function"
  },
  {
    "inputs": [
      {
        "internalType": "address",
        "name": "tokenAddress",
        "type": "address"
      },
      {
        "internalType": "uint256",
        "name": "amount",
        "type": "uint256"
      }
    ],
    "name": "withdraw",
    "outputs": [],
    "stateMutability": "nonpayable",
    "type": "function"
  },
  {
    "inputs": [
      {
        "internalType": "address",
        "name": "user",
        "type": "address"
      },
      {
        "internalType": "address",
        "name": "tokenAddress",
        "type": "address"
      }
    ],
    "name": "getBalance",
    "outputs": [
      {
        "internalType": "uint256",
        "name": "",
        "type": "uint256"
      }
    ],
    "stateMutability": "view",
    "type": "function"
  },
  {
    "inputs": [
      {
        "internalType": "address",
        "name": "",
        "type": "address"
      },
      {
        "internalType": "address",
        "name": "",
        "type": "address"
      }
    ],
    "name": "lastWithdrawTime",
    "outputs": [
      {
        "internalType": "uint256",
        "name": "",
        "type": "uint256"
      }
    ],
    "stateMutability": "view",
    "type": "function"
  }
]

// 智能钱包地址
const SMART_WALLET_ADDRESS = '0xf954A73f1A972Ff6Bb6E21bb7d5496b9C3B16750'

// 响应式状态
const provider = ref(null)
const signer = ref(null)
const userAddress = ref('')
const tokenContract = ref(null)
const walletContract = ref(null)
const selectedToken = ref(tokens[0]) // 默认选择第一个代币 (USDT)
const balance = ref('0')
const walletBalance = ref('0')
const lastWithdrawTime = ref(0)
const depositAmount = ref('')
const withdrawAmount = ref('')
const toAddress = ref('')
const transferAmount = ref(0)
const txHash = ref('')
const isConnected = ref(false)
const isConnecting = ref(false)
const isProcessing = ref(false)
const errorMessage = ref('')
const successMessage = ref('')
const withdrawLimitInfo = ref('')

// Sepolia Etherscan 交易链接
const explorerUrl = computed(() => `https://sepolia.etherscan.io/tx/${txHash.value}`)

// 格式化时间显示
const formatTime = (timestamp) => {
  if (!timestamp) return '无记录'
  return new Date(timestamp * 1000).toLocaleString()
}

// 代币变更处理
const onTokenChange = async () => {
  if (!signer.value) return

  try {
    // 重新初始化代币合约
    tokenContract.value = new ethers.Contract(selectedToken.value.address, tokenABI, signer.value)

    // 更新余额信息
    await updateAll()
  } catch (error) {
    console.error('切换代币错误:', error)
    errorMessage.value = `切换代币失败: ${error.message}`
  }
}

// 连接 OKX 钱包
async function connectWallet() {
  let ethereumProvider = window.okxwallet || window.ethereum
  if (!ethereumProvider) {
    errorMessage.value = '请安装 OKX Wallet 或 MetaMask！'
    console.error('钱包未检测到')
    return
  }

  try {
    isConnecting.value = true
    errorMessage.value = ''

    // 请求账户授权
    await ethereumProvider.request({ method: 'eth_requestAccounts' })

    // 初始化 Web3Provider，禁用 ENS
    provider.value = markRaw(new ethers.providers.Web3Provider(ethereumProvider, {
      chainId: 11155111, // Sepolia chainId
      name: 'sepolia',
      ensAddress: null // 显式禁用 ENS
    }))
    signer.value = provider.value.getSigner()
    userAddress.value = await signer.value.getAddress()

    // 简化网络检测，绕过 getNetwork
    let chainId
    try {
      chainId = await ethereumProvider.request({ method: 'eth_chainId' })
      chainId = parseInt(chainId, 16)
    } catch (error) {
      console.warn('查询 eth_chainId 失败:', error)
      throw new Error('无法获取网络信息，请检查 OKX Wallet 设置')
    }

    if (chainId !== 11155111) {
      try {
        await ethereumProvider.request({
          method: 'wallet_switchEthereumChain',
          params: [{ chainId: '0xAA36A7' }]
        })
        chainId = parseInt(await ethereumProvider.request({ method: 'eth_chainId' }), 16)
      } catch (switchError) {
        if (switchError.code === 4902) {
          await ethereumProvider.request({
            method: 'wallet_addEthereumChain',
            params: [{
              chainId: '0xAA36A7',
              chainName: 'Sepolia Testnet',
              rpcUrls: ['https://rpc.sepolia.org'],
              nativeCurrency: { name: 'SepoliaETH', symbol: 'SEPETH', decimals: 18 },
              blockExplorerUrls: ['https://sepolia.etherscan.io']
            }]
          })
          chainId = parseInt(await ethereumProvider.request({ method: 'eth_chainId' }), 16)
        } else {
          throw switchError
        }
      }
    }

    if (chainId !== 11155111) {
      throw new Error('无法切换到 Sepolia 测试网，请手动检查 OKX Wallet 网络设置')
    }

    isConnected.value = true

    // 验证智能钱包地址
    if (!ethers.utils.isAddress(SMART_WALLET_ADDRESS)) {
      errorMessage.value = '无效的智能钱包地址，请检查 SMART_WALLET_ADDRESS'
      return
    }

    // 初始化合约
    tokenContract.value = new ethers.Contract(selectedToken.value.address, tokenABI, signer.value)
    walletContract.value = new ethers.Contract(SMART_WALLET_ADDRESS, walletABI, signer.value)

    await updateAll()
    successMessage.value = '钱包连接成功！'
  } catch (error) {
    errorMessage.value = `连接失败：${error.message || '未知错误'}`
    console.error('连接钱包错误:', error)
  } finally {
    isConnecting.value = false
  }
}

// 更新所有信息
async function updateAll() {
  await updateBalances()
  await updateLastWithdrawTime()
  await updateWithdrawLimitInfo()
}

// 更新余额
async function updateBalances() {
  if (!signer.value) return
  try {
    const userBal = await tokenContract.value.balanceOf(userAddress.value)
    balance.value = ethers.utils.formatUnits(userBal, selectedToken.value.decimals)

    const walletBal = await walletContract.value.getBalance(userAddress.value, selectedToken.value.address)
    walletBalance.value = ethers.utils.formatUnits(walletBal, selectedToken.value.decimals)
  } catch (error) {
    console.error('更新余额错误:', error)
  }
}

// 更新上次提取时间
async function updateLastWithdrawTime() {
  if (!signer.value) return
  try {
    lastWithdrawTime.value = await walletContract.value.lastWithdrawTime(userAddress.value, selectedToken.value.address)
  } catch (error) {
    console.error('更新提取时间错误:', error)
    lastWithdrawTime.value = 0
  }
}

// 更新提取限制信息
async function updateWithdrawLimitInfo() {
  if (!signer.value) return

  try {
    const userBal = await walletContract.value.getBalance(userAddress.value, selectedToken.value.address)
    const lastTime = lastWithdrawTime.value
    const currentTime = Math.floor(Date.now() / 1000)

    const timeSinceLastWithdraw = lastTime === 0 ? currentTime : currentTime - lastTime

    if (timeSinceLastWithdraw >= 30 * 24 * 60 * 60) {
      withdrawLimitInfo.value = '可提取全部余额'
    } else if (timeSinceLastWithdraw >= 24 * 60 * 60) {
      const thirtyPercent = userBal.mul(30).div(100)
      const formattedAmount = ethers.utils.formatUnits(thirtyPercent, selectedToken.value.decimals)
      withdrawLimitInfo.value = `24小时内最多可提取${formattedAmount} ${selectedToken.value.symbol} (30%)`
    } else {
      const remainingTime = 24 * 60 * 60 - timeSinceLastWithdraw
      const hours = Math.floor(remainingTime / 3600)
      const minutes = Math.floor((remainingTime % 3600) / 60)
      withdrawLimitInfo.value = `还需等待 ${hours}小时${minutes}分钟 才能再次提取`
    }
  } catch (error) {
    console.error('更新提取限制信息错误:', error)
    withdrawLimitInfo.value = '无法获取提取限制信息'
  }
}

// 检查提取条件
async function checkWithdrawConditions(amount) {
  try {
    const amountWei = ethers.utils.parseUnits(amount.toString(), selectedToken.value.decimals)
    const userBal = await walletContract.value.getBalance(userAddress.value, selectedToken.value.address)

    // 获取上次提取时间
    const lastTime = lastWithdrawTime.value
    const currentTime = Math.floor(Date.now() / 1000)

    // 计算距离上次提取的时间（秒）
    const timeSinceLastWithdraw = lastTime === 0 ? currentTime : currentTime - lastTime

    // 检查是否满足30天条件
    if (timeSinceLastWithdraw >= 30 * 24 * 60 * 60) {
      return userBal.gte(amountWei) // 30天后可全额提取
    }

    // 检查是否满足24小时条件
    if (timeSinceLastWithdraw >= 24 * 60 * 60) {
      const thirtyPercent = userBal.mul(30).div(100)
      return amountWei.lte(thirtyPercent) && userBal.gte(amountWei) // 24小时后可提取30%
    }

    // 不足24小时，不能提取
    return false
  } catch (error) {
    console.error('检查提取条件错误:', error)
    return false
  }
}

// 授权代币
async function approveToken(amount) {
  if (!signer.value) {
    errorMessage.value = '请先连接钱包'
    return false
  }

  const amountWei = ethers.utils.parseUnits(amount.toString(), selectedToken.value.decimals)
  try {
    // 检查当前授权额度
    const allowance = await tokenContract.value.allowance(userAddress.value, SMART_WALLET_ADDRESS)

    // 如果已有足够授权，直接返回
    if (allowance.gte(amountWei)) {
      console.log('已有足够授权，无需再次授权')
      return true
    }

    // 如果授权不足，先尝试重置授权为0（避免某些代币需要先重置）
    if (!allowance.isZero()) {
      console.log('重置授权为0...')
      const resetTx = await tokenContract.value.approve(SMART_WALLET_ADDRESS, 0)
      await resetTx.wait()
    }

    // 进行新授权
    console.log('进行新授权...')
    const tx = await tokenContract.value.approve(SMART_WALLET_ADDRESS, amountWei)
    successMessage.value = `正在授权... 交易哈希：${tx.hash}`
    await tx.wait()
    successMessage.value = '授权成功'
    return true
  } catch (error) {
    errorMessage.value = `授权失败：${error.message}`
    console.error('授权错误:', error)
    return false
  }
}

// 存入代币
async function deposit() {
  if (!signer.value) {
    errorMessage.value = '请先连接钱包'
    return
  }
  if (!depositAmount.value || isNaN(depositAmount.value) || depositAmount.value <= 0) {
    errorMessage.value = '请输入有效的存款金额'
    return
  }

  try {
    isProcessing.value = true
    errorMessage.value = ''
    successMessage.value = ''

    const amountWei = ethers.utils.parseUnits(depositAmount.value.toString(), selectedToken.value.decimals)
    const userBal = await tokenContract.value.balanceOf(userAddress.value)

    if (userBal.lt(amountWei)) {
      errorMessage.value = '余额不足'
      return
    }

    const approved = await approveToken(depositAmount.value)
    if (!approved) return

    const tx = await walletContract.value.deposit(selectedToken.value.address, amountWei)
    successMessage.value = `正在存款... 交易哈希：${tx.hash}`
    await tx.wait()
    successMessage.value = '存款成功'
    await updateAll()
    depositAmount.value = ''
  } catch (error) {
    errorMessage.value = `存款失败：${error.message}`
    console.error('存款错误:', error)
  } finally {
    isProcessing.value = false
  }
}

// 提取代币
async function withdraw() {
  if (!signer.value) {
    errorMessage.value = '请先连接钱包'
    return
  }
  if (!withdrawAmount.value || isNaN(withdrawAmount.value) || withdrawAmount.value <= 0) {
    errorMessage.value = '请输入有效的提取金额'
    return
  }

  try {
    isProcessing.value = true
    errorMessage.value = ''
    successMessage.value = ''

    // 检查提取条件
    const canWithdraw = await checkWithdrawConditions(withdrawAmount.value)
    if (!canWithdraw) {
      errorMessage.value = '提取条件不满足：可能未满足时间限制或金额超过允许范围'
      return
    }

    const amountWei = ethers.utils.parseUnits(withdrawAmount.value.toString(), selectedToken.value.decimals)
    const tx = await walletContract.value.withdraw(selectedToken.value.address, amountWei)

    successMessage.value = `正在提取... 交易哈希：${tx.hash}`
    await tx.wait()
    successMessage.value = '提取成功'
    await updateAll()
    withdrawAmount.value = ''
  } catch (error) {
    errorMessage.value = `提取失败：${error.message}`
    console.error('提取错误:', error)
  } finally {
    isProcessing.value = false
  }
}

// 直接转账
async function transferToken() {
  if (!signer.value) {
    errorMessage.value = '请先连接钱包'
    return
  }
  if (!ethers.utils.isAddress(toAddress.value) || !transferAmount.value || transferAmount.value <= 0) {
    errorMessage.value = '请输入有效的接收地址和金额'
    return
  }

  try {
    isProcessing.value = true
    errorMessage.value = ''
    successMessage.value = ''

    const amountWei = ethers.utils.parseUnits(transferAmount.value.toString(), selectedToken.value.decimals)
    const userBal = await tokenContract.value.balanceOf(userAddress.value)

    if (userBal.lt(amountWei)) {
      errorMessage.value = '余额不足'
      return
    }

    const tx = await tokenContract.value.transfer(toAddress.value, amountWei)
    successMessage.value = `正在转账... 交易哈希：${tx.hash}`
    await tx.wait()
    txHash.value = tx.hash
    successMessage.value = '转账成功'
    await updateBalances()
    toAddress.value = ''
    transferAmount.value = 0
  } catch (error) {
    errorMessage.value = `转账失败：${error.message}`
    console.error('转账错误:', error)
  } finally {
    isProcessing.value = false
  }
}

// 页面加载时检查是否已连接钱包
onMounted(async () => {
  const ethereumProvider = window.okxwallet || window.ethereum
  if (ethereumProvider && ethereumProvider.selectedAddress) {
    // 如果钱包已连接，自动连接
    await connectWallet()
  }
})
</script>

<style scoped>
.wallet-container {
  min-height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  background-color: #f0f2f5;
  padding: 20px;
}

.card {
  background-color: white;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  padding: 24px;
  max-width: 500px;
  width: 100%;
  text-align: center;
}

h1 {
  font-size: 24px;
  font-weight: bold;
  margin-bottom: 20px;
}

h2 {
  font-size: 18px;
  font-weight: 600;
  margin-bottom: 10px;
}

.section {
  margin-bottom: 20px;
  padding: 15px;
  border: 1px solid #eaeaea;
  border-radius: 8px;
}

.address {
  font-size: 14px;
  margin-bottom: 20px;
  color: #333;
  word-break: break-all;
}

.input {
  width: 100%;
  padding: 8px;
  margin-bottom: 10px;
  border: 1px solid #ccc;
  border-radius: 4px;
  font-size: 14px;
}

.btn {
  width: 100%;
  padding: 10px;
  border: none;
  border-radius: 4px;
  color: white;
  font-size: 16px;
  cursor: pointer;
  margin-top: 10px;
}

.btn-primary {
  background-color: #007bff;
}

.btn-primary:hover:not(:disabled) {
  background-color: #0056b3;
}

.btn-success {
  background-color: #28a745;
}

.btn-success:hover:not(:disabled) {
  background-color: #1e7e34;
}

.btn-danger {
  background-color: #dc3545;
}

.btn-danger:hover:not(:disabled) {
  background-color: #b02a37;
}

.btn:disabled {
  background-color: #6c757d;
  cursor: not-allowed;
}

.success {
  color: #28a745;
  margin-top: 10px;
}

.error {
  color: #dc3545;
  margin-top: 10px;
}

.tx-link a {
  color: #007bff;
  text-decoration: none;
}

.tx-link a:hover {
  text-decoration: underline;
}

.info-box {
  background-color: #f8f9fa;
  padding: 10px;
  border-radius: 4px;
  margin-top: 10px;
  font-size: 14px;
}
</style>
