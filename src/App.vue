<template>
  <div id="app">
    <H1>Vue2 Reach Rock Paper Scissors </H1>

    <H3>Choose your role:</H3>
    <button @click="Alice()">Alice</button>
    <button @click="Bob()">Bob</button>

    <BR />

    <HR />

    <div v-if="role == 1">
      <h3>Alice - Contract Creator</h3>

      Enter wager amount?
      <input v-model="wager" placeholder="" />
      <button @click="attachContract(wager)">Submit</button> <BR/>

      contract: (Copy below to Bob)<BR /><BR /> {{ ctcInfoStr }}<BR /><BR />

      <div v-if="getHandState && role==1">
          Play your hand : <BR/>
          <button @click="readHand(0)">ROCK</button>
          <button @click="readHand(1)">PAPER</button>
          <button @click="readHand(2)">SCISSORS</button> 
          <BR/>
      </div>

      <BR/>
      wager: {{ wager }} <BR/>
      bal: {{ bal }}<BR /> addr: {{ addr }} <BR />


    </div>



    <div v-else-if="role == 2">
      <h3>Bob - Contract Attacher</h3>


      wager: {{ wager }} <BR/>
      bal: {{ bal }}<BR /> addr: {{ addr }} <BR />
      <input v-model="ctcInfoStr" placeholder="paste contract here" />
      <button @click="attachContract1()">Attach Contract</button>

       <div v-if="wager>0">
        Do you want to accept wager of {{wager}} ? 
      <button @click="yesnoWager(true)">YES</button>
      <button @click="yesnoWager(false)">NO</button>
    </div>
      <BR />

      <div v-if="getHandState && role==2">
          Play your hand : <BR/>
          <button @click="readHand(0)">ROCK</button>
          <button @click="readHand(1)">PAPER</button>
          <button @click="readHand(2)">SCISSORS</button> 
          <BR/>
      </div>

    </div>

    <BR />

    <HR />
    bal: {{ bal }} <BR /> balAtomic: {{ balAtomic }}<BR />

    <button @click="updateBalance()">updateBalance</button>
  </div>
</template>

<script>
import * as backend from "../build/index.main.mjs";
import { loadStdlib } from "@reach-sh/stdlib";
//const stdlib = loadStdlib(process.env);
const stdlib = loadStdlib("ALGO");

console.log(`The consensus network is ${stdlib.connector}.`);

const suStr = stdlib.standardUnit;
console.log("Unit is ", suStr);
//const toAU = (su) => stdlib.parseCurrency(su);
//const toSU = (au) => stdlib.formatCurrency(au, 4);
const fmt = (x) => stdlib.formatCurrency(x, 4);


// Defined all interact as global for backend calls, later convert them to Vue methods
let playerInterface = {};
let aInterface = {};
let bInterface = {};

const HANDS = ["Rock", "Paper", "Scissors"]
const OUTCOME = ["Bob wins", "Draw", "Alice wins"];

export default {
  name: "App",
  components: {},
  data: () => {
    return {
      role: 0,
      acc: undefined,
      addr: undefined,
      balAtomic: undefined,
      bal: undefined,
      ctc: undefined,

      ctcInfoStr: undefined,
      timeout: undefined,
      tempHand: undefined,
      hand: undefined,
      wager: undefined,
      wagerAtomic: undefined,
      acceptWager: undefined,

      // Keep track of different states
      getHandState: false,
      outcomeState: false,
      timeoutState: false,
      paymentState: false,

    };
  },
  methods: {
    waitUntil(condition) {
      return new Promise((resolve) => {
        let interval = setInterval(() => {
          if (!condition()) {
            return;
          }
          clearInterval(interval);
          resolve();
        }, 100);
      });
    },

    async updateBalance() {
      try {
        this.balAtomic = await stdlib.balanceOf(this.acc);
        this.bal = String(stdlib.formatCurrency(this.balAtomic, 4));
      } catch (err) {
        console.log(err);
      }
    },

    // Create a Vue methods for every commonInteract methods
    commonFunctions() {
      playerInterface = {
        ...stdlib.hasRandom,
        getHand: async () => {
          console.log("getHand called from backend");

          // this will use v-if to display the input
          this.getHandState = true
          // waitUtil hand is not undefined
          await this.waitUntil(() => this.hand !== undefined );
          console.log("You played ", HANDS[this.hand]);
          const hand = stdlib.parseCurrency(this.hand);
          console.log(fmt(hand))
          return hand
        },

        seeOutcome: async (outcome) => {
          console.log("seeOutcome called from backend", outcome);
          this.outcomeState = true;
          this.seeOutcome(outcome);
        },

        informTimeout: async () => { 
          console.log("informTimeout called from backend");
          this.timeoutState = true;
          this.informTimeout();
          },

        reportPayment: (payment) => { 
          console.log("reportPayment called from backend", fmt(payment));
          this.paymentState = true;
          this.reportPayment(payment); 
          },

      };
    },

    // Mapped backend method to Vue methods

      async reportPayment(payment) {
        this.paymentMsg = 'Payment of ' + fmt(payment) + ' send to the contract';
        console.log('Payment of ' + fmt(payment) + ' send to the contract');
        
        console.log("updateBalance");
        await this.updateBalance();
      },

    async seeOutcome(outcome) {
      console.log("seeOutcome", OUTCOME[outcome]);
    },

    async informTimeout() {
      console.log(`There was a timeout.`);
      process.exit(1);
    },

    readHand(hand) {
      console.log("readHand hand: ", hand)
      this.hand = hand;
      console.log("hand: ", this.hand)
    },

    //////////////////////////// Alice methods
    async Alice() {
      this.commonFunctions();

      try {
        // Set role, create account,
        this.role = 1;
        this.acc = await stdlib.newTestAccount(stdlib.parseCurrency(1000));
        //console.log("this.acc: ", this.acc)

        this.addr = stdlib.formatAddress(this.acc.getAddress());

        this.balAtomic = await stdlib.balanceOf(this.acc);
        //this.bal = String(stdlib.formatCurrency(this.balAtomic, 4));
        this.bal = fmt(this.balAtomic);


      } catch (err) {
        console.log(err);
      }
    },

    async attachContract(wager) {

        this.wagerAtomic = await stdlib.parseCurrency(wager);

        aInterface = {
        ...playerInterface,
        wager: this.wagerAtomic,
        deadline: stdlib.parseCurrency(10),
        };
        console.log("Alice: ", aInterface);

        this.ctc = await this.acc.contract(backend);
        //console.log("this.ctc: ", this.ctc);

        // bind all the seller methods
        console.log("aInterface");
        this.ctc.p.Alice(aInterface);

        const info = await this.ctc.getInfo();
        this.ctcInfoStr = JSON.stringify(info);
        console.log("this.ctcInfoStr: ", this.ctcInfoStr);
    },

    async yesnoWager(res) {
        console.log("yesnoWager: ", res)
        this.acceptWager = res;
    },

    //////////////////////////// Bob Methods
    async Bob() {
      this.commonFunctions();
      bInterface = {
        ...playerInterface,
        acceptWager: async (wager) => {
          console.log("*** acceptWager", wager);
          this.wagerAtomic = wager;
          this.wager =  String(stdlib.formatCurrency(this.wagerAtomic, 4));
          this.waitUntil( ()=> this.acceptWager == true)
          // Exit if false
          if ( this.acceptWager == false) {
              process.exit(0);
          }
        },
      };
      console.log("Bob: ", bInterface);

      try {
        // Set role, create account
        this.role = 2;
        this.acc = await stdlib.newTestAccount(stdlib.parseCurrency(1000));
        this.addr = stdlib.formatAddress(this.acc.getAddress());
      } catch (err) {
        console.log(err);
      }
    },

    async attachContract1() {
      this.ctc = await this.acc.contract(backend, JSON.parse(this.ctcInfoStr));
      await this.ctc.p.Bob(bInterface);
      console.log("Contract attached: ", this.ctcInfoStr);
    },
  },
  computed: {

  },
  mounted() {},
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
#wizard {
  width: 1024px;
}
.content {
  width: 1024px;
}
.navigation-buttons {
  display: flex;
  justify-content: space-between;
}
</style>
