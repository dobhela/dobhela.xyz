<script>
    export default {
        data() {
            return {
                links: {
                    home: {
                        "name": "home",
                        "href": "/"
                    },
                    about: {
                        "name": "about",
                        "href": "/about"
                    },
                    projects: {
                        "name": "projects",
                        "href": "/projects"
                    }
                }
            }
        }
    }
</script>

<template>
    <header>
        <nav class="themeswitch">
            
        </nav>
        <nav class="navlist">
            <a 
                v-for="link in links"
                :key="link._id"
                :href="link.href"
            >
                {{link.name}}
            </a>
            
        </nav>
    </header>
</template>

<style>
</style>