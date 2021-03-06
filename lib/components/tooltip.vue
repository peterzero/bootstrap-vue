<template>
    <!-- Container for possible title content. -->
    <div v-show="false" class="d-none" aria-hidden="true">
        <div ref="title"><slot></slot></div>
    </div>
</template>

<script>
    import ToolTip from '../classes/tooltip';
    import { isArray } from '../utils/array';
    import { assign } from '../utils/object';
    import { isElement } from '../utils/dom';
    import { observeDom, warn } from '../utils';
    
    export default {
        data() {
            return {
                toolTip: null
            }
        },
        props: {
            target: {
                // String ID of element, or element/component reference
                type: [String, Object]
            },
            title: {
                type: String,
                default: ''
            },
            triggers: {
                type: [String, Array],
                default: 'hover focus'
            },
            placement: {
                type: String,
                default: 'top'
            },
            delay: {
                type: Number,
                default: 0
            },
            offset: {
                type: [Number, String],
                default: 0
            },
            noFade: {
                type: Boolean,
                default: false
            },
            container: {
                // String ID of container, if null body is used (default)
                type: String,
                default: null
            }
        },
        mounted() {
            // We do this in a $nextTick in hopes that the target element is in the DOM
            // And that our children have rendered
            this.$nextTick(() => {
                const target = this.getTarget();
                if (target) {
                    // Instantiate ToolTip on target
                    this.toolTip = new ToolTip(target, this.getConfig(), this.$root);
                    // Listen to close signals from others
                    this.$on('close', this.onClose);
                    // Observe content Child changes so we can notify popper of possible size change
                    observeDom(this.$refs.title, this.updatePosition.bind(this), {
                        subtree: true,
                        childList: true,
                        attributes: true,
                        attributeFilter: ['class', 'style']
                    });
                } else {
                    warn("b-tooltip: 'target' element not found!");
                }
            });
        },
        deactivated() {
            // Called when component is inside a <keep-alive> and component taken offline
            if (this.toolTip) {
                this.toolTip.hide();
            }
        },
        updated() {
            // If content/props changes, etc
            if (this.toolTip) {
                this.toolTip.updateConfig(this.getConfig());
            }
        },
        beforeDestroy() {
            this.$off('close', this.onClose);
            if (this.toolTip) {
                this.toolTip.destroy();
                this.tooltip = null;
            }
            // bring our content back if needed
            this.bringItBack();
        },
        computed: {
            baseConfig() {
            const cont = this.container;
                return {
                    title: this.title.trim() || '',
                    placement: this.placement || 'top',
                    // Container curently needs to be an ID with '#' prepended, if null then body is used
                    container: cont ? (/^#/.test(cont) ? cont : `#${cont}`) : false,
                    delay: parseInt(this.delay, 10) || 0,
                    // Offset can be css distance. if no units, pixels are assumed
                    offset: this.offset || 0,
                    animation: Boolean(this.noFade),
                    trigger: isArray(this.triggers) ? this.triggers.join(' ') : this.triggers
                };
            }
        },
        methods: {
            onClose(callback) {
                if (this.toolTip) {
                    this.toolTip.hide(callback);
                } else if (typeof callback === 'function') {
                    callback();
                }
            },
            updatePosition() {
                if (this.toolTip) {
                    // Instruct popper to reposition popover if necessary
                    this.toolTip.update();
                }
            },
            getConfig() {
                const cfg = assign({}, this.baseConfig);
                if (this.$refs.title.innerHTML.trim()) {
                    // If slot has content, it overrides 'title' prop
                    // We use the DOM node as content to allow components!
                    cfg.title = this.$refs.title;
                    cfg.html = true;
                }
                // Callbacks so we can trigger events on component
                cfg.callbacks = {
                    show: this.onShow,
                    shown: this.onShown,
                    hide: this.onHide,
                    hidden: this.onHidden
                };
                return cfg;
            },
            getTarget() {
                const target = this.target;
                if (typeof target === 'string') {
                    // Assume ID of element
                    return document.getElementById(/^#/.test(target) ? target.slice(1) : target) || null;
                } else if (typeof target === 'object' && target.$el) {
                    // Component reference
                    return target.$el;
                } else if (typeof target === 'object' && isElement(target)) {
                    // Element reference
                    return target;
                }
                return null;
            },
            onShow(evt) {
                this.$emit('show', evt);
            },
            onShown(evt) {
                this.$emit('shown', evt);
            },
            onHide(evt) {
                this.$emit('hide', evt)
            },
            onHidden(evt) {
                // bring our content back if needed to keep Vue happy
                // Tooltip class will move it back to tip when shown again
                this.bringItBack();
                this.$emit('hidden', evt);
            },
            bringItBack() {
                if (this.$el && this.$refs.title) {
                    this.$el.appendChild(this.$refs.title);
                }
            }
        }
    };
</script>
