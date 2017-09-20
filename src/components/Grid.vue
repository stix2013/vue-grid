<template>
  <div class="flex flex-column relative bg-white h-100 vg-container" ref="container">

    <div class="flex flex-column justify-center h-100" v-if="!initialized">
      <spinner size="large" message="Loading..."></spinner>
    </div>

    <div class="flex flex-column relative bg-white h-100 overflow-hidden" v-else>
      <!-- grid header row handle -->
      <grid-header-row-handle class="z-2 ba bg-near-white vg-td"
        :row-height="rowHeight"
        :row-handle-width="rowHandleWidth"
        @start-row-handle-resize="onStartRowHandleResize"
        v-if="initialized"
      ></grid-header-row-handle>

      <!-- grid header -->
      <div
        class="flex-none overflow-hidden bg-near-white vg-thead"
        :style="'margin-left: '+(rowHandleWidth+leftOfRenderColsWidth-scrollLeft+1)+'px'"
      >
        <grid-header
          :row-handle-width="rowHandleWidth"
          :columns="renderCols"
          @start-column-resize="onStartColumnResize"
          @determine-cell-auto-width="initializeColumnWidths"
        ></grid-header>
      </div>

      <!-- row handles -->
      <div class="absolute z-1" :style="'top: '+(rowHeight-scrollTop)+'px'">
        <grid-row-handle
          v-for="(row, index) in render_rows"
          class="ba bg-near-white vg-td"
          :key="index"
          :row-index="start+index"
          :row-height="rowHeight"
          :row-handle-width="rowHandleWidth"
        ></grid-row-handle>
      </div>

      <!-- grid body -->
      <div
        ref="tbody"
        class="flex-fill relative overflow-auto vg-tbody"
        :style="'margin-left: '+(rowHandleWidth+1)+'px'"
        @scroll="onScroll"
        v-resize="onResize"
      >
        <!-- yardsticks -->
        <div class="absolute top-0 left-0" :style="'z-index: -1; width: 1px; height: '+totalHeight+'px'"></div>
        <div class="absolute top-0 left-0" :style="'z-index: -1; height: 1px; width: '+totalWidth+'px'"></div>

        <!-- rows -->
        <grid-row
          v-for="(row, index) in render_rows"
          :key="index"
          :row="row"
          :row-index="start+index"
          :row-height="rowHeight"
          :columns="renderCols"
          :style="'padding-left: '+leftOfRenderColsWidth+'px'"
          @determine-cell-auto-width="initializeColumnWidths"
        ></grid-row>
      </div>
    </div>
  </div>
</template>

<script>
  import {
    DEFAULT_START,
    DEFAULT_LIMIT,
    DEFAULT_ROW_HEIGHT,
    DEFAULT_ROW_HANDLE_WIDTH,
    ROW_HANDLE_MIN_WIDTH,
    ROW_HANDLE_MAX_WIDTH,
    DEFAULT_COLUMN_WIDTH,
    COLUMN_MIN_WIDTH,
    COLUMN_MAX_WIDTH
  } from '../constants'
  // import _ from 'lodash'
  import uniqueId from 'lodash/uniqueId'
  import size from 'lodash/size'
  import first from 'lodash/first'
  import last from 'lodash/last'
  import omit from 'lodash/omit'
  import pick from 'lodash/pick'
  import mapValues from 'lodash/mapValues'
  import assign from 'lodash/assign'
  import debounce from 'lodash/debounce'
  import throttle from 'lodash/throttle'
  import isNil from 'lodash/isNil'
  import isNumber from 'lodash/isNumber'
  import isString from 'lodash/isString'
  import isArray from 'lodash/isArray'
  import cloneDeep from 'lodash/cloneDeep'
  import find from 'lodash/find'
  import get from 'lodash/get'
  import defaultTo from 'lodash/defaultTo'
  
  import axios from 'axios'
  import resize from 'vue-resize-directive'
  import GridHeaderRowHandle from './GridHeaderRowHandle.vue'
  import GridHeader from './GridHeader.vue'
  import GridRowHandle from './GridRowHandle.vue'
  import GridRow from './GridRow.vue'
  import Spinner from 'vue-simple-spinner'

  const getOffset = function (el) {
    let top = 0
    let left = 0

    do {
      if (!isNaN(el.offsetTop)) { top += el.offsetTop }
      if (!isNaN(el.offsetLeft)) { left += el.offsetLeft }
    } while (el === el.offsetParent)

    return { left, top }
  }

  export default {
    name: 'vue-grid',
    props: {
      'data-url': {
        type: String,
        default: ''
      },
      'custom-headers': {
        type: Object,
        default: () => { return {} }
      },
      /* show empty cells on vertical scroll */
      'live-scroll': {
        type: Boolean,
        default: false
      },
      /* remove off-screen cells from the DOM on horizontal scroll */
      'virtual-scroll': {
        type: Boolean,
        default: true
      }
    },
    components: {
      GridHeaderRowHandle,
      GridHeader,
      GridRowHandle,
      GridRow,
      Spinner
    },
    directives: {
      resize
    },
    watch: {
      scrollTop (val, oldVal) {
        this.$emit('scroll-top-change', val, oldVal)
      },
      scrollLeft (val, oldVal) {
        this.$emit('scroll-left-change', val, oldVal)
      },
      totalHeight (val, oldVal) {
        this.$emit('total-height-change', val, oldVal)
      },
      rendered_row_count (val, oldVal) {
        this.$emit('rendered-row-count-change', val, oldVal)
      },
      cached_row_count (val, oldVal) {
        this.$emit('cached-row-count-change', val, oldVal)
      },
      totalRowCount (val, oldVal) {
        this.$emit('total-row-count-change', val, oldVal)
      },
      first_visible_row (val, oldVal) {
        this.$emit('first-visible-row-change', val, oldVal)
      },
      last_visible_row (val, oldVal) {
        this.$emit('last-visible-row-change', val, oldVal)
      },
      visible_row_count (val, oldVal) {
        this.$emit('visible-row-count-change', val, oldVal)
      },
      first_visible_column (val, oldVal) {
        this.$emit('first-visible-column-change', val, oldVal)
      },
      last_visible_column (val, oldVal) {
        this.$emit('last-visible-column-change', val, oldVal)
      },
      visible_column_count (val, oldVal) {
        this.$emit('visible-column-count-change', val, oldVal)
      },
      metrics (val, oldVal) {
        this.$emit('metrics-change', val, oldVal)
      }
    },
    data () {
      return {
        // uid: _.uniqueId('vue-grid-'),
        uid: uniqueId('vue-grid-'),

        initialized: false,
        col_widths_initialized: false,

        start: DEFAULT_START, // initial start
        limit: DEFAULT_LIMIT, // initial limit
        totalRowCount: 0, // returned by query

        columns: [],
        resize_col: null,
        resize_row_handle: null,

        rows: [],
        cachedRows: {},
        rowHeight: DEFAULT_ROW_HEIGHT,
        rowHandleWidth: DEFAULT_ROW_HANDLE_WIDTH,

        clientHeight: 0,
        clientWidth: 0,

        container_offset_top: 0,
        container_offset_left: 0,

        offsetTop: 0,
        offsetLeft: 0,
        offsetHeight: 0,
        offsetWidth: 0,

        scrollTop: 0,
        scrollLeft: 0,

        mousedown_x: -1,
        mousedown_y: -1,
        mouseX: 0,
        mouseY: 0,

        is_horizontal_scroll_active: false
      }
    },
    computed: {
      totalHeight () {
        return this.rowHeight * this.totalRowCount
      },
      totalWidth () {
        return this.columns.map(column => column.pixelWidth).reduce((acc, width) => acc + width, 0)
      },
      rendered_row_count () {
        return size(this.rows)
      },
      cached_row_count () {
        return size(this.cachedRows)
      },
      first_visible_row () {
        return Math.floor(this.scrollTop / this.rowHeight)
      },
      last_visible_row () {
        return Math.min(Math.ceil((this.scrollTop + this.clientHeight) / this.rowHeight), this.totalRowCount)
      },
      visible_row_count () {
        return this.last_visible_row - this.first_visible_row
      },
      rows_in_cache () {
        let missingRows = false

        if (this.cached_row_count === 0) { return false }

        for (let i = this.start; i < this.start + this.limit; ++i) {
          if (i < this.totalRowCount && !this.cachedRows[i]) {
            missingRows = true
            break
          }
        }

        return !missingRows
      },
      render_rows () {
        if (this.liveScroll) { return this.rows }

        let rows = []
        for (let i = this.start; i < this.start + this.limit; ++i) {
          if (i < this.totalRowCount) { rows.push(this.cachedRows[i] || {}) }
        }
        return rows
      },
      left_of_render_cols () {
        if (!this.virtualScroll) { return [] }

        // this value should be an empty array since we're showing
        // all columns when scrolling horizontally
        if (this.is_potential_horizontal_scroll) { return [] }

        let left = (-1 * this.scrollLeft)
        return this.columns.filter(c => {
          const isOffscreenLeft = left + c.pixelWidth + 1 < 0
          left += c.pixelWidth + 1
          return isOffscreenLeft
        })
      },
      leftOfRenderColsWidth () {
        this.left_of_render_cols.map(c => c.pixelWidth + 1).reduce((sum, width) => sum + width, 0)
      },
      renderCols () {
        if (!this.virtualScroll) { return this.columns }

        // horizontal scroll operations are far more performance with
        // all columns visible since this is a normal browser scroll event
        // and there are no expensive calculations that need to happen
        if (this.is_potential_horizontal_scroll) { return this.columns }

        // if we haven't yet initialized our column widths,
        // we need to return all columns so that the cells
        // are rendered and the cell contents can be measured
        if (!this.col_widths_initialized) { return this.columns }

        let left = (-1 * this.scrollLeft)
        // return _.filter(this.columns, (c) => {
        return this.columns.filter(c => {
          const isOffscreenLeft = left + c.pixelWidth + 1 < 0
          const isOffscreenRight = left > this.clientWidth
          left += c.pixelWidth + 1
          return !isOffscreenLeft && !isOffscreenRight
        })
      },
      first_visible_column () {
        // return _.first(this.renderCols)
        return first(this.renderCols)
      },
      last_visible_column () {
        // return _.last(this.renderCols)
        return last(this.renderCols)
      },
      visible_column_count () {
        // return _.size(this.renderCols)
        return size(this.renderCols)
      },
      totalColumnCount () {
        // return _.size(this.columns)
        return size(this.columns)
      },
      resize_delta () {
        return this.mousedown_x === -1 ? 0 : this.mouseX - this.mousedown_x
      },
      vertical_scrollbar_left () {
        return this.offsetLeft + this.clientWidth
      },
      vertical_scrollbar_right () {
        return this.offsetLeft + this.offsetWidth
      },
      horizontal_scrollbar_top () {
        return this.offsetTop + this.clientHeight
      },
      horizontal_scrollbar_bottom () {
        return this.offsetTop + this.offsetHeight
      },
      is_potential_horizontal_scroll () {
        if (this.is_horizontal_scroll_active) { return true }

        // handle bottom-right corner: if we're closer to the vertical
        // scrollbar than we are to the horizontal scrollbar, return 'false'
        if (this.mouseX / this.vertical_scrollbar_left >
            this.mouseY / this.horizontal_scrollbar_top) { return false }

        // for the rest of the viewport, return 'true' when we get close
        // to the horizontal scrollbar
        if (this.mouseY >= this.horizontal_scrollbar_top * 2 / 3 &&
            this.mouseY < this.horizontal_scrollbar_bottom) {
          return true
        }

        return false
      },
      metrics () {
        let computedData = this._computedWatchers
        computedData = omit(computedData, ['metrics'])
        computedData = mapValues(computedData, (k, v) => { return this[v] })

        const filteredData = pick(this.$data, [
          'start',
          'limit',
          'rows',
          'totalRowCount',
          'columns',
          'totalColumnCount',
          'cachedRows',
          'rowHeight',
          'rowHandleWidth',
          'mouseX',
          'mouseY',
          'scrollTop',
          'scrollLeft',
          'offsetTop',
          'offsetLeft',
          'offsetHeight',
          'offsetWidth',
          'clientHeight',
          'clientWidth',
          'totalHeight',
          'totalWidth'
        ])

        return assign({}, filteredData, computedData)
      }
    },
    mounted () {
      this.active_xhr = null
      this.cancelXhr = null
      this.default_col_widths = {}

      // initialize container offset
      const offset = getOffset(this.$refs['container'])
      this.container_offset_top = offset.top
      this.container_offset_left = offset.left

      // initialize the client dimensions
      this.clientHeight = this.$el.clientHeight
      this.clientWidth = this.$el.clientWidth

      // establish our debounced and throttled methods (we need these functions
      // to be members of this component since we've run into reference
      // issues using debounce() and throttle() directly on the method
      this.tryFetchDebounced = debounce(this.tryFetch, 50, { leading: false, trailing: true })
      this.resizeRowHandleThrottled = throttle(this.resizeRowHandle, 20)
      this.resizeColumnThrottled = throttle(this.resizeColumn, 20)
      // this.scrollVerticalThrottled = debounce(this.scrollVertical, 5, { leading: false, trailing: true })
      this.scrollVerticalThrottled = throttle(this.scrollVertical, 5, { leading: false, trailing: true })

      // do our initial fetch
      this.tryFetch()

      // document-level mouse down
      this.onDocumentMousedown = (evt) => {
        this.mousedown_x = evt.pageX
        this.mousedown_y = evt.pageY
      }

      // document-level mouse up
      this.onDocumentMouseup = (evt) => {
        this.mousedown_x = -1
        this.mousedown_y = -1
        this.resize_col = null
        this.resize_row_handle = null
        this.is_horizontal_scroll_active = false
        this.updateStyle('cursor', '')
        this.updateStyle('noselect', '')
      }

      // document-level mouse move
      this.onDocumentMousemove = (evt) => {
        this.mouseX = evt.pageX
        this.mouseY = evt.pageY

        if (!isNil(this.resize_row_handle)) { this.resizeRowHandleThrottled() }

        if (!isNil(this.resize_col)) { this.resizeColumnThrottled() }
      }

      // create document-level event handlers
      document.addEventListener('mousedown', this.onDocumentMousedown)
      document.addEventListener('mouseup', this.onDocumentMouseup)
      document.addEventListener('mousemove', this.onDocumentMousemove)
    },
    beforeDestroy () {
      // destroy document-level event handlers
      document.removeEventListener('mousedown', this.onDocumentMousedown)
      document.removeEventListener('mouseup', this.onDocumentMouseup)
      document.removeEventListener('mousemove', this.onDocumentMousemove)
    },
    methods: {
      getFetchUrl (start, limit) {
        const url = this.dataUrl + '?start=' + start + '&limit=' + limit
        return this.initialized ? url : url + '&metadata=true'
      },
      tryFetch () {
        const me = this

        // we need to get the start and limit before we do the AJAX call this the grid's
        // start and limit could potentially have changed by the time the callback resolves
        const fetchStart = this.start
        const fetchLimit = this.limit
        const fetchUrl = this.getFetchUrl(fetchStart, fetchLimit)

        // if the last XHR is still active, kill it now
        if (!isNil(this.active_xhr) && !isNil(this.cancelXhr)) {
          this.cancelXhr()

          // reset XHR variables
          this.active_xhr = null
          this.cancelXhr = null
        }

        // if all of the rows exist in our row cache, we're done
        if (this.rows_in_cache) { return }

        const CancelToken = axios.CancelToken
        this.active_xhr = axios.get(fetchUrl, {
          headers: this.customHeaders,
          cancelToken: new CancelToken(function executor (c) {
            // an executor function receives a cancel function as a parameter
            me.cancelXhr = c
          })
        }).then(response => {
          const resdata = response.data

          if (isNumber(resdata.total_count)) { this.totalRowCount = resdata.total_count }

          // store our column info
          if (!this.initialized && isArray(resdata.columns)) {
            // include default column info with each column
            const tempCols = resdata.columns.map((col) => {
              return assign({ pixelWidth: DEFAULT_COLUMN_WIDTH }, col)
            })

            this.columns = [...tempCols]
          }

          // store the current set of rows
          this.rows = [...resdata.rows]

          // cache the current set of rows
          const start = fetchStart
          const limit = fetchLimit
          const rowCount = this.totalRowCount
          let tempCachedRows = {}
          let idx = 0

          for (let r = start; r < start + limit && r < rowCount; ++r) {
            tempCachedRows[r] = this.rows[idx]
            idx++
          }

          // add the temporary cached rows to our stored cached rows
          const cacheRows = Object.assign(this.cached_rows, tempCachedRows)
          this.cachedRows = assign({}, cacheRows)

          // set our init flag to true so we don't get columns after this
          this.initialized = true

          // reset XHR variables
          this.active_xhr = null
          this.cancelXhr = null
        })
      },

      onStartRowHandleResize (col) {
        this.resize_row_handle = { old_width: this.rowHandleWidth }
        this.updateStyle('cursor', 'html, body { cursor: ew-resize !important; }')
        this.updateStyle('noselect', 'html, body { -moz-user-select: none !important; user-select: none !important }')
      },

      onStartColumnResize (col) {
        this.resize_col = cloneDeep(col)
        this.updateStyle('cursor', 'html, body { cursor: ew-resize !important; }')
        this.updateStyle('noselect', 'html, body { -moz-user-select: none !important; user-select: none !important }')
      },

      onResize (resizeEl) {
        // var container_el = this.$refs['container']
        this.offsetTop = this.container_offset_top + resizeEl.offsetTop
        this.offsetLeft = this.container_offset_left + resizeEl.offsetLeft
        this.offsetHeight = resizeEl.offsetHeight
        this.offsetWidth = resizeEl.offsetWidth
        this.clientHeight = resizeEl.clientHeight
        this.clientWidth = resizeEl.clientWidth

        this.$nextTick(() => {
          // check to see if all of the visible rows have been queried for
          if (this.last_visible_row >= this.start + this.rendered_row_count) {
            this.start = this.first_visible_row
            this.tryFetchDebounced()
          }
        })
      },

      onScroll () {
        var newScrollTop = this.$refs['tbody'].scrollTop
        var newScrollLeft = this.$refs['tbody'].scrollLeft

        // vertical scrolls
        if (this.scrollTop !== newScrollTop) { return this.scrollVerticalThrottled(newScrollTop, this.scrollTop) }

        // horizontal scrolls
        if (this.scrollLeft !== newScrollLeft) { return this.scrollHorizontal(newScrollLeft, this.scrollLeft) }
      },

      scrollVertical (val, oldVal) {
        this.scrollTop = val

        // scrolling down
        if (this.last_visible_row >= this.start + this.rendered_row_count) {
          this.start = this.first_visible_row
          this.tryFetchDebounced()
        }

        // scrolling up
        if (this.first_visible_row < this.start) {
          this.start = this.first_visible_row
          this.tryFetchDebounced()
        }
      },

      scrollHorizontal (val, oldVal) {
        this.is_horizontal_scroll_active = true
        this.scrollLeft = val
      },

      resizeRowHandle () {
        const oldWidth = this.resize_row_handle.old_width
        let newWidth = oldWidth + this.resize_delta
        newWidth = Math.max(ROW_HANDLE_MIN_WIDTH, newWidth)
        newWidth = Math.min(ROW_HANDLE_MAX_WIDTH, newWidth)
        this.rowHandleWidth = newWidth
      },

      resizeColumn () {
        var lookupCol = find(this.columns, { name: get(this.resize_col, 'name') })
        if (!isNil(lookupCol)) {
          const tempCols = this.columns.map((col) => {
            if (get(col, 'name') === get(lookupCol, 'name')) {
              const oldWidth = get(this.resize_col, 'pixelWidth', DEFAULT_COLUMN_WIDTH)
              let pixelWidth = oldWidth + this.resize_delta
              pixelWidth = Math.max(COLUMN_MIN_WIDTH, pixelWidth)
              pixelWidth = Math.min(COLUMN_MAX_WIDTH, pixelWidth)
              return assign({}, lookupCol, { pixelWidth })
            }

            return col
          })

          this.columns = [...tempCols]
        }
      },

      initializeColumnWidths (rowIndex, col, width) {
        // once we've initialized our column widths, we're done
        if (this.col_widths_initialized) { return }

        const minWidth = defaultTo(this.default_col_widths[col.name], COLUMN_MIN_WIDTH)
        let newWidth = Math.max(minWidth, width + 20) // make columns a little wider than they need to be
        newWidth = Math.min(newWidth, COLUMN_MAX_WIDTH)
        this.default_col_widths[col.name] = newWidth

        const isHeader = rowIndex === 'header'
        const isLastRow = rowIndex === this.rendered_row_count - 1

        if (isHeader || isLastRow) {
          const tempCols = this.columns.map((col) => {
            const pixelWidth = this.default_col_widths[col.name]
            return assign({}, col, { pixelWidth })
          })

          this.columns = [...tempCols]
          this.$nextTick(() => { this.col_widths_initialized = true })
        }
      },

      updateStyle (idSuffix, styleStr) {
        const headEl = document.head || document.getElementsByTagName('head')[0]
        let styleEl = document.getElementById(this.uid + '-' + idSuffix)

        // a style with this ID already exists; remove it
        if (styleEl) { headEl.removeChild(styleEl) }

        // no style string specified; we're done
        if (!isString(styleStr) || styleStr.length === 0) { return }

        // add style to the <head>
        styleEl = document.createElement('style')
        styleEl.type = 'text/css'
        styleEl.id = this.uid + '-' + idSuffix

        if (styleEl.styleSheet) {
          styleEl.styleSheet.cssText = styleStr
        } else {
          styleEl.appendChild(document.createTextNode(styleStr))
        }

        headEl.appendChild(styleEl)
      }
    }
  }
</script>

<style lang="scss">
  $border-color: #ddd;

  .vg-container {
    font-family: Arial, sans-serif;
    font-size: 13px;
    color: #333;
  }

  .vg-thead {
    box-shadow: inset 0 -1px 0 $border-color;
  }

  .vg-th,
  .vg-td {
    margin-top: -1px;
    margin-left: -1px;
    padding-top: 1px;
    border-color: $border-color;
  }

  .vg-th-inner,
  .vg-td-inner {
    padding: 4px 5px 4px;
  }

  /* misc */

  .flex-fill {
    flex: 1 1;
    min-width: 0; /* 1 */
    min-height: 0; /* 1 */
  }

  .b--light-moon-gray {
    border-color: $border-color;
  }

  .lh-1 {
    line-height: 1;
  }

  .cursor-resize-ew {
    cursor: ew-resize;
  }
</style>
