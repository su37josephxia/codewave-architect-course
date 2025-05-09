<template>
<span :class="[$style['lcp-iconv'], isSvgUrl(name) ? $style.iconcus : '', icotype==='left' ? $style.wrapleft : '']"
    @click="onClick($event)"
    @dblclick="onDblclick($event)"
    @contextmenu="onContextMenu($event)"
    @mouseenter="onmouseover($event)"
    @mouseleave="onmouseout($event)"
    @mousedown="onmousedown($event)"
>
    <span :class="[$style.iconwrap, isSvgUrl(name) ? $style.iconwrapsvg : '']">
        <online-svg-icon v-if="isSvgUrl(name)" :url="name" :purecss="!ie11()"></online-svg-icon>
        <svg :class="$style.iconsvg" aria-hidden="true" v-else>
            <use :xlink:href="`#${iconconfig.css_prefix_text}${getName()}`" />
        </svg>
    </span>
    <div :class="[$style.icontext, icotype==='left' ? $style.textleft : '']" vusion-slot-name="default" v-if="!endNotext">
        <slot></slot>
        <s-empty v-if="(!$slots.default) && $env.VUE_APP_DESIGNER && !notext && !!$attrs['vusion-node-path']"></s-empty>
    </div>
</span>
</template>

<script>
import './icon.js';
import iconconfig from './iconconfig.js';
import SEmpty from '../../components/s-empty.vue';
import encodeUrl from '../../utils/encodeUrl';

import { onlineSvgIcon, ie11 } from './online-svg-icon-vue2/src/main.js';

export default {
    name: 'i-ico',
    components: { SEmpty, onlineSvgIcon },
    props: {
        name: String,
        notext: Boolean,
        href: String,
        target: { type: String, default: '_self' },
        to: [String, Object],
        replace: { type: Boolean, default: false },
        append: { type: Boolean, default: false },
        decoration: { type: Boolean, default: true },
        download: { type: Boolean, default: false },
        destination: String,
        icotype: String,
        link: [String, Function],
    },
    data() {
        return {
            iconconfig,
            ie11,
        };
    },
    computed: {
        endNotext() {
            return this.icotype === 'only';
        },
    },
    methods: {
        getName() {
            const item = this.iconconfig.glyphs.find((v) => v.font_class === this.name);
            return item ? item.font_class : 'default';
        },
        async onClick(ev) {
            const props = this._props;
            const parent = this.$parent;
            this.$emit('click', ev);

            if (this.link) {
                const res = await this.$linkpao(props.link, props.target);
                if (res)
                    return;
            }
            function currentHref() {
                if (props.href !== undefined)
                    return encodeUrl(props.href);
                else if (parent && parent.$router && props.to !== undefined)
                    return encodeUrl(parent && parent.$router.resolve(props.to, parent && parent.$route, props.append).href);
                else
                    return undefined;
            }
            const hrefR = currentHref();

            if (hrefR === undefined) {
                let to;
                if (props.destination) {
                    if (props.destination.startsWith('http')) {
                        this.$linkpao(encodeUrl(props.destination));
                        return;
                    }
                    to = props.destination;
                }
                const currentTo = to || props.to;
                if (currentTo === undefined)
                    return;
                const $router = parent && parent.$router;
                const $route = parent && parent.$route;
                const { location, href } = $router.resolve(
                    currentTo,
                    $route,
                    props.append,
                );
                if(props.target === '_blank' && href) return window.open(href);
                props.replace ? $router.replace(location) : $router.push(location);
            } else {
                // eslint-disable-next-line no-inner-declarations
                function downloadClick() {
                    const a = document.createElement('a');
                    a.setAttribute('href', hrefR);
                    document.body.appendChild(a);
                    a.click();
                    setTimeout(() => {
                        document.body.removeChild(a);
                    }, 500);
                }
                downloadClick();
            }
        },
        onDblclick(ev) {
            this.$emit('dblclick', ev);
        },
        onContextMenu(ev) {
            ev.preventDefault();
            this.$emit('rightclick', ev);
        },
        onmouseover(ev) {
            this.$emit('over', ev);
        },
        onmouseout(ev) {
            this.$emit('out', ev);
        },
        onmousedown(ev) {
            this.$emit('down', ev);
        },
        isSvgUrl(name) {
            return name && name.indexOf('/') !== -1 && /\.svg/i.test(name);
        },
    },
};
</script>
<style module>
.root {
}
.lcp-iconv {
    display: inline-block;
    text-align: center;
    font-size: 16px;
}
.wrapleft {
    display: inline-flex;
    align-items: center;
}
.iconwrap {
    /* font-size: 16px;  */
    /* padding: 8px; */
}

.iconcus {
    /* color: #666; */
}
.iconwrapsvg {
    line-height: 1;
    vertical-align: middle;
}
.iconsvg {
    width: 1em; height: 1em;
    vertical-align: -0.15em;
    fill: currentColor;
    overflow: hidden;
}
.icontext {
    line-height: 1;
}
.icontext [class^="u-text_"]{
    font-style: initial;
    font-size: 14px;
    color: #666;
}
.textleft {
    display: inline-block;
    margin-left: 6px;
}
</style>
