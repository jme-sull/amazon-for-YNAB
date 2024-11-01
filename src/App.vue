<template>
  <div id="app">
    <Nav />
    <div class="container">
      <!-- Display a loading message if loading -->
      <h1 v-if="loading" class="display-4">Loading...</h1>

      <!-- Display an error if we got one -->
      <div v-if="error">
        <h1 class="display-4">Oops!</h1>
        <p class="lead">{{ error }}</p>
        <button class="btn btn-primary" @click="resetToken">
          Try Again &gt;
        </button>
      </div>

      <!-- Otherwise show our app contents -->
      <div v-else>
        <!-- If we dont have a token ask the user to authorize with YNAB -->
        <form v-if="!ynab.token">
          <div class="form-group">
            <h2>Welcome!</h2>
            <p class="lead">
              Match your detailed Amazon order history to your unapproved YNAB
              transactions for easy categorizing
            </p>
            <p class="lead">To get started, please authorize with YNAB!</p>
            <button @click="authorizeWithYNAB" class="btn btn-primary">
              Authorize This App With YNAB &gt;
            </button>
          </div>
        </form>

        <!-- Otherwise if we have a token, show the budget select -->
        <Budgets
          v-else-if="!budgetId"
          :budgets="budgets"
          :selectBudget="selectBudget"
        />

        <Uploader
          v-else-if="!csvData.length"
          :handleFileUpload="handleFileUpload"
        />

        <!-- If a budget has been selected, display transactions from that budget -->
        <div v-else>
          <Transactions :transactions="transactions" />
          <button class="btn btn-info" @click="matchTransactions">Match</button>
          <button class="btn btn-info" @click="budgetId = null">
            &lt; Select Another Budget
          </button>
        </div>
      </div>

      <Footer />
    </div>
  </div>
</template>

<script>
import * as ynab from "ynab";

import Papa from "papaparse";

// Import our config for YNAB
import config from "./config.json";

// Import Our Components to Compose Our App
import Budgets from "./components/Budgets.vue";
import Footer from "./components/Footer.vue";
import Nav from "./components/Nav.vue";
import Transactions from "./components/Transactions.vue";
import Uploader from "./components/Uploader.vue";

export default {
  // The data to feed our templates
  data() {
    return {
      csvData: [],
      orderMap: {},
      ynab: {
        clientId: config.clientId,
        redirectUri: config.redirectUri,
        token: null,
        api: null,
      },
      loading: false,
      error: null,
      budgetId: null,
      budgets: [],
      transactions: [],
    };
  },
  // When this component is created, check whether we need to get a token,
  // budgets or display the transactions
  created() {
    this.ynab.token = this.findYNABToken();
    if (this.ynab.token) {
      this.api = new ynab.api(this.ynab.token);
      if (!this.budgetId) {
        this.getBudgets();
      } else {
        this.selectBudget(this.budgetId);
      }
    }
  },
  methods: {
    // This uses the YNAB API to get a list of budgets
    convertMilliUnitsToCurrencyAmount:
      ynab.utils.convertMilliUnitsToCurrencyAmount,
    getBudgets() {
      this.loading = true;
      this.error = null;
      this.api.budgets
        .getBudgets()
        .then((res) => {
          this.budgets = res.data.budgets;
        })
        .catch((err) => {
          this.error = err.error.detail;
        })
        .finally(() => {
          this.loading = false;
        });
    },
    // This selects a budget and gets all the transactions in that budget
    selectBudget(id) {
      this.loading = true;
      this.error = null;
      this.budgetId = id;
      this.transactions = [];
      this.api.transactions
        .getTransactionsByType(id, "unapproved")
        .then((res) => {
          this.transactions = res.data.transactions
            .filter((transaction) => transaction.payee_name === "Amazon")
            .map((item) => {
              //normalize amounts
              item.amount = ynab.utils
                .convertMilliUnitsToCurrencyAmount(item.amount)
                .toFixed(2)
                .replace(/^-/, "");
              return item;
            });
        })
        .catch((err) => {
          console.log(err);
          // this.error = err.error.detail;
        })
        .finally(() => {
          this.loading = false;
        });
    },
    // This builds a URI to get an access token from YNAB
    // https://api.ynab.com/#outh-applications
    authorizeWithYNAB(e) {
      e.preventDefault();
      const uri = `https://app.ynab.com/oauth/authorize?client_id=${this.ynab.clientId}&redirect_uri=${this.ynab.redirectUri}&response_type=token`;
      location.replace(uri);
    },
    // Method to find a YNAB token
    // First it looks in the location.hash and then sessionStorage
    findYNABToken() {
      let token = null;
      const search = window.location.hash
        .substring(1)
        .replace(/&/g, '","')
        .replace(/=/g, '":"');
      if (search && search !== "") {
        // Try to get access_token from the hash returned by OAuth
        const params = JSON.parse('{"' + search + '"}', function (key, value) {
          return key === "" ? value : decodeURIComponent(value);
        });
        token = params.access_token;
        sessionStorage.setItem("ynab_access_token", token);
        window.location.hash = "";
      } else {
        // Otherwise try sessionStorage
        token = sessionStorage.getItem("ynab_access_token");
      }
      return token;
    },
    // Clear the token and start authorization over
    resetToken() {
      sessionStorage.removeItem("ynab_access_token");
      this.ynab.token = null;
      this.error = null;
    },
    handleParseComplete(result) {
      this.csvData = result.data;
      result.data.map((order) => {
        //normalize
        order.price = order.price.replace(/^\$/, "");
        //add to map
        if (!this.orderMap[order.price]) {
          this.orderMap[order.price] = [order.description];
        } else {
          this.orderMap[order.price].push(order.description);
        }
      });
    },
    handleFileUpload(event) {
      const file = event.target.files[0];
      if (file && file.type === "text/csv") {
        Papa.parse(file, {
          complete: this.handleParseComplete,
          header: true,
          transformHeader: function (h) {
            return h.replace(/\s/g, "");
          },
        });
      } else {
        alert("Please upload a valid CSV file.");
      }
    },
    matchTransactions(e) {
      e.preventDefault();
      this.transactions = this.transactions.map((item) => {
        console.log(this.orderMap);
        item.memo = this.orderMap[item.amount];
        return item;
      });
    },
  },
  // Specify which components we want to make available to our templates
  components: {
    Nav,
    Footer,
    Uploader,
    Budgets,
    Transactions,
  },
};
</script>
