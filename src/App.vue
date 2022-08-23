d<template>
  <div id="app">

  <H1>Reach Rental Agreement</H1>

    <button @click="landlord()"> Landbord</button>
    <button @click="tenant()"> Tenant</button>
    <BR/>
  
  <HR/>

  <div v-if="role==0">
  <h3>Landlord</h3>
  contract: (Copy below to Tenant)<BR/> 
  {{ctcInfoStr}}<BR/><BR/>

  addr: {{ addr}} <BR/>

  </div>

  <div v-else-if="role==1">
  <h3>Tenant</h3>
  addr: {{ addr}} <BR/>
  <input v-model="ctcStr" placeholder="paste contract here"> <button @click="attachContract1()">Attach Contract</button>
  
    <BR/>  
  </div>



  <BR/>

  <HR/>
  bal: {{ bal }} <BR/>
  balAtomic: {{ balAtomic }}<BR/>

  <button @click="updateBalance()">updateBalance</button>

  </div>
</template>

<script>
import * as backend from '../build/index.main.mjs';
import { loadStdlib } from '@reach-sh/stdlib';
//const stdlib = loadStdlib(process.env);
const stdlib = loadStdlib("ALGO");

console.log(`The consensus network is ${stdlib.connector}.`);

const suStr = stdlib.standardUnit;
console.log("Unit is ", suStr)
const toAU = (su) => stdlib.parseCurrency(su);
const toSU = (au) => stdlib.formatCurrency(au, 4);

// Defined all interact as global for backend calls, later convert them to Vue methods
let Actor = { };
let landlordInteract = { };
let tenantInteract = { };

export default {
  name: 'App',
  components: {
    //WizardSteps
  },
  data: () => {
    return {
      role: 0,
      acc: undefined,
      addr: undefined,
      balAtomic: undefined,
      bal: undefined,
      ctc: undefined,
      ctcInfoStr: undefined,
    
    };
  },
   methods: {

       waitUntil (condition) {
    return new Promise((resolve) => {
        let interval = setInterval(() => {
            if (!condition()) {
                return
            }

            clearInterval(interval)
            resolve()
        }, 100)
      })
    },

      // Create a Vue methods for every commonInteract methods
      commonFunctions() {
        Actor = {
              leave: async () => {}
          }
      },


 
    // 0 - seller, 1 - buyer, 2 - transport
    //////////////////////////// Seller methods
    async landlord() {
      this.commonFunctions();
      landlordInteract = {
        ...Actor,
        terms,
      }
      console.log("Landlord: ", landlordInteract);

      try {
          // Set role, create account, 
          this.role = 0;
          this.acc = await stdlib.newTestAccount(stdlib.parseCurrency(1000));
          //console.log("this.acc: ", this.acc)

          this.addr = stdlib.formatAddress(this.acc.getAddress());

          this.balAtomic = await stdlib.balanceOf(this.acc);
          this.bal = String(stdlib.formatCurrency(this.balAtomic, 4));
          //console.log("this.bal: ",this.bal)

          // create the contract here
          // https://docs.reach.sh/frontend/#ref-frontends-js-ctc
          this.ctc = await this.acc.contract(backend);
          console.log("this.ctc: ", this.ctc)

          // bind all the seller methods
          await this.ctc.p.Landlord(landlordInteract);

      } catch (err) {
        console.log(err);
      }
    },
    
    
      async attachContract1() {
        this.ctc = await this.acc.contract(backend, JSON.parse(this.ctcStr));     
        await this.ctc.p.Tenant(tenantInteract);

        console.log("Contract attached: ",this.ctcStr)
    },
    //////////////////////////// Tenant
    async tenant() {
      this.commonFunctions();
      tenantInteract = {
        ...Actor,
        acceptTerms: async () => {}
      }
      console.log("Tenant: ", Tenant);

      try {
        // Set role, create account
          this.role = 1;
          this.acc = await stdlib.newTestAccount(stdlib.parseCurrency(1000));
          this.addr = stdlib.formatAddress(this.acc.getAddress());
      } catch (err) {
        console.log(err);
      }
    },  
    
    async updateBalance() {
      try {
        this.balAtomic = await stdlib.balanceOf(this.acc);
        this.bal = String(stdlib.formatCurrency(this.balAtomic, 4));
      } catch (err) {
        console.log(err);
      }
    },
  },
  mounted() {
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
