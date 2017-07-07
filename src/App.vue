<template>
  <div id="app">
    <nav class="navbar ">
      <div class="navbar-brand">
        <a class="navbar-item app-name">CSV File Viewer</a>
      </div>
    </nav>
    <div class="main-window">
      <main-window></main-window>
    </div>
  </div>
</template>

<script>
import MainWindow from './components/MainWindow'

export default {
  name: 'app',
  components: {
    MainWindow
  }
}
</script>

<style lang="scss">
@import "~bulma";
@import "~c3/c3.css";

#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: black;
}

.app-name {
  font-size: 20px;
  font-weight: bold;
}

.main-window {
  padding-bottom: 10px;
  padding-left: 20px;
  padding-right: 20px;
}
</style>
