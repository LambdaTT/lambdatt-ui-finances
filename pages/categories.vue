<template>
  <Page Title="Finanças" :Breadcrumb="breadcrumb">
    <Category></Category>
  </Page>
</template>
<script>
// Services:
import { auth, permissions } from 'src/modules/lambdatt-ui-iam/services.js'
import Category from 'src/components/analytics/finances/transactions-category.vue';

export default {
  name: 'finances-pages-category',

  components: {
    Category,
  },

  computed: {
    breadcrumb() {
      return [
        { label: 'Home', icon: "fas fa-home", to: "/" },
        { label: 'Finanças', icon: "fas fa-hand-holding-dollar" },
      ]
    }
  },

  mounted() {
    auth.authenticate(this);
    // if (!permissions.canExecute('geral-ver-relatorios-dg2e1o2f2a1d8')) this.$router.push('/forbidden');
  },
}
</script>