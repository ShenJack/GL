<template>
  <div id="app">
    <Painter/>
  </div>
</template>

<script>
import Painter from './components/Painter.vue'

export default {
  name: 'app',
  components: {
      Painter
  }
}
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
}
</style>
