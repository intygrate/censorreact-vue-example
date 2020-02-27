<template>
  <div id="app">
    <appHeader />
    <appDemo/>
  </div>
</template>

<script>
import appHeader from './components/main/header';
import appDemo from './components/appDemo/appDemo';

export default {
  name: 'App',
  components: {
    appHeader,
    appDemo,
  }
}
</script>

<style lang='scss'>

@import '@/assets/scss/styles';

</style>
