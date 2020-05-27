<template>
    <div id="app">
        <mask-tool ref="maskTool" @export="handleMaskToolExport"
                   src="https://upload.wikimedia.org/wikipedia/commons/9/9a/Gull_portrait_ca_usa.jpg"></mask-tool>

        <button @click="$refs.maskTool.createExport()" class="button is-primary">
            Export
        </button>
    </div>
</template>

<script>
    import MaskTool from './components/MaskTool.vue';

    export default {
        name: 'App',

        components: {
            MaskTool
        },
        methods: {
            handleMaskToolExport() {

            }
        }
    }
</script>

<style>
    #app {
        font-family: Avenir, Helvetica, Arial, sans-serif;
        -webkit-font-smoothing: antialiased;
        -moz-osx-font-smoothing: grayscale;
        text-align: center;
        color: #2c3e50;
        margin-top: 60px;
    }
</style>
