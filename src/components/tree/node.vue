<template>
    <ul :class="classes">
        <li @contextmenu.stop="handleContextmenu(data, $event)" @selectstart.stop="handlePreventSelect(data, $event)">
                <span :class="arrowClasses" @click="handleExpand">
                    <Icon v-if="showArrow" :type="arrowType" :custom="customArrowType" :size="arrowSize" />
                    <Icon v-if="showLoading" type="ios-loading" class="ivu-load-loop" />
                </span>
            <Checkbox
                v-if="showCheckbox"
                :model-value="data.checked"
                :indeterminate="data.indeterminate"
                :disabled="data.disabled || data.disableCheckbox"
                @click.prevent="handleCheck"></Checkbox>
            <span :class="titleClasses" @click="handleClickNode">
                <Render v-if="data.render" :render="data.render" :data="data" :node="node"></Render>
                <Render v-else-if="isParentRender" :render="parentRender" :data="data" :node="node"></Render>
                <template v-else>{{ data.title }}</template>
            </span>
            <collapse-transition :appear="appear">
                <div class="ivu-tree-expand" v-if="data.expand">
                    <TreeNode
                        :appear="appearByClickArrow"
                        v-for="(item, i) in children"
                        :key="i"
                        :data="item"
                        :multiple="multiple"
                        :show-checkbox="showCheckbox"
                        :children-key="childrenKey">
                    </TreeNode>
                </div>
            </collapse-transition>
        </li>
    </ul>
</template>
<script>
    import { getCurrentInstance, nextTick } from 'vue';
    import Checkbox from '../checkbox/checkbox.vue';
    import Icon from '../icon/icon.vue';
    import Render from './render';
    import CollapseTransition from '../base/collapse-transition.vue';
    import { findComponentUpward } from '../../utils/assist';

    const prefixCls = 'ivu-tree';

    export default {
        name: 'TreeNode',
        inject: ['TreeInstance'],
        components: { Checkbox, Icon, CollapseTransition, Render },
        props: {
            data: {
                type: Object,
                default: () => {}
            },
            multiple: {
                type: Boolean,
                default: false
            },
            childrenKey: {
                type: String,
                default: 'children'
            },
            showCheckbox: {
                type: Boolean,
                default: false
            },
            appear: {
                type: Boolean,
                default: false
            }
        },
        data () {
            return {
                prefixCls: prefixCls,
                appearByClickArrow: false,
                globalConfig: {}
            };
        },
        computed: {
            classes () {
                return [
                    `${prefixCls}-children`
                ];
            },
            selectedCls () {
                return [
                    {
                        [`${prefixCls}-node-selected`]: this.data.selected
                    }
                ];
            },
            arrowClasses () {
                return [
                    `${prefixCls}-arrow`,
                    {
                        [`${prefixCls}-arrow-disabled`]: this.data.disabled,
                        [`${prefixCls}-arrow-open`]: this.data.expand
                    }
                ];
            },
            titleClasses () {
                return [
                    `${prefixCls}-title`,
                    {
                        [`${prefixCls}-title-selected`]: this.data.selected
                    }
                ];
            },
            showArrow () {
                return (this.data[this.childrenKey] && this.data[this.childrenKey].length) || ('loading' in this.data && !this.data.loading);
            },
            showLoading () {
                return 'loading' in this.data && this.data.loading;
            },
            isParentRender () {
                const Tree = findComponentUpward(this, 'Tree');
                return Tree && Tree.render;
            },
            parentRender () {
                const Tree = findComponentUpward(this, 'Tree');
                if (Tree && Tree.render) {
                    return Tree.render;
                } else {
                    return null;
                }
            },
            node () {
                const Tree = findComponentUpward(this, 'Tree');
                if (Tree) {
                    // 将所有的 node（即flatState）和当前 node 都传递
                    return [Tree.flatState, Tree.flatState.find(item => item.nodeKey === this.data.nodeKey)];
                } else {
                    return [];
                }
            },
            children () {
                return this.data[this.childrenKey];
            },
            // 3.4.0, global setting customArrow 有值时，arrow 赋值空
            arrowType () {
                const config = this.globalConfig;
                let type = 'ios-arrow-forward';

                if (config) {
                    if (config.tree.customArrow) {
                        type = '';
                    } else if (config.tree.arrow) {
                        type = config.tree.arrow;
                    }
                }
                return type;
            },
            // 3.4.0, global setting
            customArrowType () {
                const config = this.globalConfig;
                let type = '';

                if (config) {
                    if (config.tree.customArrow) {
                        type = config.tree.customArrow;
                    }
                }
                return type;
            },
            // 3.4.0, global setting
            arrowSize () {
                const config = this.globalConfig;
                let size = '';

                if (config) {
                    if (config.tree.arrowSize) {
                        size = config.tree.arrowSize;
                    }
                }
                return size;
            }
        },
        methods: {
            handleExpand () {
                const item = this.data;
                // if (item.disabled) return;

                // Vue.js 2.6.9 对 transition 的 appear 进行了调整，导致 iView 初始化时无动画，加此方法来判断通过点击箭头展开时，加 appear，否则初始渲染时 appear 为 false
                this.appearByClickArrow = true;

                // async loading
                if (item[this.childrenKey].length === 0) {
                    const tree = findComponentUpward(this, 'Tree');
                    if (tree && tree.loadData) {
                        this.data.loading = true; // eslint-disable-line
                        tree.loadData(item, children => {
                            this.data.loading = false; // eslint-disable-line
                            if (children.length) {
                                this.data[this.childrenKey] = children; // eslint-disable-line
                                nextTick(() => this.handleExpand());
                            }
                        });
                        return;
                    }
                }

                if (item[this.childrenKey] && item[this.childrenKey].length) {
                    this.data.expand = !this.data.expand; // eslint-disable-line
                    this.TreeInstance.handleToggleExpand(this.data);
                }
            },
            handleClickNode () {
                if (this.TreeInstance.expandNode) {
                    if (this.showArrow) this.handleExpand();
                } else if (this.TreeInstance.selectNode) {
                    this.handleSelect();
                }
            },
            handleSelect () {
                if (this.data.disabled) return;
                if (this.TreeInstance.showCheckbox && this.TreeInstance.checkDirectly) {
                    this.handleCheck();
                } else {
                    this.TreeInstance.handleOnSelected(this.data.nodeKey);
                }
            },
            handleCheck () {
                if (this.data.disabled) return;
                const changes = {
                    checked: !this.data.checked && !this.data.indeterminate,
                    nodeKey: this.data.nodeKey
                };
                this.TreeInstance.handleOnCheck(changes);
            },
            handleContextmenu (data, event) {
                if (data.contextmenu) {
                    event.preventDefault();
                    this.TreeInstance.handleOnContextmenu({ data, event });
                }
            },
            handlePreventSelect (data, event) {
                if (data.contextmenu) {
                    event.preventDefault();
                }
            }
        },
        created () {
            const instance = getCurrentInstance();
            this.globalConfig = instance.appContext.config.globalProperties.$VIEWUI;
        }
    };
</script>
