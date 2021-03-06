<template>
  <div class="paginate-table">
    <div class="el-table"
      :class="{
        'el-table--fit': fit,
        'el-table--striped': stripe,
        'el-table--border': border,
        'el-table--fluid-height': maxHeight,
        'el-table--enable-row-hover': !store.states.isComplex,
        'el-table--enable-row-transition': true || (store.states.data || []).length !== 0 && (store.states.data || []).length < 100
      }"
      @mouseleave="handleMouseLeave($event)">
      <div class="hidden-columns" ref="hiddenColumns"><slot></slot></div>
      <div class="el-table__header-wrapper" ref="headerWrapper" v-if="showHeader">
        <table-header
          :store="store"
          :layout="layout"
          :border="border"
          :default-sort="defaultSort"
          :style="{ width: layout.bodyWidth ? layout.bodyWidth + 'px' : '' }">
        </table-header>
      </div>
      <div
        class="el-table__body-wrapper"
        ref="bodyWrapper"
        :style="[bodyHeight]">
        <table-body
          :context="context"
          :store="store"
          :layout="layout"
          :row-class-name="rowClassName"
          :row-style="rowStyle"
          :highlight="highlightCurrentRow"
          :style="{ width: bodyWidth }">
        </table-body>
        <div :style="{ width: bodyWidth }" class="el-table__empty-block" v-if="!data || data.length === 0">
          <span class="el-table__empty-text"><slot name="empty">{{ emptyText || t('el.table.emptyText') }}</slot></span>
        </div>
      </div>
      <div class="el-table__fixed" ref="fixedWrapper"
        v-if="fixedColumns.length > 0"
        :style="[
          { width: layout.fixedWidth ? layout.fixedWidth + 'px' : '', overflowY: 'hidden'},
          fixedHeight
        ]">
        <div class="el-table__fixed-header-wrapper" ref="fixedHeaderWrapper" v-if="showHeader">
          <table-header
            fixed="left"
            :border="border"
            :store="store"
            :layout="layout"
            :style="{ width: layout.fixedWidth ? layout.fixedWidth + 'px' : '' }"></table-header>
        </div>
        <div class="el-table__fixed-body-wrapper" ref="fixedBodyWrapper"
          :style="[
            { top: layout.headerHeight + 'px' },
            fixedBodyHeight
          ]">
          <table-body
            fixed="left"
            :store="store"
            :layout="layout"
            :highlight="highlightCurrentRow"
            :row-class-name="rowClassName"
            :row-style="rowStyle"
            :style="{ width: layout.fixedWidth ? layout.fixedWidth + 'px' : '' }">
          </table-body>
        </div>
      </div>
      <div class="el-table__fixed-right" ref="rightFixedWrapper"
        v-if="rightFixedColumns.length > 0"
        :style="[
          { width: layout.rightFixedWidth ? layout.rightFixedWidth + 'px' : '' },
          { right: layout.scrollY ? (border ? layout.gutterWidth : (layout.gutterWidth || 1)) + 'px' : '' },
          { overflowY: 'hidden '},
          fixedHeight
        ]">
        <div class="el-table__fixed-header-wrapper" ref="rightFixedHeaderWrapper" v-if="showHeader">
          <table-header
            fixed="right"
            :border="border"
            :store="store"
            :layout="layout"
            :style="{ width: layout.rightFixedWidth ? layout.rightFixedWidth + 'px' : '' }"></table-header>
        </div>
        <div class="el-table__fixed-body-wrapper" ref="rightFixedBodyWrapper"
          :style="[
            { top: layout.headerHeight + 'px' },
            fixedBodyHeight
          ]">
          <table-body
            fixed="right"
            :store="store"
            :layout="layout"
            :row-class-name="rowClassName"
            :row-style="rowStyle"
            :highlight="highlightCurrentRow"
            :style="{ width: layout.rightFixedWidth ? layout.rightFixedWidth + 'px' : '' }">
          </table-body>
        </div>
      </div>
      <div class="el-table__fixed-right-patch"
        v-if="rightFixedColumns.length > 0"
        :style="{ width: layout.scrollY ? layout.gutterWidth + 'px' : '0', height: layout.headerHeight + 'px' }"></div>
      <div class="el-table__column-resize-proxy" ref="resizeProxy" v-show="resizeProxyVisible"></div>
    </div>
    <el-pagination ref="pagination"
    :class="paginateOption.class" 
    :total="paginateOption.total"
    :page-size="paginateOption.pageSize"
    :current-page="paginateOption.currentPage"
    :page-sizes="paginateOption.pageSizesArr"
    :layout="paginateOption.layout"
    @current-change="handleCurrentChange" @size-change="handleSizeChange"
    ></el-pagination>
  </div>
  
</template>

<script type="text/babel">
    import {
        Pagination
    } from 'element-ui'
    // import ElCheckbox from 'element-ui/packages/checkbox';
    import throttle from 'throttle-debounce/throttle';
    import debounce from 'throttle-debounce/debounce';
    import {
        addResizeListener,
        removeResizeListener
    } from 'element-ui/src/utils/resize-event';
    import Locale from 'element-ui/src/mixins/locale';
    import TableStore from './table-store';
    import TableLayout from './table-layout';
    import TableBody from './table-body';
    import TableHeader from './table-header';
    import {
        mousewheel,
        isArray
    } from './util';

    let tableIdSeed = 1;

    export default {
        name: 'ElTable',

        mixins: [Locale],

        props: {
            dataList: Array,

            paginateUserOption: {},

            width: [String, Number],

            height: [String, Number],

            maxHeight: [String, Number],

            fit: {
                type: Boolean,
                default: true
            },

            stripe: Boolean,

            border: Boolean,

            rowKey: [String, Function],

            context: {},

            showHeader: {
                type: Boolean,
                default: true
            },

            rowClassName: [String, Function],

            rowStyle: [Object, Function],

            highlightCurrentRow: Boolean,

            currentRowKey: [String, Number],

            emptyText: String,

            expandRowKeys: Array,

            defaultExpandAll: Boolean,

            defaultSort: Object,

            tooltipEffect: String
        },

        components: {
            TableHeader,
            TableBody,
            // ElCheckbox,
            Pagination
        },

        methods: {
            getPaginateDataList(dataList) {
                let itemArr = [];
                let dataArr;
                dataArr = dataList ? dataList : this.tableDataList
                if (this.paginateDataList.length) {
                    this.paginateDataList = []
                }
                if (!dataArr.length) {
                    return
                }
                dataArr.forEach((item, index) => {
                    itemArr.push(item)
                        // 最后一页 index + 1 === arr.length
                    if ((index + 1) % this.paginateOption.pageSize === 0 || index + 1 === dataArr.length) {
                        this.paginateDataList.push(itemArr)
                        itemArr = []
                    }
                })
            },
            handleCurrentChange(val) {
                this.data = val;
            },
            handleSizeChange(val) {
                this.paginateOption.pageSize = val;
                this.getPaginateDataList()
                this.data = 'pageSizesChange'
            },
            toggleRowSelection(row, selected) {
                this.store.toggleRowSelection(row, selected);
                this.store.updateAllSelected();
            },

            clearSelection() {
                this.store.clearSelection();
            },

            handleMouseLeave() {
                this.store.commit('setHoverRow', null);
                if (this.hoverState) this.hoverState = null;
            },

            updateScrollY() {
                this.layout.updateScrollY();
            },

            bindEvents() {
                const {
                    headerWrapper
                } = this.$refs;
                const refs = this.$refs;
                this.bodyWrapper.addEventListener('scroll', function() {
                    if (headerWrapper) headerWrapper.scrollLeft = this.scrollLeft;
                    if (refs.fixedBodyWrapper) refs.fixedBodyWrapper.scrollTop = this.scrollTop;
                    if (refs.rightFixedBodyWrapper) refs.rightFixedBodyWrapper.scrollTop = this.scrollTop;
                });

                if (headerWrapper) {
                    mousewheel(headerWrapper, throttle(16, event => {
                        const deltaX = event.deltaX;

                        if (deltaX > 0) {
                            this.bodyWrapper.scrollLeft += 10;
                        } else {
                            this.bodyWrapper.scrollLeft -= 10;
                        }
                    }));
                }

                if (this.fit) {
                    this.windowResizeListener = throttle(50, () => {
                        if (this.$ready) this.doLayout();
                    });
                    addResizeListener(this.$el, this.windowResizeListener);
                }
            },

            doLayout() {
                this.store.updateColumns();
                this.layout.update();
                this.updateScrollY();
                this.$nextTick(() => {
                    if (this.height) {
                        this.layout.setHeight(this.height);
                    } else if (this.maxHeight) {
                        this.layout.setMaxHeight(this.maxHeight);
                    } else if (this.shouldUpdateHeight) {
                        this.layout.updateHeight();
                    }
                });
            }
        },

        created() {
            this.tableId = 'el-table_' + tableIdSeed + '_';
            this.debouncedLayout = debounce(50, () => this.doLayout());
        },

        computed: {
            data: {
                get() {
                    return this.currentData || this.paginateDataList[0]
                },
                set(data) {
                    this.currentData = data === 'pageSizesChange' ?  this.paginateDataList[0] : this.paginateDataList[data - 1]
                }
            },
            paginateOption() {
                return Object.assign({}, this.paginateDefaultOption, this.paginateUserOption)
            },

            bodyWrapper() {
                return this.$refs.bodyWrapper;
            },

            shouldUpdateHeight() {
                return typeof this.height === 'number' ||
                    this.fixedColumns.length > 0 ||
                    this.rightFixedColumns.length > 0;
            },

            selection() {
                return this.store.states.selection;
            },

            columns() {
                return this.store.states.columns;
            },

            tableData() {
                return this.store.states.data;
            },

            tableDataList() {
                return this.store.states.dataList;
            },

            fixedColumns() {
                return this.store.states.fixedColumns;
            },

            rightFixedColumns() {
                return this.store.states.rightFixedColumns;
            },

            bodyHeight() {
                let style = {};

                if (this.height) {
                    style = {
                        height: this.layout.bodyHeight ? this.layout.bodyHeight + 'px' : ''
                    };
                } else if (this.maxHeight) {
                    style = {
                        'max-height': (this.showHeader ? this.maxHeight - this.layout.headerHeight : this.maxHeight) + 'px'
                    };
                }

                return style;
            },

            bodyWidth() {
                const {
                    bodyWidth,
                    scrollY,
                    gutterWidth
                } = this.layout;
                return bodyWidth ? bodyWidth - (scrollY ? gutterWidth : 0) + 'px' : '';
            },

            fixedBodyHeight() {
                let style = {};

                if (this.height) {
                    style = {
                        height: this.layout.fixedBodyHeight ? this.layout.fixedBodyHeight + 'px' : ''
                    };
                } else if (this.maxHeight) {
                    let maxHeight = this.layout.scrollX ? this.maxHeight - this.layout.gutterWidth : this.maxHeight;

                    if (this.showHeader) {
                        maxHeight -= this.layout.headerHeight;
                    }

                    style = {
                        'max-height': maxHeight + 'px'
                    };
                }

                return style;
            },

            fixedHeight() {
                let style = {};

                if (this.maxHeight) {
                    style = {
                        bottom: (this.layout.scrollX && this.data.length) ? this.layout.gutterWidth + 'px' : ''
                    };
                } else {
                    style = {
                        height: this.layout.viewportHeight ? this.layout.viewportHeight + 'px' : ''
                    };
                }

                return style;
            }
        },

        watch: {
            height(value) {
                this.layout.setHeight(value);
            },

            currentRowKey(newVal) {
                this.store.setCurrentRowKey(newVal);
            },

            tableDataList(v) {
                this.currentData = null
                this.$refs.pagination.internalCurrentPage = 1
                this.getPaginateDataList();
            },
            dataList: {
                immediate: true,
                handler(val) {
                    if (isArray(val)) {
                        this.store.commit('setDataList', val);
                    }
                }
            },

            data: {
                immediate: true,
                handler(val) {
                    this.store.commit('setData', val);
                }
            },

            expandRowKeys(newVal) {
                this.store.setExpandRowKeys(newVal);
            }
        },

        destroyed() {
            if (this.windowResizeListener) removeResizeListener(this.$el, this.windowResizeListener);
        },

        mounted() {
            this.bindEvents();
            this.doLayout();

            // init filters
            this.store.states.columns.forEach(column => {
                if (column.filteredValue && column.filteredValue.length) {
                    this.store.commit('filterChange', {
                        column,
                        values: column.filteredValue,
                        silent: true
                    });
                }
            });

            this.$ready = true;
        },

        data() {
            const store = new TableStore(this, {
                rowKey: this.rowKey,
                defaultExpandAll: this.defaultExpandAll
            });
            const layout = new TableLayout({
                store,
                table: this,
                fit: this.fit,
                showHeader: this.showHeader
            });
            return {
                paginateDefaultOption: {
                    total: 0,
                    pageSize: 10,
                    currentPage: 1,
                    layout: 'total, prev, pager, next, jumper',
                    class: 'textRight',
                },
                currentData: null, // 当前页数据 利用computed方法
                paginateDataList: [],
                store,
                layout,
                renderExpanded: null,
                resizeProxyVisible: false
            };
        }
    };
</script>