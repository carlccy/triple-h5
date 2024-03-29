<template>
  <div class="page liquidity">
    <div class="main">
      <h6>Your liquidity</h6>
      <div class="top">
        <div class="fs12 warn">
          Pool providers earn a 0.3% fee on all trades proportional to
          their share of the pool. Fees are added to the pool, accrue in real
          time and can be claimed by withdrawing your liquidity.
        </div>
        <el-button type="primary" round @click="handleAdd"
          >Add Liquidity</el-button
        >
      </div>

      <div class="list" v-if="list.length">
        <el-collapse v-model="activeName" accordion @change="handleChange">
          <el-collapse-item
            v-for="(item, index) in list"
            :key="index"
            :name="index"
          >
            <template slot="title">
              <div class="flex list-head">
                <div class="icons">
                  <div class="icon icon-1">
                    <img :src="require(`@/assets/coin/coin_${item.token.toUpperCase()}.png`)" alt="" />
                  </div>
                  <div class="icon icon-2">
                    <img :src="require(`@/assets/coin/coin_${item.currency.toUpperCase()}.png`)" alt="" />
                  </div>
                </div>
                <div class="fs16b title">
                  {{ item.token.toUpperCase() }}/{{ item.currency.toUpperCase() }}
                </div>
              </div>
            </template>
            <div class="list-body" v-loading="loading">
              <div class="coin-item">
                <div class="fs14 name">LP Token</div>
                <div class="fs14b value">{{ formatDecimals(balanceOf) | formatMoney }}</div>
              </div>
              <div class="coin-item">
                <div class="fs14 name">Valued USDT</div>
                <div class="fs14b value">{{ formatDecimals(poolNet * balanceOf / totalSupply) | formatMoney }}</div>
              </div>
              <div class="coin-item">
                <div class="fs14 name">Your pool share</div>
                <div class="fs14b value">
                  {{ poolShare }}%
                </div>
              </div>
              <div class="btns flex flex-bt">
                <el-button class="btn" type="primary" round @click="add(item)"
                  >Add</el-button
                >
                <el-button class="btn btn-remove" round @click="remove(item)"
                  >Remove</el-button
                >
              </div>
            </div>
          </el-collapse-item>
        </el-collapse>
      </div>

      <div class="no-liquidity" v-else>
        <svg-icon icon-class="ic_Nothing" class-name="icon-noliq"></svg-icon>
        <div class="fs12">No liquidity found.</div>
        <div class="flex import">
          <span class="fs12">Don't see a pool you joined?</span>
          <a class="fs12" href="#">Import it.</a>
        </div>
      </div>

      <el-dialog
        title="Remove Pool"
        :visible.sync="removeModel"
        :close-on-click-modal="false"
        width="375px"
      >
        <div class="content-remove">
          <div class="flex flex-ac flex-bt">
            <span class="fs14">LP Token</span>
            <span class="fs14 value">{{ formatDecimals(balanceOf) | formatMoney }}</span>
          </div>
          <div style="height: 18px"></div>
          <div class="flex flex-ac flex-bt">
            <span class="fs14">Available LP</span>
            <span class="fs14 value">{{ formatDecimals(maxLiquidity) | formatMoney }}</span>
          </div>
          <div style="height: 29px"></div>
          <el-input-number
            style="width: 100%"
            v-model="liquidity"
            :precision="precision"
            :step="step"
            :min="0"
            :max="maxLiquidity * 1"
          ></el-input-number>
          <div style="height: 8px"></div>
          <el-slider
            v-model="precent"
            :min="0"
            :max="100"
            :format-tooltip="formatPrecent"
            @input="precentChange"
          ></el-slider>
          <div style="height: 32px"></div>
          <el-button
            class="btn-remove"
            type="danger"
            round
            @click="handleRemoveLiquidity"
            :disabled="!liquidity"
            >Remove</el-button
          >
        </div>
      </el-dialog>
    </div>
  </div>
</template>

<script>
import { bus } from "@/utils/bus";
import { formatNum } from "@/utils/util";
import abi from "@/contracts/HedgexSingle.json";
import erc20abi from "@/contracts/TokenERC20.json"; // 标准ERC20代币ABI
import { getTradePairs } from "@/api";
export default {
  name: "Pool",
  data() {
    return {
      activeName: "",
      list: [
        // {
        //   token: "eth",
        //   currency: "usdt"
        // }
      ],
      curPair: {},

      removeModel: false,
      removeLoading: false,
      liquidity: 0,
      maxLiquidity: 0,
      precent: 0,

      pcontract: null,
      ptoken0: null,

      // 1: balanceOf用来获取用户lp量，totalSupply用来获取lp总量，totalPool用来获取对冲池总价值。
      // 对于流动性提供者，用这几个量进行计算就行。
      loading: false,
      name: "",
      symbol: "",
      balanceOf: 0,
      totalSupply: 0,
      totalPool: 0,
      poolNet: 0,
      token0BalanceOf: 0,
      token0Decimals: 0
    };
  },
  computed: {
    precision() {
      return Math.abs(this.token0Decimals) || 2;
    },
    step() {
      return 1 / Math.pow(10, this.token0Decimals) || 0.01;
    },
    poolShare() {
      if (this.totalSupply) {
        const share = (this.balanceOf / this.totalSupply) * 100;
        if (share > 0.01) {
          return formatNum(share, 2);
        } else {
          return "<0.01";
        }
      } else {
        return "0";
      }
    }
  },
  watch: {
    removeModel(n) {
      if (!n) {
        this.liquidity = "";
        this.precent = 0;
      }
    },
    curPair(n) {
      if (n) {
        this.getContractInfo();
      }
    }
    // 交易对改变，合约改变
  },
  created() {
    this.getPairsInfo();
    bus.$on("accountsChanged", () => {
      this.getPairsInfo();
    });
  },
  beforeDestroy() {
    bus.$off("accountsChanged");
  },
  methods: {
    formatDecimals(val) {
      // 合约换算和精度
      return formatNum(
        val / Math.pow(10, this.token0Decimals),
        this.token0Decimals * 1
      );
    },
    async getPairsInfo() {
      const res = await getTradePairs();
      if (res.result) {
        this.list = res.data.map(item => {
          return {
            token: item.trade_coin,
            currency: item.margin_coin,
            contractAddress: item.contract,
            token0Address: item.margin_address
          };
        });
      }
    },
    handleChange(i) {
      if (typeof i == "number") {
        this.curPair = this.list[i];
      }
    },
    async getContractInfo() {
      if (!this.coinbase) {
        this.$message({
          type: "error",
          message: "Please connect wallet first."
        });
        return;
      }
      try {
        this.loading = true;
        this.pcontract = new this.web3.eth.Contract(
          abi,
          this.curPair.contractAddress
        );
        this.ptoken0 = new this.web3.eth.Contract(
          erc20abi,
          this.curPair.token0Address
        ); // 生成token0合约对象
        // this.name = await this.pcontract.methods.name().call();
        // this.symbol = await this.pcontract.methods.symbol().call();
        const [
          balanceOf,
          totalSupply,
          totalPool,
          token0BalanceOf,
          token0Decimals
        ] = await Promise.all([
          this.pcontract.methods.balanceOf(this.coinbase).call(),
          this.pcontract.methods.totalSupply().call(), // lp总量
          this.pcontract.methods.totalPool().call(), // 对冲池总价值
          this.ptoken0.methods.balanceOf(this.coinbase).call(),
          this.ptoken0.methods.decimals().call()
        ]);
        this.balanceOf = balanceOf * 1;
        this.totalSupply = totalSupply * 1;
        this.totalPool = totalPool * 1;
        this.token0BalanceOf = token0BalanceOf * 1;
        this.token0Decimals = token0Decimals * 1;
        try {
          const poolNet = await this.pcontract.methods.getPoolNet().call(); // 对冲池总价值
          this.poolNet = poolNet * 1;
        } catch (error) {
          this.poolNet = 0;
        }
        this.loading = false;
      } catch (error) {
        this.loading = false;
        console.log(error);
      }
    },
    handleAdd() {
      this.$router.push("/pool/add/usdt");
    },
    add(item) {
      this.$router.push(`/pool/add/${item.currency}/${item.token}`);
    },
    remove(item) {
      this.token = item.token.toUpperCase();
      this.currency = item.currency.toUpperCase();
      this.removeModel = true;
      this.getRemoveMax();
    },
    // remove
    formatPrecent(val) {
      return val + "%";
    },
    precentChange() {
      this.liquidity = this.formatDecimals(
        (this.maxLiquidity * this.precent) / 100
      );
    },
    async handleRemoveLiquidity() {
      this.removeLoading = true;
      const params = {
        liquidity: this.toBN(
          this.liquidity * Math.pow(10, this.token0Decimals)
        ),
        to: this.coinbase
      };
      console.log(params);
      try {
        const res = await this.pcontract.methods
          .removeLiquidity(params.liquidity, params.to)
          .send({ from: this.coinbase });
        console.log("removeLiquidity", res);
        this.removeModel = false;
        this.getPairsInfo();
        this.getContractInfo();
      } catch (error) {
        console.log(error);
      }
      this.removeLoading = false;
    },
    async getRemoveMax() {
      let maxLiquidity;
      const [
        totalPool,
        totalSupply,
        indexPrice,
        poolLongPrice,
        poolLongAmount,
        poolShortPrice,
        poolShortAmount,
        leverage,
        balanceOf
      ] = await Promise.all([
        this.pcontract.methods.totalPool().call(),
        this.pcontract.methods.totalSupply().call(),
        this.pcontract.methods.getLatestPrice().call(),
        this.pcontract.methods.poolLongPrice().call(),
        this.pcontract.methods.poolLongAmount().call(),
        this.pcontract.methods.poolShortPrice().call(),
        this.pcontract.methods.poolShortAmount().call(),
        this.pcontract.methods.leverage().call(),
        this.pcontract.methods.balanceOf(this.coinbase).call()
      ]);
      const net =
        totalPool * 1 +
        (poolLongAmount * indexPrice + poolShortAmount * poolShortPrice) -
        (poolLongAmount * poolLongPrice + poolShortAmount * indexPrice);
      if (net < 0) {
        maxLiquidity = 0;
      } else {
        const usedMargin =
          Math.abs(poolLongAmount - poolShortAmount) * indexPrice;
        let canWithdraw = net - usedMargin;
        if (canWithdraw > net / leverage) {
          canWithdraw = net / leverage;
        }
        if (canWithdraw > 0) {
          maxLiquidity = (canWithdraw * totalSupply) / net;
          if (balanceOf < maxLiquidity) {
            maxLiquidity = balanceOf * 1;
            // maxAmount = net * maxLiquidity / totalSupply
          }
        } else {
          maxLiquidity = 0;
        }
      }
      this.maxLiquidity = maxLiquidity;
    }
  }
};
</script>

<style lang="scss" scoped>
@import "./index.scss";
</style>