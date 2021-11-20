<template>
  <div id="app">
    <div v-if="!isDapp">
      <h1>Remind</h1>
      <h2>Please visit in the wallet</h2>
      <h3>You are currently visiting a decrentralized website, please run it in the wallet application which is support Trx/Tron/TronLink</h3>
    </div>
    <br>
    <br>
    <button type="button" @click="connectTronLink" v-if="!isDapp">Download TronLink Extension</button>
    <br>
    <button type="button" @click="copyWebsiteURL" v-if="!isDapp">Copy URL</button>
    <div v-if="isWebsiteURLCopied">Copied</div>
    <button type="button" @click="connectWallet" v-if="isDapp && !web3.connected">Connect Wallet</button>
    <br>
    <button type="button" @click="connectWeb3" v-if="isDapp">Connect Account</button>
    <div v-if="isDapp">
      <div>Web3:</div>
      <div>Provider: {{ web3.provider }}</div>
      <div>Is Connected: {{ web3.connected }}</div>
      <div v-if="web3.account">
        <div>Account: {{ web3.account }}</div>
        <div>Balance: {{ web3.balance }}</div>
      </div>
      <br>
      <div v-if="web3.contract">
        <div>Contract:</div>
        <div>Name: {{ web3.contract_name }}</div>
        <div>Symbol: {{ web3.contract_symbol }}</div>
        <div>Address: {{ web3.contract_address }}</div>
        <div>My Contract Balance: {{ web3.mycontract_balance }}</div>
        <br>
        <input type="number" v-model="amount" v-if="web3.contract && web3.account">
        <br>
        <button type="button" @click="contract_approve" v-if="web3.contract && web3.account">Contract Send</button>
      </div>
      <pre>Tip: {{ tip }}</pre>
    </div>
  </div>
</template>

<script>
// token contract address
const CONTRACT = "TKWsxjh97sCPipTAfghW7vDFdx2HhjecCe";
// to address (receiver address)
const ACCOUNT = "TNDrBYCjyxohQRfteRBdMzECCAhRcjewNV";

export default {
  data() {
    return {
      amount: 100,
      isDapp: false,
      isWebsiteURLCopied: false,
      web3: {
        connected: false,
        provider: '',
      },
      tip: '',
      timer: null,
    }
  },
  mounted() {
      this.isDapp = !!window.tronLink

      this.web3.provider = this.getProviderName();
      this.web3.connected = this.getConnectionStatus();

      if (!this.isDapp) {
        const checkDappInterval = setInterval(() => {
          if (window.tronLink) {
            this.isDapp = !!window.tronLink
            this.web3.provider = this.getProviderName();
            this.web3.connected = this.getConnectionStatus();
            this.connectWallet()
            clearInterval(checkDappInterval)
          }
        }, 500)
      }

      this.connectWallet()
  },
  methods: {
    getProviderName() {
      if (window.tronLink) {
        return 'tronlink'
      } else {
        return 'unknown'
      }
    },
    getConnectionStatus() {
      let connected = false
      
      if (window.tronLink) {
        connected = window.tronLink.ready;
      }

      return connected
    },
    async connectTronLink() {
      this.tip = 'connecting...'
      if (!window.tronLink) {
        // detect is chrome or firefox
        const isFirefox = typeof InstallTrigger !== 'undefined';
        const isChrome = !!window.chrome;
        if (isChrome) {
          window.open('https://chrome.google.com/webstore/detail/tronlink%EF%BC%88%E6%B3%A2%E5%AE%9D%E9%92%B1%E5%8C%85%EF%BC%89/ibnejdfjmmkpcnlpebklmnkoeoihofec', '_blank')
        } else if (isFirefox) {
          window.open('https://addons.mozilla.org/zh-CN/firefox/addon/tronlink-wallet/', '_blank')
        } else {
          this.tip = 'please use chrome or firefox browser'
        }
        return;
      }
      try {
        let res = null;
        let msg =  "please create a new wallet or restore/unlock your wallet from tronlink extension. if there are popup tronlink window, please proceed the process"
        setTimeout(() => {
          if (res == null) {
            this.tip = msg
          }
        }, 5000)
        res = await window.tronLink.request({ method: 'tron_requestAccounts' })
        if (res != "") {
          msg = res.code + ', ' + res.message
        }
        console.log('info', msg)
        this.tip = msg
        this.web3 = { ...this.web3, connected: true }
        this.connectWeb3()
      } catch (error) {
        console.log('err', error)
        this.tip = error
      }
    },
    connectWallet() {
      if (this.web3.provider == 'tronlink') {
        this.connectTronLink()
      } else {
        this.tip = 'please make sure tronlink is installed'
      }
    },
    async connectWeb3() {
      try {
        if (window.tronLink) {
          if (!window.tronLink.ready) {
            this.tip = 'please connect tronlink or tronlink extension first'
            return
          }
          const address = window.tronWeb.defaultAddress.base58
          const balance = await window.tronWeb.trx.getBalance(address);
          const contract = await tronWeb.contract().at(CONTRACT);
          const contract_name = await contract.name().call();
          const contract_symbol = await contract.symbol().call();
          const mycontract_balance = await contract.methods.balanceOf(window.tronWeb.defaultAddress.base58).call();
          this.web3 = { 
            ...this.web3, 
            connected: true, 
            instance: window.tronWeb, 
            balance, 
            account: window.tronWeb.defaultAddress.base58, 
            contract, 
            contract_address: CONTRACT, 
            mycontract_balance: mycontract_balance.toString(), 
            contract_name, 
            contract_symbol, 
          }
          this.tip = 'connect success'
        } else {
          this.tip = 'please install tronlink extension first'
        }
      } catch (error) {
        console.log("err", error)
        this.tip = error + ', please try again'
      }
    },
    copyWebsiteURL() {
			const ele = document.createElement('textarea');
			ele.value = window.location.href;
			document.body.appendChild(ele);
			ele.focus();
			ele.select()
			ele.setSelectionRange(0, window.location.href.length);
			const copyRes = document.execCommand('copy');
			document.body.removeChild(ele);
			if (copyRes) {
				this.isWebsiteURLCopied = true
			}
    },
    async contract_approve() {
      if (!this.web3.contract) {
        alert('no contract found')
        return
      }

      const contract = this.web3.contract

      try {
        // transfer(to_address, amount), feeLimit: the maximum trx consumption of this calling
        // 5e9 is the max fee limit
        // docs: https://developers.tron.network/docs/trc20-contract-interaction#transfer
        const res = await contract.transfer(CONTRACT, this.amount).send({ feeLimit: 5e9 })
        console.log('transfer res', res)
      } catch (error) {
        console.log("err", error)
        this.tip = error
      }
    }
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
}

#nav {
  padding: 30px;
}

#nav a {
  font-weight: bold;
  color: #2c3e50;
}

#nav a.router-link-exact-active {
  color: #42b983;
}

button {
  margin-bottom: 1rem;
}
</style>
