<template>
<transition v-if="(currentVisible || animationVisible)"
    enter-active-class="animate__animated animate__fadeIn"
    leave-active-class="animate__animated animate__fadeOut animate__fast"
    >
    <div :class="$style.root" :static="this.static" @click="handleClose" :functional-modal="functionalModal" :style="$env.VUE_APP_DESIGNER ? {zIndex: 7020} : {}">
        <transition
            enter-active-class="animate__animated animate__fadeInDownSmall"
            leave-active-class="animate__animated animate__fadeOutUpSmall animate__fast">
            <div :class="[$style.dialog, $env.VUE_APP_DESIGNER ? $style.pos : null, customClass]" ref="dialog"
                v-if="currentVisible && animationVisible"
                :style="{ width: width + 'px' }" :size="size" @click.stop="noop">
                <slot name="inject"></slot>
                <div :class="$style.head" vusion-slot-name="head" :child-cut-disabled="true" v-if="showHead">
                    <slot name="head">
                        <div v-if="(title || $slots.title) && !isTitleSlotEmpty || $env.VUE_APP_DESIGNER" vusion-slot-name="title" vusion-slot-name-edit="title" :class="$style.title" :child-cut-disabled="true">
                            <slot name="title">
                                <s-empty v-if="(!$slots.title) && $env.VUE_APP_DESIGNER && !!$attrs['vusion-node-path']"></s-empty>
                                <template v-else>
                                    {{ title }}
                                </template>
                            </slot>
                        </div>
                        <a :class="$style.close" @click="cancel()"></a>
                    </slot>
                </div>
                <div :class="$style.body" :icon="icon" vusion-slot-name="body" :child-cut-disabled="true">
                    <slot name="body">
                        <s-empty v-if="(!$slots.body) && $env.VUE_APP_DESIGNER && !!$attrs['vusion-node-path']"></s-empty>
                        <div :class="$style.text" v-else>
                            <div :class="$style.heading"><slot name="heading">{{ heading }}</slot></div>
                            <div :class="$style.content"><slot>{{ content }}</slot></div>
                            <div v-if="!!description || $slots.description" :class="$style.description"><slot name="description">{{ description }}</slot></div>
                        </div>
                        <slot name="extra"></slot>
                    </slot>
                </div>
                <div :class="$style.foot" vusion-slot-name="foot" :child-cut-disabled="true" v-if="showFoot && (okButton || cancelButton) && (!isFootSlotEmpty || $env.VUE_APP_DESIGNER)">
                    <slot name="foot">
                        <s-empty v-if="(!$slots.foot) && $env.VUE_APP_DESIGNER && !!$attrs['vusion-node-path']"></s-empty>
                        <u-linear-layout gap="small" justify="end" v-else>
                            <u-button :class="$style.button" v-if="cancelButton" :color="primaryButton === 'cancelButton' ? 'primary' : ''" :disabled="disableCancel" @click="cancel()">{{ cancelButton }}</u-button>
                            <u-button :class="$style.button" v-if="okButton" :color="primaryButton === 'okButton' ? 'primary' : ''" :disabled="disableOk" @click="ok()">{{ okButton }}</u-button>
                        </u-linear-layout>
                    </slot>
                </div>
            </div>
        </transition>
    </div>
</transition>
</template>

<script>
import { sync } from '@lcap/vue2-utils';
import { clickOutside } from '../../directives';
import i18n from './i18n';
import MEmitter from '../m-emitter.vue';
import SEmpty from '../../components/s-empty.vue';
import i18nMixin from '../../mixins/i18n';

export const UModal = {
    name: 'u-modal',
    components: { SEmpty },
    directives: { clickOutside },
    mixins: [
      MEmitter,
      i18nMixin('u-modal'),
      sync({
        visible: 'currentVisible',
      }),
    ],
    // i18n,
    props: {
        visible: { type: Boolean, default: false },
        title: {
            type: String,
            default() {
                return this.$tt('dialog');
            },
        },
        content: String,
        heading: String,
        description: String,
        okButton: {
            type: String,
            default() {
                return this.$tt('ok');
            },
        },
        cancelButton: {
            type: String,
            default() {
                return this.$tt('cancel');
            },
        },
        primaryButton: { type: String, default: 'okButton' },
        disableOk: { type: Boolean, default: false },
        disableCancel: { type: Boolean, default: false }, // @deprecated
        width: { type: [String, Number], default: '' },
        size: { type: String, default: 'normal' },
        static: { type: Boolean, default: false },
        icon: { type: String, default: null },
        maskClose: { type: Boolean, default: false },
        disableEsc: { type: Boolean, default: false },
        customClass: { type: String, default: undefined },
        functionalModal: { type: Boolean, default: false },
        showFoot: { type: Boolean, default: true },
        isTitleSlotEmpty: { type: Boolean, default: false }, // 有插槽template但是不想展示的情况
        isFootSlotEmpty: { type: Boolean, default: false },
        showHead: { type: Boolean, default: true },
        disableRootElemShift:  { type: Boolean, default: false }, // 取消根节点移位
    },
    data() {
        return {
            currentVisible: this.visible,
            animationVisible: undefined,
        };
    },
    watch: {
        visible(visible) {
            this.currentVisible = visible;
        },
        currentVisible: {
            immediate: true,
            handler(visible) {
                this.$nextTick(() => this.animationVisible = visible);
                if (visible)
                    document.addEventListener('keydown', this.escPress);
                // 按esc退出弹框
                else
                    document.removeEventListener('keydown', this.escPress);
            },
        },
    },
    mounted() {
        if (!this.disableRootElemShift && this.$el && !this.static) {
            const container = window.LcapMicro && window.LcapMicro.appendTo ? window.LcapMicro.appendTo : document.body;
            container.appendChild(this.$el);
        }
    },
    destroyed() {
        if (this.$el && !this.disableRootElemShift) {
            const container = window.LcapMicro && window.LcapMicro.appendTo ? window.LcapMicro.appendTo : document.body;
            this.$el.parentNode === container && container.removeChild(this.$el);
        }

        if (this.currentVisible)
            document.removeEventListener('keydown', this.escPress);
    },
    methods: {
        open() {
            if (this.$emitPrevent('before-open', {}, this))
                return;
            // this.$emit('before-open', undefined, this);
            if (!this.$el) {
                const el = document.createElement('div');
                this.$mount(el);
                const container = window.LcapMicro && window.LcapMicro.appendTo ? window.LcapMicro.appendTo : document.body;
                container.appendChild(this.$el);
            }
            this.currentVisible = true;
            // 增加一个通过vm.open 打开弹窗标识符，防止多次触发事件(组件内部使用)
            this._openByMethod = true;
            this.$emit('open');
        },
        close(ok) {
            if (this.$emitPrevent('before-close', { ok }, this))
                return;
            this.currentVisible = false;
            this.$emit('update:visible', false);
            this.$emit('close', { ok }, this);
        },
        ok() {
            if (this.disableOk)
                return;
            this.$emit('ok', undefined, this);
            this.close(true);
        },
        cancel() {
            if (this.disableCancel)
                return;
            this.$emit('cancel', undefined, this);
            this.close(false);
        },
        escPress(event) {
            if (event.keyCode === 27 && !this.disableEsc)
                this.cancel();
        },
        handleClose(e) {
            if (!this.$refs.dialog)
                return false;
            if (this.maskClose && !this.$refs.dialog.contains(e.target))
                this.close();
        },
        noop() { // 因为dialog 直接用overlay 套了实际额内容，导致只能stop 方式阻止事件冒泡....
        },
    },
    install(Vue, id) {
        const Ctor = Vue.component(id);
        Vue.prototype.$alert = (content, title, okButton) => new Promise((resolve, reject) => {
            const propsData = typeof content === 'object' ? content : { content, title, okButton, cancelButton: '' };
            const instance = new Ctor({
                propsData: {
                    functionalModal: true,
                    ...propsData,
                },
            });

            instance.$on('ok', () => resolve(true));
            instance.open();
        });
        Vue.prototype.$confirm = (content, title, okButton, cancelButton) => new Promise((resolve, reject) => {
            const propsData = typeof content === 'object' ? content : { content, title, okButton, cancelButton: '' };
            const instance = new Ctor({
                propsData: {
                    functionalModal: true,
                    ...propsData,
                },
            });

            instance.$on('ok', () => resolve(true));
            instance.$on('cancel', () => reject(false));
            instance.open();
        });
        Vue.prototype.$confirmResult = (content, title, okButton, cancelButton) => new Promise((resolve, reject) => {
            const propsData = typeof content === 'object' ? content : { content, title, okButton, cancelButton: '' };
            const instance = new Ctor({
                propsData: {
                    functionalModal: true,
                    ...propsData,
                },
            });

            instance.$on('ok', () => resolve(true));
            instance.$on('cancel', () => resolve(false));
            instance.open();
        });
    },
};

export default UModal;
</script>

<style module>
.root {
    position: fixed!important;
    top: 0!important;
    right: 0!important;
    bottom: 0!important;
    left: 0!important;
    width: unset!important;
    height: unset!important;
    z-index: var(--z-index-modal)!important;
    -webkit-overflow-scrolling: touch;
    touch-action: cross-slide-y pinch-zoom double-tap-zoom;
    text-align: center;
    overflow: hidden;
    overflow-y: auto;
    background: rgba(0, 0, 0, 0.6);
    padding-top: 65px;
    padding-bottom: 30px;
    white-space: nowrap;
}

.root::before {
    content: '';
    display: inline-block;
    vertical-align: middle;
    height: 100%;
}

.dialog {
    position: relative;
    width: var(--modal-dialog-width);
    display: inline-block;
    vertical-align: middle;
    text-align: left;
    /* background: var(--modal-dialog-background); */
    background-color: var(--modal-dialog-background-color);
    background-image: var(--modal-dialog-background-image);
    background-size: var(--modal-dialog-background-size);
    background-repeat: var(--modal-dialog-background-repeat);
    background-position: var(--modal-dialog-background-position);

    border: 1px solid var(--modal-border-color);
    border-radius: var(--modal-dialog-border-radius);
    box-shadow: var(--modal-dialog-box-shadow);
    white-space: initial;
}

.pos {
    vertical-align: top;
    margin-top: 300px;
}

.dialog[size="small"] {
    width: var(--modal-dialog-width-small);
}

.dialog[size="normal"] {
    width: var(--modal-dialog-width);
}

.dialog[size="large"] {
    width: var(--modal-dialog-width-large);
}

.dialog[size="huge"] {
    width: var(--modal-dialog-width-huge);
}

.dialog[size="auto"] {
    width: auto;
    max-width: 99%;
}

.head {
    position: relative;
    padding: var(--modal-head-padding);
    border-bottom: 1px solid var(--modal-border-color);
}

.title {
    margin: 0;
    font-size: var(--modal-title-font-size);
    font-weight: var(--modal-title-font-weight);
}

.close {
    position: absolute;
    right: var(--modal-close-right);
    top: 50%;
    transform: translateY(-50%);
    line-height: 1;
    color: var(--modal-close-color);
}

.close:hover {
    color: var(--modal-close-color-hover);
}

.close::before {
content: "\e668";
    font-family: "lcap-ui-icons";
    font-style: normal;
    font-weight: normal;
    font-variant: normal;
    text-decoration: inherit;
    text-rendering: optimizeLegibility;
    text-transform: none;
    -moz-osx-font-smoothing: grayscale;
    -webkit-font-smoothing: antialiased;
    font-smoothing: antialiased;
    font-size: 12px;
}

.body {
    position: relative;
    margin: var(--modal-body-margin);
}

.body[icon]:not([icon=""]) {
    margin: 32px 48px;
    padding-left: calc(48px + 16px);
}

.body[icon]:not([icon=""]) .text {
    min-height: 48px;
}

.body[icon]:not([icon=""]) > :not(.text){
    min-height: 48px;
}

.body[icon]::before {
    position: absolute;
    left: 0;
    top: 0;
    background: radial-gradient(circle, #fff 45%, transparent 45%);
}

.body[icon="warning"]::before {
content: "\e654";
    font-family: "lcap-ui-icons";
    font-style: normal;
    font-weight: normal;
    font-variant: normal;
    text-decoration: inherit;
    text-rendering: optimizeLegibility;
    text-transform: none;
    -moz-osx-font-smoothing: grayscale;
    -webkit-font-smoothing: antialiased;
    font-smoothing: antialiased;
    color: #FFAF0F;
    font-size: 48px;
    line-height: 48px;
}

.body[icon="success"]::before {
content: "\e67c";
    font-family: "lcap-ui-icons";
    font-style: normal;
    font-weight: normal;
    font-variant: normal;
    text-decoration: inherit;
    text-rendering: optimizeLegibility;
    text-transform: none;
    -moz-osx-font-smoothing: grayscale;
    -webkit-font-smoothing: antialiased;
    font-smoothing: antialiased;
    color: #26BD71;
    font-size: 48px;
    line-height: 48px;
}

.body[icon="error"]::before {
content: "\e654";
    font-family: "lcap-ui-icons";
    font-style: normal;
    font-weight: normal;
    font-variant: normal;
    text-decoration: inherit;
    text-rendering: optimizeLegibility;
    text-transform: none;
    -moz-osx-font-smoothing: grayscale;
    -webkit-font-smoothing: antialiased;
    font-smoothing: antialiased;
    color: #F24957;
    font-size: 48px;
    line-height: 48px;
}

.content {
    color: var(--color-base);
    /* font-size: 16px; */
}

.description {
    font-size: var(--font-size-base);
    color: var(--font-second-color);
}

.foot {
    margin: var(--modal-foot-margin);
    padding: var(--modal-foot-padding);
    text-align: center;
    border-top: 1px solid var(--modal-border-color);
}

.foot [class^="u-linear-layout_"][direction="horizontal"] > *:not(:last-child) {
    margin-right: 10px;
}

@media (--small-window) {
    .dialog {
        width: auto;
        margin: 10px;
    }
}

.root[static] {
    position: static !important;
    padding: var(--space-base);
}

</style>
