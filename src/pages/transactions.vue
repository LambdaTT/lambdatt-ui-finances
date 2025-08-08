<template>
  <La1Page Title="Finanças" :Breadcrumb="breadcrumb">
    <FinTransactions AccountId="1"></FinTransactions>
  </La1Page>
</template>
<script>
export default {
  name: 'finances-pages-transactions',

  computed: {
    breadcrumb() {
      return [
        { label: 'Home', icon: "fas fa-home", to: "/" },
        { label: 'Finanças', icon: "fas fa-hand-holding-dollar" },
      ]
    }
  },

  mounted() {
    this.$getService('iam/auth').authenticate(this);
    // if (!permissions.canExecute('geral-ver-relatorios-dg2e1o2f2a1d8')) this.$router.push('/forbidden');
  },
}
</script>