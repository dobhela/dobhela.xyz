<script>
import Header from "./components/Header.vue"
import Hero from "./components/Hero.vue"

export default {
    components: {
        Header,
        Hero
    }
}
</script>

<template>
    <Header />
    <main>
        <Hero />
    </main>
</template>

<style>
    @import "./css/main.css"
</style>