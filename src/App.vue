<template>
    <v-app>
        <v-app-bar
            app
            color="primary"
            dark>
            WebSocket Tester
        </v-app-bar>
        <v-main>
            <Main/>
        </v-main>
    </v-app>
</template>

<script>
    import Main from './components/Main';
    export default {
        name: 'App',
        components: {
            Main,
        },
    };
</script>
