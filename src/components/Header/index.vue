<template>
  <div class="header">
    <router-link to="/" class="logo">
      <svg-icon icon-class="Logo" class-name="logo-1"></svg-icon>
      <svg-icon icon-class="Logo_2" class-name="logo-2"></svg-icon>
    </router-link>
    <router-link class="tab btn14" to="/trade">Trade</router-link>
    <router-link class="tab btn14" to="/pool">Pool</router-link>
    <!-- <router-link class="tab btn14" to="/about">Components</router-link> -->
    <div v-if="!coinbase" class="wallet btn14" @click="model = true">
      Connect Wallet
    </div>
    <div v-else class="wallet2 fs14" @click="model = true">
      <span>{{ balance | formatBalance }} ETH</span>
      <div class="address">
        <span>{{ coinbase | formatCoinbase }}</span>
        <svg-icon
          v-if="wallet === 'MetaMask'"
          icon-class="ic_metamask"
          class-name="icon-wallet"
        ></svg-icon>
        <svg-icon
          v-if="wallet === 'WalletConnect'"
          icon-class="ic_wallet"
          class-name="icon-wallet"
        ></svg-icon>
      </div>
    </div>
    <!-- <svg-icon
      icon-class="ic_Add"
      class-name="s24 fullscreen"
      @click.native="getBalance()"
    ></svg-icon> -->
    <!-- <svg-icon
      icon-class="ic_Add"
      class-name="s24 fullscreen"
      @click.native="sendCoin()"
    ></svg-icon> -->
    <div class="more">
      <svg-icon icon-class="ic_more" class-name="s24 icon-more"></svg-icon>
      <div class="more-wrap">
        <div class="more-list">
          <a :href="v" target="_blank" class="more-item" v-for="(v, k) in links" :key="k">
            <span class="fs14">{{k}}</span>
            <svg-icon :icon-class="`ic_${k}`" class-name="fs16"></svg-icon>
          </a>
        </div>
      </div>
    </div>
    <svg-icon
      v-if="!isFull"
      icon-class="ic_Full1"
      class-name="s24 fullscreen"
      @click.native="screenFull"
    ></svg-icon>
    <svg-icon
      v-else
      icon-class="ic_Full2"
      class-name="s24 fullscreen"
      @click.native="exitFull"
    ></svg-icon>

    <el-dialog
      title="Select a Wallet"
      :visible.sync="model"
      :show-header="false"
      width="375px"
    >
      <div class="wallets" @click.stop="model = false">
        <!-- v-if="isMetaMask" -->
        <div class="wallet-item" @click="metaMaskInit">
          <svg-icon icon-class="ic_metamask" class-name="icon"></svg-icon>
          <h6>MetaMask</h6>
        </div>
        <!-- <a
          v-else
          class="wallet-item"
          target="_blank"
          href="https://metamask.io/"
          rel="noopener noreferrer"
        >
          <svg-icon icon-class="ic_metamask" class-name="icon"></svg-icon>
          <h6>MetaMask</h6>
        </a> -->
        <div class="wallet-item" @click="walletConnectInit">
          <svg-icon icon-class="ic_wallet" class-name="icon"></svg-icon>
          <h6>Wallet</h6>
          <svg-icon
            v-if="wallet === 'WalletConnect'"
            icon-class="ic_close"
            class-name="s24 close"
            @click.native.stop="disconnect()"
          ></svg-icon>
        </div>
        <a
          class="wallet-item"
          target="_blank"
          href="https://trustwallet.com/"
          rel="noopener noreferrer"
        >
          <svg-icon icon-class="ic_trust" class-name="icon"></svg-icon>
          <h6>Trust</h6>
        </a>
        <a
          class="wallet-item"
          target="_blank"
          href="https://mathwallet.org/"
          rel="noopener noreferrer"
        >
          <svg-icon icon-class="ic_math" class-name="icon"></svg-icon>
          <h6>Math</h6>
        </a>
      </div>
    </el-dialog>
  </div>
</template>

<script>
import { mapState, mapActions } from "vuex";
import { screenFull, exitFull, checkFull } from "@/utils/util";
export default {
  name: "Header",
  data() {
    return {
      model: false,
      isFull: false,

      links: {
        FAQ: "docs.triple.fi",
        Twitter: "https://twitter.com/triplefinance",
        Medium: "https://triplefi.medium.com/",
        Telegram: "https://t.me/triplefi",
        Discord: "https://discord.gg/pyMNKCjf5r"
      }
    };
  },
  computed: {
    ...mapState(["coinbase", "balance", "wallet", "isMetaMask"])
  },
  filters: {
    formatCoinbase(val) {
      return val ? val.slice(0, 6) + "..." + val.slice(-4) : "";
    },
    formatBalance(val) {
      // return Number(val).toFixed(4);
      return val;
    }
  },
  mounted() {
    checkFull(this);
  },
  methods: {
    ...mapActions(["metaMaskInit", "walletConnectInit", "disconnect"]),
    screenFull() {
      screenFull();
    },
    exitFull() {
      exitFull();
    }
  }
};
</script>

<style lang="scss" scoped>
@import "./index.scss";
</style>
