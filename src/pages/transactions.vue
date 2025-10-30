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
    this.$getService('iam/auth').authenticate();
    if (this.$moduleExists('iam')) {
      const permissions = this.$getService('iam/permissions');
      if (!this.$getService('iam/permissions').validatePermissions({ 'FNC_TRANSACTION': 'R' })) {
        this.$router.push('/error/forbidden');
      }
    }
  },
}
</script>