<template>
  <div class="container">
    <h5>Unapproved Amazon Transactions</h5>
    <table class="table">
      <thead>
        <tr>
          <th>Account</th>
          <th>Date</th>
          <th>Payee</th>
          <th>Memo</th>
          <th>Amount</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="transaction in transactions" v-bind:key="transaction.id">
          <td>{{ transaction.account_name }}</td>
          <td>{{ transaction.date }}</td>
          <td>{{ transaction.payee_name }}</td>
          <td>{{ transaction.memo }}</td>
          <td>{{ transaction.amount }}</td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<script>
export default {
  props: ["transactions"],
};
</script>
