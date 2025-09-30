# 🚀 Server Setup & Deployment Guide

## 🔑 Connect to Server
```bash
ssh root@your-server-ip
```

---

## 📦 Update System
```bash
apt update && apt upgrade -y
```

---

## 🛠 Install Dependencies

### Install Node.js (v18+ required)
```bash
curl -fsSL https://deb.nodesource.com/setup_18.x | bash -
apt-get install -y nodejs
```

### Install Git
```bash
apt install git -y
```

### Install Nginx (for frontend)
```bash
apt install nginx -y
```

### Configure Firewall
```bash
ufw allow 22    # SSH
ufw allow 80    # HTTP
ufw allow 443   # HTTPS
ufw enable
```

---

## 📂 Create Project Directory
```bash
mkdir zama-dapp
cd zama-dapp
```

---

## 📥 Clone Zama Template
```bash
git clone https://github.com/zama-ai/fhevm-hardhat-template .
```

---

## 📦 Install Dependencies & FHEVM SDK
```bash
npm install
npm install @fhevm @fhevm-web3sdk/fhevm-web3sdk
```

---

## ⚙️ Configure Hardhat
Edit `hardhat.config.js`:

```js
require("@nomicfoundation/hardhat-toolbox");

/** @type import('hardhat/config').HardhatUserConfig */
module.exports = {
  solidity: {
    version: "0.8.20",
    settings: {
      optimizer: { enabled: true, runs: 200 },
    },
  },
  networks: {
    // Zama Testnet
    zamatest: {
      url: "https://devnet.zama.ai",
      chainId: 8009,
      accounts: process.env.PRIVATE_KEY ? [process.env.PRIVATE_KEY] : [],
    },
    // Local development
    localhost: {
      url: "http://127.0.0.1:8545",
      chainId: 31337,
    }
  },
  paths: {
    sources: "./contracts",
    tests: "./test",
    cache: "./cache",
    artifacts: "./artifacts"
  },
};
```

---

## 📜 Create Deployment Script
`scripts/deploy.js`:

```js
const { ethers } = require("hardhat");
const { FhevmInstances } = require("@fhevm/web3sdk");

async function main() {
  console.log("🚀 Deploying Zama EncryptedNotebook dApp...");
  
  const EncryptedNotebook = await ethers.getContractFactory("EncryptedNotebook");
  const notebook = await EncryptedNotebook.deploy();
  await notebook.waitForDeployment();

  const address = await notebook.getAddress();
  console.log("✅ EncryptedNotebook deployed at:", address);
  console.log("📝 Tx hash:", notebook.deploymentTransaction().hash);
  
  const instance = await FhevmInstances.createInstance(ethers.provider);
  console.log("🔑 FHEVM Instance created successfully");
  
  return { notebook, instance, address };
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error("❌ Deployment failed:", error);
    process.exit(1);
  });
```

---

## 🔧 Environment Variables
Create `.env` file:

```bash
# Sepolia RPC URL (Infura or Alchemy)
SEPOLIA_RPC_URL=https://eth-sepolia.g.alchemy.com

# Private Key (0x...)
PRIVATE_KEY=0x2fc6888

# Etherscan API Key (for contract verification)
ETHERSCAN_API_KEY=

# Frontend port
FRONTEND_PORT=5300
```

---

## 🚀 Deploy Contract
```bash
npx hardhat run scripts/deploy.js --network zamatest
```

---

## 🎨 Frontend Setup
```bash
mkdir zama-frontend
cd zama-frontend
npx create-react-app .
npm install ethers @fhevm/web3sdk axios
```


## Then edit `src/App.js` with the provided frontend code.  


```bash
import React, { useState, useEffect } from 'react';
import { ethers } from 'ethers';
import './App.css';

import SimpleVotingABI from './contracts/SimpleVoting.json';
import contractAddress from './contracts/contract-address.json';

function App() {
  const [contract, setContract] = useState(null);
  const [account, setAccount] = useState('');
  const [votings, setVotings] = useState([]);
  const [loading, setLoading] = useState(false);
  const [activeTab, setActiveTab] = useState('votings');

  const [newVoting, setNewVoting] = useState({
    title: '',
    description: '',
    options: ['Yes', 'No'],
    duration: 24
  });

  // Check and switch to Sepolia network
  const ensureSepoliaNetwork = async () => {
    const sepoliaChainId = '0xaa36a7';
    
    try {
      const currentChainId = await window.ethereum.request({ 
        method: 'eth_chainId' 
      });

      if (currentChainId !== sepoliaChainId) {
        try {
          await window.ethereum.request({
            method: 'wallet_switchEthereumChain',
            params: [{ chainId: sepoliaChainId }],
          });
        } catch (switchError) {
          if (switchError.code === 4902) {
            await window.ethereum.request({
              method: 'wallet_addEthereumChain',
              params: [
                {
                  chainId: sepoliaChainId,
                  chainName: 'Sepolia Test Network',
                  rpcUrls: ['https://sepolia.infura.io/v3/'],
                  blockExplorerUrls: ['https://sepolia.etherscan.io'],
                  nativeCurrency: {
                    name: 'Sepolia ETH',
                    symbol: 'ETH',
                    decimals: 18,
                  },
                },
              ],
            });
          }
        }
      }
      return true;
    } catch (error) {
      console.error('Network switch error:', error);
      return false;
    }
  };

  // Load all votings from contract
  const loadVotings = async (votingContract = contract) => {
    if (!votingContract) {
      console.log("No contract available");
      return;
    }

    try {
      setLoading(true);
      console.log("Loading votings from contract:", contractAddress.SimpleVoting);
      
      const count = await votingContract.votingCount();
      console.log("Total votings:", Number(count));
      
      const votingList = [];

      for (let i = 0; i < Number(count); i++) {
        try {
          const info = await votingContract.getVotingInfo(i);
          const hasVoted = account ? await votingContract.hasUserVoted(i, account) : false;
          
          const votingData = {
            id: i,
            title: info.title || 'Untitled',
            description: info.description || '',
            options: info.options || [],
            creator: info.creator,
            endTime: new Date(Number(info.endTime) * 1000),
            isActive: info.isActive && (Number(info.endTime) * 1000 > Date.now()),
            hasVoted: hasVoted,
            isCreator: account ? info.creator.toLowerCase() === account.toLowerCase() : false
          };

          votingList.push(votingData);
        } catch (error) {
          console.warn(`Error loading voting ${i}:`, error);
          continue;
        }
      }

      console.log("Loaded votings:", votingList.length);
      setVotings(votingList);
      
      // Cache to localStorage
      localStorage.setItem('cachedVotings', JSON.stringify({
        data: votingList,
        timestamp: Date.now(),
        contractAddress: contractAddress.SimpleVoting
      }));
      
    } catch (error) {
      console.error("Error loading votings:", error);
      
      // Try to load from cache
      const cached = localStorage.getItem('cachedVotings');
      if (cached) {
        const cachedData = JSON.parse(cached);
        if (cachedData.contractAddress === contractAddress.SimpleVoting) {
          setVotings(cachedData.data);
        }
      }
    } finally {
      setLoading(false);
    }
  };

  // Connect wallet
  const connectWallet = async () => {
    if (window.ethereum) {
      try {
        // Ensure we're on Sepolia
        const networkOk = await ensureSepoliaNetwork();
        if (!networkOk) {
          alert('Please switch to Sepolia network manually in MetaMask.');
          return;
        }

        // Get accounts
        const accounts = await window.ethereum.request({ 
          method: 'eth_requestAccounts' 
        });
        
        const userAddress = accounts[0];
        console.log("Connected:", userAddress);
        setAccount(userAddress);

        // Create provider and signer
        const provider = new ethers.BrowserProvider(window.ethereum);
        const signer = await provider.getSigner();

        // Create contract instance
        const votingContract = new ethers.Contract(
          contractAddress.SimpleVoting,
          SimpleVotingABI.abi,
          signer
        );
        
        setContract(votingContract);
        console.log("Contract instance created");

        // Load votings
        await loadVotings(votingContract);
        
      } catch (error) {
        console.error("Wallet connection failed:", error);
        alert("Connection failed: " + error.message);
      }
    } else {
      alert("Please install MetaMask!");
    }
  };

  // Auto-connect on page load
  useEffect(() => {
    const autoConnect = async () => {
      if (window.ethereum) {
        try {
          const accounts = await window.ethereum.request({ 
            method: 'eth_accounts' 
          });
          
          if (accounts.length > 0) {
            console.log("Auto-connecting to:", accounts[0]);
            setAccount(accounts[0]);
            
            const provider = new ethers.BrowserProvider(window.ethereum);
            const signer = await provider.getSigner();
            const votingContract = new ethers.Contract(
              contractAddress.SimpleVoting,
              SimpleVotingABI.abi,
              signer
            );
            
            setContract(votingContract);
            await loadVotings(votingContract);
          }
        } catch (error) {
          console.error("Auto-connect error:", error);
        }
      }
    };

    autoConnect();
  }, []);

  // Listen for account changes
  useEffect(() => {
    if (window.ethereum) {
      const handleAccountsChanged = (accounts) => {
        if (accounts.length === 0) {
          setAccount('');
          setContract(null);
          setVotings([]);
        } else if (accounts[0] !== account) {
          setAccount(accounts[0]);
          // Contract remains the same, just reload votings for new account
          if (contract) {
            loadVotings();
          }
        }
      };

      const handleChainChanged = () => {
        window.location.reload();
      };

      window.ethereum.on('accountsChanged', handleAccountsChanged);
      window.ethereum.on('chainChanged', handleChainChanged);

      return () => {
        window.ethereum.removeListener('accountsChanged', handleAccountsChanged);
        window.ethereum.removeListener('chainChanged', handleChainChanged);
      };
    }
  }, [contract, account]);

  // Create new voting
  const createVoting = async () => {
    if (!contract) return;

    try {
      setLoading(true);
      const durationInSeconds = newVoting.duration * 3600;

      const tx = await contract.createVoting(
        newVoting.title.trim(),
        newVoting.description.trim(),
        newVoting.options.filter(opt => opt.trim()),
        durationInSeconds
      );

      await tx.wait();
      alert("Voting created successfully!");
      
      setNewVoting({
        title: '',
        description: '',
        options: ['Yes', 'No'],
        duration: 24
      });

      setActiveTab('votings');
      await loadVotings();
      
    } catch (error) {
      console.error("Error creating voting:", error);
      alert("Error: " + error.message);
    } finally {
      setLoading(false);
    }
  };

  // Cast vote
  const castVote = async (votingId, optionIndex) => {
    if (!contract) return;

    try {
      setLoading(true);
      const tx = await contract.castVote(votingId, optionIndex);
      await tx.wait();
      
      alert("Vote cast successfully!");
      await loadVotings();
    } catch (error) {
      console.error("Error casting vote:", error);
      alert("Error: " + error.message);
    } finally {
      setLoading(false);
    }
  };

  // View results
  const viewResults = async (votingId) => {
    if (!contract) return;

    try {
      const results = await contract.getResults(votingId);
      const voting = votings.find(v => v.id === votingId);
      
      let resultText = "Voting Results:\n\n";
      voting.options.forEach((option, index) => {
        resultText += `${option}: ${results[index]} votes\n`;
      });
      
      alert(resultText);
    } catch (error) {
      console.error("Error viewing results:", error);
      alert("Error: " + error.message);
    }
  };

  // Form helpers
  const addOption = () => {
    setNewVoting(prev => ({
      ...prev,
      options: [...prev.options, '']
    }));
  };

  const updateOption = (index, value) => {
    setNewVoting(prev => ({
      ...prev,
      options: prev.options.map((opt, i) => i === index ? value : opt)
    }));
  };

  const removeOption = (index) => {
    if (newVoting.options.length <= 2) {
      alert("Minimum 2 options required");
      return;
    }
    setNewVoting(prev => ({
      ...prev,
      options: prev.options.filter((_, i) => i !== index)
    }));
  };

  return (
    <div className="App">
      <header className="app-header">
        <div className="header-content">
          <div className="logo">
            <h1>🗳️ SecureVote</h1>
            <p>Decentralized Voting Platform</p>
          </div>
          
          {!account ? (
            <button onClick={connectWallet} className="connect-btn">
              Connect Wallet
            </button>
          ) : (
            <div className="wallet-info">
              <div className="account-address">
                {account.slice(0, 6)}...{account.slice(-4)}
              </div>
              <div className="network-badge">Sepolia</div>
              <button 
                onClick={() => window.open('https://sepoliafaucet.com', '_blank')}
                className="faucet-btn"
                title="Get Sepolia ETH"
              >
                🚰 Faucet
              </button>
            </div>
          )}
        </div>

        <nav className="navigation">
          <button 
            className={`nav-btn ${activeTab === 'votings' ? 'active' : ''}`}
            onClick={() => setActiveTab('votings')}
          >
            Active Votings
          </button>
          <button 
            className={`nav-btn ${activeTab === 'create' ? 'active' : ''}`}
            onClick={() => setActiveTab('create')}
          >
            Create Voting
          </button>
        </nav>
      </header>

      <div className="container">
        {activeTab === 'create' && account && (
          <div className="create-voting-section">
            <div className="section-header">
              <h2>Create New Voting</h2>
              <p>Set up a new decentralized voting session</p>
            </div>
            
            <div className="form-card">
              <div className="form-group">
                <label>Voting Title *</label>
                <input
                  type="text"
                  placeholder="Enter voting title..."
                  value={newVoting.title}
                  onChange={(e) => setNewVoting({...newVoting, title: e.target.value})}
                  className="form-input"
                />
              </div>

              <div className="form-group">
                <label>Description</label>
                <textarea
                  placeholder="Describe the purpose of this voting..."
                  value={newVoting.description}
                  onChange={(e) => setNewVoting({...newVoting, description: e.target.value})}
                  className="form-textarea"
                  rows="3"
                />
              </div>

              <div className="form-group">
                <label>Voting Options *</label>
                <div className="options-list">
                  {newVoting.options.map((option, index) => (
                    <div key={index} className="option-item">
                      <input
                        type="text"
                        placeholder={`Option ${index + 1}`}
                        value={option}
                        onChange={(e) => updateOption(index, e.target.value)}
                        className="form-input"
                      />
                      {newVoting.options.length > 2 && (
                        <button 
                          type="button" 
                          onClick={() => removeOption(index)}
                          className="remove-option-btn"
                        >
                          ×
                        </button>
                      )}
                    </div>
                  ))}
                </div>
                <button type="button" onClick={addOption} className="add-option-btn">
                  + Add Option
                </button>
              </div>

              <div className="form-group">
                <label>Voting Duration (hours) *</label>
                <input
                  type="number"
                  min="1"
                  max="720"
                  value={newVoting.duration}
                  onChange={(e) => setNewVoting({...newVoting, duration: parseInt(e.target.value) || 24})}
                  className="form-input"
                />
              </div>

              <button 
                onClick={createVoting} 
                disabled={loading || !newVoting.title.trim() || newVoting.options.some(opt => !opt.trim())}
                className="submit-btn"
              >
                {loading ? "Creating..." : "Create Voting Session"}
              </button>
            </div>
          </div>
        )}

        {activeTab === 'votings' && (
          <div className="votings-section">
            <div className="section-header">
              <h2>Active Voting Sessions</h2>
              <p>Participate in ongoing decentralized votes</p>
            </div>

            {loading ? (
              <div className="loading-state">
                <div className="spinner"></div>
                <p>Loading voting sessions...</p>
              </div>
            ) : votings.length === 0 ? (
              <div className="empty-state">
                <div className="empty-icon">🗳️</div>
                <h3>No Active Votings</h3>
                <p>Create the first voting session to get started</p>
                {account && (
                  <button 
                    onClick={() => setActiveTab('create')} 
                    className="create-first-btn"
                  >
                    Create Voting Session
                  </button>
                )}
              </div>
            ) : (
              <div className="voting-grid">
                {votings.map((voting) => (
                  <div key={voting.id} className="voting-card">
                    <div className="card-header">
                      <h3>{voting.title}</h3>
                      <div className={`status-badge ${voting.isActive ? 'active' : 'ended'}`}>
                        {voting.isActive ? 'Active' : 'Ended'}
                      </div>
                    </div>
                    
                    <p className="card-description">{voting.description}</p>
                    
                    <div className="card-meta">
                      <div className="meta-item">
                        <span className="meta-label">Creator:</span>
                        <span className="meta-value">{voting.creator.slice(0, 8)}...{voting.creator.slice(-6)}</span>
                      </div>
                      <div className="meta-item">
                        <span className="meta-label">Ends:</span>
                        <span className="meta-value">{voting.endTime.toLocaleString()}</span>
                      </div>
                    </div>

                    {voting.isActive && !voting.hasVoted && (
                      <div className="vote-section">
                        <h4>Cast Your Vote:</h4>
                        <div className="vote-options-grid">
                          {voting.options.map((option, index) => (
                            <button
                              key={index}
                              onClick={() => castVote(voting.id, index)}
                              className="vote-option-btn"
                            >
                              {option}
                            </button>
                          ))}
                        </div>
                      </div>
                    )}

                    {voting.hasVoted && (
                      <div className="voted-indicator">
                        <span className="voted-icon">✓</span>
                        You have already voted in this session
                      </div>
                    )}

                    <div className="card-actions">
                      <button
                        onClick={() => viewResults(voting.id)}
                        className="results-btn"
                        disabled={voting.isActive && !voting.isCreator}
                      >
                        View Results
                      </button>
                    </div>
                  </div>
                ))}
              </div>
            )}
          </div>
        )}

        {!account && activeTab === 'votings' && (
          <div className="connect-prompt">
            <div className="prompt-content">
              <div className="prompt-icon">🔒</div>
              <h3>Connect Your Wallet</h3>
              <p>Please connect your wallet to view and participate in voting sessions</p>
              <button onClick={connectWallet} className="connect-prompt-btn">
                Connect Wallet
              </button>
            </div>
          </div>
        )}
      </div>
    </div>
  );
}

export default App;
```

**🎨 Then edit `src/App.css` file** 

```bash
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Roboto', sans-serif;
  background: #f8fafc;
  color: #334155;
  line-height: 1.6;
}

.App {
  min-height: 100vh;
}

/* Header Styles */
.app-header {
  background: white;
  border-bottom: 1px solid #e2e8f0;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.header-content {
  max-width: 1200px;
  margin: 0 auto;
  padding: 1rem 2rem;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.logo h1 {
  font-size: 1.75rem;
  font-weight: 700;
  color: #1e293b;
  margin-bottom: 0.25rem;
}

.logo p {
  color: #64748b;
  font-size: 0.9rem;
}

.wallet-info {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.account-address {
  background: #f1f5f9;
  padding: 0.5rem 1rem;
  border-radius: 8px;
  font-family: 'Monaco', 'Consolas', monospace;
  font-size: 0.9rem;
  color: #475569;
  border: 1px solid #e2e8f0;
}

.network-badge {
  background: #10b981;
  color: white;
  padding: 0.25rem 0.75rem;
  border-radius: 6px;
  font-size: 0.8rem;
  font-weight: 600;
}

.connect-btn {
  background: #3b82f6;
  color: white;
  border: none;
  padding: 0.75rem 1.5rem;
  border-radius: 8px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
}

.connect-btn:hover {
  background: #2563eb;
  transform: translateY(-1px);
}

/* Navigation */
.navigation {
  max-width: 1200px;
  margin: 0 auto;
  padding: 0 2rem;
  display: flex;
  gap: 0.5rem;
  border-top: 1px solid #f1f5f9;
}

.nav-btn {
  background: none;
  border: none;
  padding: 1rem 1.5rem;
  color: #64748b;
  font-weight: 500;
  cursor: pointer;
  border-bottom: 2px solid transparent;
  transition: all 0.2s;
}

.nav-btn:hover {
  color: #3b82f6;
}

.nav-btn.active {
  color: #3b82f6;
  border-bottom-color: #3b82f6;
}

/* Container */
.container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 2rem;
}

/* Section Headers */
.section-header {
  text-align: center;
  margin-bottom: 2rem;
}

.section-header h2 {
  font-size: 2rem;
  font-weight: 700;
  color: #1e293b;
  margin-bottom: 0.5rem;
}

.section-header p {
  color: #64748b;
  font-size: 1.1rem;
}

/* Form Styles */
.form-card {
  background: white;
  padding: 2rem;
  border-radius: 12px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.05);
  border: 1px solid #e2e8f0;
  max-width: 600px;
  margin: 0 auto;
}

.form-group {
  margin-bottom: 1.5rem;
}

.form-group label {
  display: block;
  margin-bottom: 0.5rem;
  font-weight: 600;
  color: #374151;
}

.form-input,
.form-textarea {
  width: 100%;
  padding: 0.75rem 1rem;
  border: 1px solid #d1d5db;
  border-radius: 8px;
  font-size: 1rem;
  transition: all 0.2s;
}

.form-input:focus,
.form-textarea:focus {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.form-textarea {
  resize: vertical;
  min-height: 80px;
}

/* Options List */
.options-list {
  margin-bottom: 1rem;
}

.option-item {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 0.5rem;
}

.option-item .form-input {
  flex: 1;
}

.remove-option-btn {
  background: #ef4444;
  color: white;
  border: none;
  width: 40px;
  border-radius: 6px;
  cursor: pointer;
  transition: background 0.2s;
}

.remove-option-btn:hover {
  background: #dc2626;
}

.add-option-btn {
  background: #f8fafc;
  color: #64748b;
  border: 1px dashed #d1d5db;
  padding: 0.5rem 1rem;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.2s;
}

.add-option-btn:hover {
  background: #f1f5f9;
  border-color: #9ca3af;
}

/* Buttons */
.submit-btn {
  width: 100%;
  background: #10b981;
  color: white;
  border: none;
  padding: 1rem 2rem;
  border-radius: 8px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
}

.submit-btn:hover:not(:disabled) {
  background: #059669;
  transform: translateY(-1px);
}

.submit-btn:disabled {
  background: #9ca3af;
  cursor: not-allowed;
  transform: none;
}

/* Voting Grid */
.voting-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(400px, 1fr));
  gap: 1.5rem;
}

.voting-card {
  background: white;
  padding: 1.5rem;
  border-radius: 12px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.05);
  border: 1px solid #e2e8f0;
  transition: all 0.2s;
}

.voting-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 15px rgba(0, 0, 0, 0.1);
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 1rem;
}

.card-header h3 {
  font-size: 1.25rem;
  font-weight: 600;
  color: #1e293b;
  flex: 1;
  margin-right: 1rem;
}

.status-badge {
  padding: 0.25rem 0.75rem;
  border-radius: 6px;
  font-size: 0.8rem;
  font-weight: 600;
}

.status-badge.active {
  background: #d1fae5;
  color: #065f46;
}

.status-badge.ended {
  background: #fef3c7;
  color: #92400e;
}

.card-description {
  color: #64748b;
  margin-bottom: 1rem;
  line-height: 1.5;
}

.card-meta {
  background: #f8fafc;
  padding: 1rem;
  border-radius: 8px;
  margin-bottom: 1rem;
}

.meta-item {
  display: flex;
  justify-content: space-between;
  margin-bottom: 0.5rem;
}

.meta-item:last-child {
  margin-bottom: 0;
}

.meta-label {
  font-weight: 500;
  color: #64748b;
}

.meta-value {
  color: #374151;
  font-family: 'Monaco', 'Consolas', monospace;
  font-size: 0.9rem;
}

/* Vote Section */
.vote-section h4 {
  margin-bottom: 0.75rem;
  color: #374151;
  font-weight: 600;
}

.vote-options-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(120px, 1fr));
  gap: 0.5rem;
  margin-bottom: 1rem;
}

.vote-option-btn {
  background: #f8fafc;
  color: #374151;
  border: 1px solid #e2e8f0;
  padding: 0.75rem 1rem;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s;
  font-weight: 500;
}

.vote-option-btn:hover {
  background: #3b82f6;
  color: white;
  border-color: #3b82f6;
}

/* Voted Indicator */
.voted-indicator {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  background: #d1fae5;
  color: #065f46;
  padding: 0.75rem 1rem;
  border-radius: 8px;
  margin-bottom: 1rem;
  font-weight: 500;
}

.voted-icon {
  background: #10b981;
  color: white;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.8rem;
}

/* Card Actions */
.card-actions {
  display: flex;
  gap: 0.5rem;
}

.results-btn {
  flex: 1;
  background: #3b82f6;
  color: white;
  border: none;
  padding: 0.75rem 1rem;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s;
  font-weight: 500;
}

.results-btn:hover:not(:disabled) {
  background: #2563eb;
}

.results-btn:disabled {
  background: #9ca3af;
  cursor: not-allowed;
}

/* Loading State */
.loading-state {
  text-align: center;
  padding: 3rem;
}

.spinner {
  border: 3px solid #f1f5f9;
  border-top: 3px solid #3b82f6;
  border-radius: 50%;
  width: 40px;
  height: 40px;
  animation: spin 1s linear infinite;
  margin: 0 auto 1rem;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

/* Empty State */
.empty-state {
  text-align: center;
  padding: 4rem 2rem;
}

.empty-icon {
  font-size: 4rem;
  margin-bottom: 1rem;
}

.empty-state h3 {
  font-size: 1.5rem;
  color: #374151;
  margin-bottom: 0.5rem;
}

.empty-state p {
  color: #64748b;
  margin-bottom: 2rem;
}

.create-first-btn {
  background: #3b82f6;
  color: white;
  border: none;
  padding: 0.75rem 1.5rem;
  border-radius: 8px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
}

.create-first-btn:hover {
  background: #2563eb;
}

/* Connect Prompt */
.connect-prompt {
  text-align: center;
  padding: 4rem 2rem;
}

.prompt-content {
  max-width: 400px;
  margin: 0 auto;
}

.prompt-icon {
  font-size: 4rem;
  margin-bottom: 1rem;
}

.prompt-content h3 {
  font-size: 1.5rem;
  color: #374151;
  margin-bottom: 0.5rem;
}

.prompt-content p {
  color: #64748b;
  margin-bottom: 2rem;
}

.connect-prompt-btn {
  background: #3b82f6;
  color: white;
  border: none;
  padding: 0.75rem 2rem;
  border-radius: 8px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
}

.connect-prompt-btn:hover {
  background: #2563eb;
}

.faucet-btn {
  background: #f59e0b;
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 6px;
  font-size: 0.8rem;
  cursor: pointer;
  transition: background 0.2s;
}

.faucet-btn:hover {
  background: #d97706;
}

/* Icon styles */
.fas, .fab {
  margin-right: 0.5rem;
}

/* Delete button */
.delete-btn {
  background: #ef4444;
  color: white;
  border: none;
  padding: 0.75rem 1rem;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s;
  font-weight: 500;
  flex: 1;
}

.delete-btn:hover {
  background: #dc2626;
}

/* Card actions layout */
.card-actions {
  display: flex;
  gap: 0.5rem;
}

.card-actions .results-btn,
.card-actions .delete-btn {
  flex: 1;
}

/* Emoji fallback */
.emoji-fallback {
  font-family: "Apple Color Emoji", "Segoe UI Emoji", "Segoe UI Symbol", "Noto Color Emoji";
}

/* Loading spinner for Font Awesome */
.fa-spin {
  animation: fa-spin 1s infinite linear;
}

@keyframes fa-spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

/* Responsive Design */
@media (max-width: 768px) {
  .header-content {
    padding: 1rem;
    flex-direction: column;
    gap: 1rem;
  }

  .navigation {
    padding: 0 1rem;
  }

  .container {
    padding: 1rem;
  }

  .voting-grid {
    grid-template-columns: 1fr;
  }

  .form-card {
    padding: 1.5rem;
  }

  .card-header {
    flex-direction: column;
    gap: 0.5rem;
  }

  .status-badge {
    align-self: flex-start;
  }
}
```
** It's time to start

```bash
PORT=4300  npm start
```

Technologies Used
Solidity (FHEVM)

Hardhat
React.js
Ethers.js
Zama DevNet
