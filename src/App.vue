
<template>
    <div class="app-wrap">
        <HomePage/>
    </div>
</template>

<script>
import HomePage from './components/HomePage.vue'

export default {
    name: 'App',
    components: {
        HomePage
    }
}
</script>

<style>
body{
    margin: 0;
    padding: 0;
}
#app {
    font-family: Avenir, Helvetica, Arial, sans-serif;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    color: #2c3e50;
    background: linear-gradient(to top, #243b55, #141e30);
    width: 100%;
    height: 100vh;
}
.app-wrap{
    width: 100%;
    height: 100%;
}
</style>
