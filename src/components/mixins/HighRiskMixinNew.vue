<script>
import {
  getHNIERCBalanceOf,
  setHDeposit,
  setHNDepositETH,
  getHNWithdraw,
  getHNAllowance,
  setHApprove,
  getETHBalance,
  getVirtualPriceFromHContract,
  getExchangeRateFromHContract,
  getHNTotalAsset,
  getHNPause,
  getHNAsset,
  getHNWithdrawFee,
  setHNApprove,
  setHNDeposit
} from "@/common/web3";
import { getAsset, getProfit, fetchTxs, fetchTotalHis } from "@/common/api";
import {
  dividedBy,
  setConfirmValue,
  setWithdrawValue,
  isLessThanOrEqualTo,
  setAssetsValue,
  random,
  isLt,
  minusLet,
  minus,
  multipliedByFixed,
  calcAPY,
} from "@/utils";
import { HContract, HNContract, TTIMER, HNMarkets } from "../../config";
import { mapState } from "vuex";
const HighExchangeRate = {
  USDC: 0.98,
  ETH: 0.97,
  WBTC: 0.98,
};
// const period = 195 * 24 * 3600;
const period = 105 * 24 * 3600;
export default {
  computed: {
    ...mapState(["MetaMaskAddress", "Pendings"]),
  },
  data() {
    return {
      markets: ["eth", "wbtc"],
      list: [],
      marks: {
        0: "0",
        25: "25%",
        50: "50%",
        75: "75%",
        100: "100%",
      },
      textList: {
        USDC: "Deposited USDC will be converted into CRV and be staked. This strategy has risk exposure on CRV token, though staking APY is high, investors could bear loss or gain higher returns when CRV price fluctuates.",
        WBTC: "BTC are deposited into AAVE to earn interests, and ETH are borrowed from AAVE and invested into ETH Leveraged Strategy to earn leveraged yield.",
        ETH: "Leveraged investment in ETH 2.0 staking.",
      },
      loading: null,
      itemData: null,
      Max: 0,
      selectConfirm: "USDC",
      selectWithdraw: "USDC",
      totalOf: 0,
      withdrawInput: 0,
      withdrawVal: 0,
      confirmInput: 0,
      confirmVal: 0,
      dialogVisible: false,
      title: "",
      diaWidth: "40%",
      dialogName: "",
      echartsData: [],
      isApprove: false,
      ratio: "0%",
      codeurl: "",
      isYield: false,
      slippage: "0",
      max: false,
    };
  },
  watch: {
    Pendings(cur, old) {
      if (old.length > cur.length) {
        if (!this.itemData) return;
        this.getTotalOf(this.itemData.code);
      }
    },
    MetaMaskAddress(cur, old) {
      this.getAssetList();
    },
    withdrawInput(newValue, oldValue) {
      this.fetchExchangeRate(this.itemData.code, newValue);
    },
    async max(newOne, oldOne) {
      const totalHis = await fetchTotalHis(
        HNContract[this.itemData.code.toUpperCase()].vault,
        newOne ? Number(new Date()) : period * 1000
      );

      const { apys } = calcAPY(totalHis.totalRec, [], newOne, 195, 60);
      this.echartsData = apys;
    },
  },
  mounted() {
    this.getAssetList();
  },
  methods: {
    maximize(event) {
      this.max = event;
    },
    closeisYield() {
      this.isYield = false;
    },
    selectWithdrawChange(val) {
      this.withdrawInput = 0;
      this.withdrawVal = 0;
    },
    selectConfirmChange(val) {
      this.getHAllowances(this.itemData);
      this.getTotalOf(this.itemData.code);
      this.confirmInput = 0;
      this.confirmVal = 0;
    },

    getTimer() {
      HNMarkets.forEach((item) => {
        const timer = random(TTIMER[0], TTIMER[1]);
        this["h" + item + "timer"] = setInterval(() => {
          this.getAssets(item);
        }, timer);
      });
    },
    async getAssets(codename) {
      const data = await this.getAssetInfo(this.MetaMaskAddress, codename);
      this.list.forEach((item, idx) => {
        if (item.code === data.code) {
          if (
            item.totalassets !== data.totalassets ||
            item.user_assets !== data.user_assets
          ) {
            let showContent = false;
            if (this.itemData && this.itemData.code === data.code) {
              showContent = this.itemData.showContent
                ? this.itemData.showContent
                : false;
              this.itemData = { ...data, showContent: showContent };
            }
            this.$set(this.list, idx, {
              ...data,
              showContent: showContent,
            });
          }
        }
      });
    },

    async getAssetInfo(account, item) {
      console.log("Item: ", item)
      const decimal = HNContract[item.toUpperCase()].Decimal;
      console.log("decimal: ", decimal)
      // Get Total
      const total = await getHNTotalAsset(item.toUpperCase());
      console.log("total: ", total)

      // Get pause
      const paused = await getHNPause(item.toUpperCase());

      // Get User individual
      const userAssets = await getHNAsset(item.toUpperCase(), account);

      let userHistory = [];
      let userProfit = 0;

      if (account) {
        try {
          userHistory = await fetchTxs(
            HNContract[item.toUpperCase()].vault,
            HNContract[item.toUpperCase()].asset,
            account
          );
          let totalDeposit = 0;
          userHistory.userAssets.reverse().forEach((item) => {
            if (item.tradeType == 0) totalDeposit += Number(item.amount);
            else {
              totalDeposit -= Number(item.amount);
              if (totalDeposit < 0) totalDeposit = 0;
            }
          });

          userProfit =
            totalDeposit < userAssets / decimal
              ? userAssets / decimal - totalDeposit
              : 0;
        } catch (err) {
          console.error(err);
        }
      }

      let totalRec, avg = 0

      try {
        const { totalRec: rec } = await fetchTotalHis(
          HNContract[item.toUpperCase()].vault,
          period * 1000
        );
        totalRec = rec

        const { avg: average } = calcAPY(totalRec, [], false, 195, 60);
        avg = average
      } catch (err) {
        console.error(err)
      }
      const ratio = await getHNWithdrawFee(item.toUpperCase());
      return {
        paused,
        total: total / decimal,
        user_assets: userAssets / decimal,
        user_profit: userProfit,
        sevendayProfit: avg,
        code: item.toUpperCase(),
        ratio: ratio / 10000,
        decimal: HNContract[item.toUpperCase()].Decimal,
      };
    },

    getAssetList() {
      this.isLoading();
      Promise.all(
        HNMarkets.map((item) => {
          return this.getAssetInfo(this.MetaMaskAddress, item);
        })
      )
        .then(([...HNMarkets]) => {
          this.list = HNMarkets.map((item) => {
            return {
              ...item,
              showContent: false,
            };
          });
          if (this.$route.params.type && this.$route.params.type === "high") {
            const data = this.list.find(
              (item) => item.code === this.$route.params.code
            );
            this.changeContent(data);
          }
          HNMarkets.forEach((item) => {
            clearInterval(this["h" + item + "timer"]);
            this["h" + item + "timer"] = null;
          });
          this.getTimer();
          this.downLoading();
        })
        .catch((err) => {
          this.downLoading();
        });
    },
    getVirtualPrice() {
      Promise.all(
        HNMarkets.map((item) => {
          return getVirtualPriceFromHContract(item);
        })
      ).then((lists) => {
        this.virtualPriceList["USDC"] = new BigNumber(1e18)
          .dividedBy(new BigNumber(lists[0]))
          .toNumber();
        this.virtualPriceList["ETH"] = new BigNumber(1e18)
          .dividedBy(new BigNumber(lists[1]))
          .toNumber();
      });
    },
    async approve() {
      if (!this.MetaMaskAddress) return this.Warning("Please link wallet");
      this.isLoading();
      try {
        const resApprove = await setHNApprove(
          new BigNumber(1e32).toString(10),
          this.MetaMaskAddress,
          this.itemData.code,
        );
        if (resApprove.status) {
          this.isApprove = false;
          this.Success("Successfully authorized.");
          this.downLoading();
        }
      } catch (error) {
        this.downLoading();
      }
    },
    async confirm(item) {
      if (!this.MetaMaskAddress) return this.Warning("Please link wallet");
      if (Number(this.confirmInput) === 0) return this.Warning("Invalid Value");
      const less = isLessThanOrEqualTo(this.confirmInput, this.totalOf);
      if (!less) return this.Warning("Invalid Value");
      if (item.code === "ETH") {
        const minNum = isLt(this.confirmInput, 0.01);
        if (minNum) return this.Warning("Minimum deposit amount 0.01 ETH.");
        const maxNum = minusLet(this.confirmInput, this.totalOf);
        if (!maxNum)
          return this.Warning(
            "The amount of asset you want to deposit exceeds the safe max value, which may lead to failure to pay gas fee.",
            "Invalid value"
          );
      }
      const decimal = HNContract[item.code];
      let bigInput = 0;
      let params = null;
      if (item.code === "ETH") {
        bigInput = setConfirmValue(this.confirmInput, decimal.Decimal);
        params = await setHNDepositETH(
          bigInput,
          this.MetaMaskAddress,
          item.code,
          "high"
        );
      } else {
        console.log("Big: ", this.confirmInput, item.code)
        bigInput = setConfirmValue(
          this.confirmInput,
          decimal.Decimal
        );
        params = await setHNDeposit(
          bigInput,
          this.MetaMaskAddress,
          item.code,
        );
        console.log("Params: ", params)
      }
      this.sendTransaction(params);
    },
    async fetchExchangeRate(code, value) {
      if (value < 1e-6) {
        this.slippage = "0";
        return;
      }
      const exchangeRate = await getExchangeRateFromHContract(code, value);
      const standardExchangeRate = await getExchangeRateFromHContract(code, 1);
      this.slippage = Number(
        ((standardExchangeRate - exchangeRate) * 100).toFixed(2)
      );
    },
    async withdrawItem(item) {
      if (!this.MetaMaskAddress) return this.Warning("Please link wallet");
      if (Number(this.withdrawInput) === 0)
        return this.Warning("Invalid Value");
      const user =
        this.selectWithdraw === "USDC" ? "user_assets" : "user_assets_origin";
      const maxNum = item.code === "ETH" ? item.user_assets : item[user];
      const less = isLessThanOrEqualTo(this.withdrawInput, maxNum);
      if (!less) return this.Warning("Invalid Value");
      if (item.code !== "ETH" && isLt(item.user_profit, 0)) {
        return (this.isYield = true);
      }
      // const exchangeRate = await getExchangeRateFromHContract(
      //   item.code,
      //   this.withdrawInput
      // );
      this.withdraw(item);
      // if (exchangeRate > HighExchangeRate[item.code]) {
      //   this.withdraw(item);
      // } else {
      //   this.$confirm(
      //     "The market is fluctuating now. Withdraw may experience big slippage results in a big loss of principle. Are you sure to continue?",
      //     "Warning",
      //     {
      //       confirmButtonText: "Continue",
      //       cancelButtonText: "Cancel",
      //       type: "warning",
      //     }
      //   )
      //     .then(() => {
      //       this.$message({
      //         type: "success",
      //         message: "Action completed",
      //       });
      //       this.withdraw(item);
      //     })
      //     .catch(() => {
      //       this.$message({
      //         type: "info",
      //         message: "Action canceled",
      //       });
      //     });
      // }
    },
    async withdraw(item) {
      this.isYield = false;
      const user =
        this.selectWithdraw === "USDC" ? "user_assets" : "user_assets_origin";
      const maxNum = item.code === "ETH" ? item.user_assets : item[user];
      const maxWithdraw = this.withdrawInput === maxNum;
      console.warn("Withdraw: ", user, maxNum, maxWithdraw)
      // const maxWithdraw = this.withdrawInput === item.user_assets;
      const amount =
        maxWithdraw === true
          ? new BigNumber(item.user_assets)
            .multipliedBy(new BigNumber(item.decimal))
            .toFixed(0)
          : new BigNumber(this.withdrawInput)
            .multipliedBy(new BigNumber(item.decimal))
            .toFixed(0);


      // const bigInput = maxWithdraw
      //   ? item.lp_token
      //   : setWithdrawValue(this.withdrawInput, item.lp_token, maxNum);
      console.log("Amount: ", amount)
      const params = await getHNWithdraw(
        amount,
        this.MetaMaskAddress,
        item.code,
        this.selectWithdraw
      );
      console.log("Params: ", params)
      this.sendTransaction(params);
    },
    inputConfirm() {
      this.confirmVal = 0;
    },
    setConfirmVal(item) {
      const num =
        this.itemData.code === "ETH" ? minus(this.totalOf) : this.totalOf;
      this.confirmInput = setAssetsValue(item, num);
    },
    setWithdrawVal(item) {
      //TODO: for wbtc
      if (this.itemData.code === "ETH" || this.itemData.code === "WBTC") {
        const decimal = HNContract[this.itemData.code].assetDecimal;
        const deRatio = this.itemData.user_assets * Math.floor(0.99999 * 1e8) / 1e8;
        this.withdrawInput =
          item === 100
            ? deRatio.toFixed(
              decimal)
            : Number(setAssetsValue(item, this.itemData.user_assets)).toFixed(
              decimal
            );
      } else {
        const user =
          this.selectWithdraw === "USDC" ? "user_assets" : "user_assets_origin";
        this.withdrawInput =
          item === 100
            ? this.itemData[user]
            : setAssetsValue(item, this.itemData[user]);
        this.inputWithdraw("set");
      }
    },
    inputWithdraw(type = null) {
      // if (type !== "set") {
      //   this.withdrawVal = 0;
      // }
      // if (!this.withdrawInput) {
      //   this.ratio = Number(this.itemData.ratio) * 100 + "%";
      //   return;
      // }
      // if (Number(this.withdrawInput) === 0) {
      //   this.ratio = Number(this.itemData.ratio) * 100 + "%";
      //   return;
      // }
      // this.ratio = multipliedByFixed(
      //   this.withdrawInput,
      //   this.itemData.ratio,
      //   HContract[this.itemData.code].Length
      // );
    },
    inputWithdrawNew(val, max) {
      this.withdrawInput = Number(val);
      const decimal = HNContract[this.itemData.code].assetDecimal;
      const deRatio = max * Math.floor(0.99999 * 1e8) / 1e8;
      const maxNum = deRatio.toFixed(decimal);
      this.$nextTick(()=>{
        if(Number(val) > maxNum){
          this.withdrawInput = maxNum;
          this.withdrawVal = 100;
        }
      })
    },
    setMax(type, val) {
      if (type === 1) {
        this.confirmInput = this.itemData.code === "ETH" ? minus(val) : val;

        this.confirmVal = 100;
      } else {
        const decimal = HNContract[this.itemData.code].assetDecimal;
        const deRatio = val.user_assets * Math.floor(0.99999 * 1e8) / 1e8;

        const user =
          this.selectWithdraw === "USDC" ? "user_assets" : "user_assets_origin";
        // this.withdrawInput =
        //   this.itemData.code === "ETH" ? deRatio.toFixed(decimal): val[user];
        //   console.warn("this.withdrawInput: ",this.withdrawInput,deRatio);


        if (this.itemData.code === "ETH") {
          this.withdrawInput = deRatio.toFixed(decimal);
        } else if (this.itemData.code === "WBTC") {
          this.withdrawInput = deRatio.toFixed(decimal);//TODO
        } else
          this.withdrawInput = val[user];




        this.inputWithdraw();
        this.withdrawVal = 100;

      }
    },
    isLoading() {
      this.loading = this.$loading({
        lock: true,
        text: "Loading",
        spinner: "el-icon-loading",
        background: "rgba(0, 0, 0, 0.7)",
      });
    },
    downLoading() {
      if (this.loading) {
        this.loading.close();
      }
      this.loading = null;
    },
    async changeContent(val) {
      this.isLoading();
      // this.withdrawInput = "Please slide to adjust the amount";
      this.withdrawInput = 0;
      this.withdrawVal = 0;
      this.confirmInput = 0;
      this.confirmVal = 0;
      this.isApprove = true;
      this.selectWithdraw = "USDC";
      if (val.code !== "ETH" && !val.showContent && this.MetaMaskAddress) {
        this.getHAllowances(val);
      }
      this.list.forEach((item) => {
        if (val.code === item.code) {
          item.showContent = !item.showContent;
          if (item.showContent) {
            this.itemData = { ...val, showContent: item.showContent };
            this.ratio = val.ratio ? Number(val.ratio) * 100 + "%" : 0;
            this.downLoading();
            this.getTotalOf(val.code);
          } else {
            this.downLoading();
            this.itemData = null;
          }
        } else {
          item.showContent = false;
        }
      });
    },
    async getHAllowances(val) {
      console.log("Code: ", val.code)
      const allowance = await getHNAllowance(
        this.MetaMaskAddress,
        val.code === "USDC" ? this.selectConfirm : val.code,
      );
      console.log("Allowance: ", allowance)
      const item = val.code === "USDC" ? this.selectConfirm : ""
      const number =
        val.code === "ETH"
          ? HContract[val.code].Decimal
          : HNContract[val.code][`${item}Decimal`];
      console.log("number: ", number)

      const myAllowance = dividedBy(
        allowance,
        number
      );
      const less = isLessThanOrEqualTo(myAllowance, 0);
      console.log("Less: ", less, myAllowance)
      this.isApprove = less;
    },
    async getTotalOf(code) {
      if (!this.MetaMaskAddress) {
        this.totalOf = 0;
        return this.downLoading();
      }
      const item = code === "USDC" ? this.selectConfirm : ""
      console.log("Total : ", item, code)
      const bcf =
        code === "ETH"
          ? await getETHBalance(this.MetaMaskAddress)
          : await getHNIERCBalanceOf(
            this.MetaMaskAddress,
            code
          );
      console.log("Amt: ", bcf)
      const number =
        code === "ETH"
          ? HContract[code].Decimal
          : HNContract[code][`${item}Decimal`];
      console.log("number: ", number)
      this.totalOf = dividedBy(bcf, number);
      this.downLoading();
    },
    closeMain(val) {
      this.title = "";
      this.dialogVisible = val;
    },

    async selectLine() {
      this.codeurl = this.itemData.code.toLowerCase();
      // const list = await getProfit(this.codeurl);
      const totalHis = await fetchTotalHis(
        HNContract[this.itemData.code.toUpperCase()].vault,
        period * 1000
      );

      const { apys } = calcAPY(totalHis.totalRec, [], false, 195, 60);

      this.echartsData = apys;
      this.title = this.itemData.code === "USDC" ? "CRV 30 Days APY" : "60 Days APY";
      this.dialogName = "EchartsLine";
      this.diaWidth = "80%";
      this.dialogVisible = true;
    },
    async selectTable() {
      this.codeurl = "hn" + this.itemData.code.toLowerCase();
      this.title = "History";
      this.dialogName = "CffTableNew";
      this.diaWidth = "80%";
      this.dialogVisible = true;
    },
  },
  destroyed() {
    HNMarkets.forEach((item) => {
      clearInterval(this["h" + item + "timer"]);
      this[item + "timer"] = null;
    });
  },
};
</script>
