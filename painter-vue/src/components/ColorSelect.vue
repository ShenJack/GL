<template>
    <div>
        <Modal
                v-model="showModal"
                title="选择颜色"
                v-bind:ok-text=okText
                cancel-text="'取消'"
                :closable="false"
                :mask-closable="false"
        >
            <colorPicker v-model="color"></colorPicker>
            <div slot="footer">
                <i-button type="text" size="large" @click.native="cancel">取消</i-button>
                <i-button type="primary" size="large" @click.native="ok">{{okText}}</i-button>
            </div>
        </Modal>

    </div>
</template>

<script>
    import colorPicker from 'vue-color/src/components/Sketch.vue'

    export default {
        name: "colorSelect",
        props: {
            showModal: {Boolean, default: false}
        },
        components: {
            colorPicker
        },
        data() {
            return {
                title: "编辑作物",
                okText: "确定",
                color:{},
            }
        },
        methods: {
            ok() {
                this.$emit('ok', this.color.hex);
            },
            cancel() {
                this.$emit('cancel');
            },
        }
    }
</script>

<style scoped>
    .form-level {
        margin-bottom: 10px;
    }

    .json-pair {
        margin-bottom: 10px;
    }
</style>