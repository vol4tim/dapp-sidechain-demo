<template>
  <div>
    <h1>Market</h1>
    <div>
      Lighthouse:
      <a
        :href="`https://etherscan.io/address/${lighthouse.address}`"
        target="blank"
      >{{ lighthouse.name }}</a>
    </div>

    <div>
      Cost: {{ price | fromWei(9, 'XRT') }} |
      Balance:
      <a
        :href="`https://etherscan.io/token/${token}?a=${account}`"
        target="blank"
      >{{ balance | fromWei(9, 'XRT') }}</a>
      <span v-if="price > 0">| Approved: {{ approveTrade | fromWei(9, 'XRT') }}</span>
    </div>

    <button
      v-if="price > 0 && approveTrade < price"
      :disabled="loadingApprove || balance < price"
      @click="approve"
    >Approve</button>
    <button v-if="approveTrade >= price" @click="demand">demand</button>
    <button v-if="approveTrade >= price" @click="offer">offer</button>

    <div v-if="liability.length>0">
      <h3>Liability</h3>
      <div class="block" v-for="(item, i) in liability" :key="i">
        <b>liability:</b>
        <a :href="`https://etherscan.io/address/${item.address}`" target="_blank">{{ item.address }}</a>
        <br>
        <b>lighthouse:</b>
        <a
          :href="`https://etherscan.io/address/${item.lighthouse}`"
          target="_blank"
        >{{ item.lighthouse }}</a>
        <br>
        <b>validator:</b>
        <a
          :href="`https://etherscan.io/address/${item.validator}`"
          target="_blank"
        >{{ item.validator }}</a>
        <br>
        <b>model:</b>
        {{ item.model }}
        <br>
        <b>objective:</b>
        {{ item.objective }}
        <br>
        <b>token:</b>
        {{ item.token }}
        <br>
        <b>cost:</b>
        {{ item.cost }}
        <br>
        <b>promisee:</b>
        {{ item.promisee }}
        <br>
        <b>promisor:</b>
        {{ item.promisor }}
        <br>
        <div v-if="item.result != ''">
          <b>Results:</b>
          <a :href="`https://ipfs.io/ipfs/${item.result}`" target="_blank">{{ item.result }}</a>
          <span v-if="item.check === true">+</span>
          <span v-else>-</span>
        </div>
        <div v-if="item.result == ''">
          <b>Results:</b>
          <button @click="result(item.address)">send</button>
        </div>
      </div>
    </div>

    <div v-if="demands.length>0">
      <h3>Demands</h3>
      <div class="block" v-for="(item, i) in demands" :key="i">
        <b>sender:</b>
        {{ item.sender }}
        <br>
        <b>validator:</b>
        {{ item.validator }}
        <br>
        <b>model:</b>
        {{ item.model }}
        <br>
        <b>objective:</b>
        {{ item.objective }}
        <br>
        <b>token:</b>
        {{ item.token }}
        <br>
        <b>cost:</b>
        {{ item.cost }}
        <br>
        <b>deadline:</b>
        {{ item.deadline }}
      </div>
    </div>

    <div v-if="offers.length>0">
      <h3>Offers</h3>
      <div class="block" v-for="(item, i) in offers" :key="i">
        <b>sender:</b>
        {{ item.sender }}
        <br>
        <b>validator:</b>
        {{ item.validator }}
        <br>
        <b>model:</b>
        {{ item.model }}
        <br>
        <b>objective:</b>
        {{ item.objective }}
        <br>
        <b>token:</b>
        {{ item.token }}
        <br>
        <b>cost:</b>
        {{ item.cost }}
        <br>
        <b>deadline:</b>
        {{ item.deadline }}
      </div>
    </div>
  </div>
</template>

<script>
import Vue from "vue";
import * as config from "../config";

export default {
  data() {
    return {
      lighthouse: {
        name: "",
        address: ""
      },
      account: "",
      price: config.PRICE,
      token: "",
      balance: 0,
      approveTrade: 0,
      loadingApprove: false,
      liability: [],
      demands: [],
      offers: [],
      nonce: 0
    };
  },
  mounted() {
    this.lighthouse.name = this.$robonomics.lighthouse.name;
    this.lighthouse.address = this.$robonomics.lighthouse.address;
    this.account = this.$robonomics.account.address;
    this.token = this.$robonomics.xrt.address;
    this.$robonomics.factory.call
      .nonceOf(this.$robonomics.account.address)
      .then(r => {
        this.nonce = Number(r);
      });
    this.$robonomics.onDemand(config.MODEL, msg => {
      const item = this.demands.find(item => item.signature === msg.signature);
      if (!item) {
        this.demands = [{ ...msg }, ...this.demands.slice(0, 10)];
      }
    });
    this.$robonomics.onOffer(config.MODEL, msg => {
      const item = this.offers.find(item => item.signature === msg.signature);
      if (!item) {
        this.offers = [{ ...msg }, ...this.offers.slice(0, 10)];
      }
    });
    this.$robonomics.onResult(msg => {
      this.setResult(msg.liability, msg.result, false);
    });
    this.$robonomics.onLiability((err, liability) => {
      const item = this.liability.find(
        item => item.address === liability.address
      );
      if (!item) {
        liability.getInfo().then(info => {
          this.liability = [
            {
              address: liability.address,
              worker: liability.worker,
              ...info
            },
            ...this.liability
          ];
        });
        liability.onResult().then(result => {
          this.setResult(liability.address, result, true);
        });
      }
    });
    this.fetchBalance();
  },
  methods: {
    fetchBalance() {
      return this.$robonomics.xrt.call
        .balanceOf(this.$robonomics.account.address)
        .then(balanceOf => {
          this.balance = balanceOf;
          if (balanceOf > 0) {
            return this.$robonomics.xrt.call.allowance(
              this.$robonomics.account.address,
              this.$robonomics.factory.address
            );
          }
          return false;
        })
        .then(allowance => {
          if (allowance) {
            this.approveTrade = allowance;
          }
        });
    },
    approve() {
      this.loadingApprove = true;
      return this.$robonomics.xrt.send
        .approve(this.$robonomics.factory.address, this.price, {
          from: this.$robonomics.account.address
        })
        .then(() => {
          this.loadingApprove = false;
          return this.fetchBalance();
        })
        .catch(() => {
          this.loadingApprove = false;
        });
    },
    setResult(address, result, check = true) {
      const i = this.liability.findIndex(item => item.address === address);
      if (i >= 0) {
        Vue.set(this.liability, i, { ...this.liability[i], result, check });
      }
    },
    demand() {
      this.$robonomics.web3.eth.getBlock("latest", (e, r) => {
        const demand = {
          model: config.MODEL,
          objective: config.OBJECTIVE,
          token: this.$robonomics.xrt.address,
          cost: this.price,
          lighthouse: this.$robonomics.lighthouse.address,
          validator: config.VALIDATOR,
          validatorFee: 0,
          deadline: r.number + 1000,
          nonce: this.nonce
        };
        this.nonce++;
        this.$robonomics.sendDemand(demand);
      });
    },
    offer() {
      this.$robonomics.web3.eth.getBlock("latest", (e, r) => {
        const offer = {
          model: config.MODEL,
          objective: config.OBJECTIVE,
          token: this.$robonomics.xrt.address,
          cost: this.price,
          lighthouse: this.$robonomics.lighthouse.address,
          validator: config.VALIDATOR,
          lighthouseFee: 0,
          deadline: r.number + 1000,
          nonce: this.nonce
        };
        this.nonce++;
        this.$robonomics.sendOffer(offer);
      });
    },
    result(address) {
      this.$robonomics.sendResult({
        liability: address,
        success: true,
        result: config.RESULT
      });
    }
  }
};
</script>

<style>
.block {
  border: 1px solid #eee;
  padding: 10px;
  margin: 10px 0;
}
</style>
