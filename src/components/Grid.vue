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
          v-for="(row, index) in renderRows"
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
          v-for="(row, index) in renderRows"
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
      dataUrl: {
        type: String,
        default: ''
      },
      customHeaders: {
        type: Object,
        default: () => { return {} }
      },
      /* show empty cells on vertical scroll */
      liveScroll: {
        type: Boolean,
        default: false
      },
      /* remove off-screen cells from the DOM on horizontal scroll */
      virtualScroll: {
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
      renderedRowCount (val, oldVal) {
        this.$emit('rendered-row-count-change', val, oldVal)
      },
      cachedRowCount (val, oldVal) {
        this.$emit('cached-row-count-change', val, oldVal)
      },
      totalRowCount (val, oldVal) {
        this.$emit('total-row-count-change', val, oldVal)
      },
      firstVisibleRow (val, oldVal) {
        this.$emit('first-visible-row-change', val, oldVal)
      },
      lastVisibleRow (val, oldVal) {
        this.$emit('last-visible-row-change', val, oldVal)
      },
      visibleRowCount (val, oldVal) {
        this.$emit('visible-row-count-change', val, oldVal)
      }
      // firstVisibleColumn (val, oldVal) {
      //   console.log('firstVisibleColumn', val, oldVal)
      //   this.$emit('first-visible-column-change', val, oldVal)
      // },
      // lastVisibleColumn (val, oldVal) {
      //   this.$emit('last-visible-column-change', val, oldVal)
      // },
      // visibleColumnCount (val, oldVal) {
      //   this.$emit('visible-column-count-change', val, oldVal)
      // },
      // metrics (val, oldVal) {
      //   this.$emit('metrics-change', val, oldVal)
      // }
    },
    data () {
      return {
        uid: uniqueId('vue-grid-'),

        initialized: false,
        colWidthsInitialized: false,

        start: DEFAULT_START, // initial start
        limit: DEFAULT_LIMIT, // initial limit
        totalRowCount: 0, // returned by query

        columns: [],
        resizeCol: null,
        resizeRowHandle: null,

        rows: [],
        cachedRows: {},
        rowHeight: DEFAULT_ROW_HEIGHT,
        rowHandleWidth: DEFAULT_ROW_HANDLE_WIDTH,

        clientHeight: 0,
        clientWidth: 0,

        containerOffsetTop: 0,
        containerOffsetLeft: 0,

        offsetTop: 0,
        offsetLeft: 0,
        offsetHeight: 0,
        offsetWidth: 0,

        scrollTop: 0,
        scrollLeft: 0,

        mousedownX: -1,
        mousedownY: -1,
        mouseX: 0,
        mouseY: 0,

        isHorizontalScrollActive: false,
        currentFirstColumn: null,
        currentLastVisibleColumn: null,
        visibleColumnCount: 0,
        currentMetrics: {}
      }
    },
    computed: {
      totalHeight () {
        return this.rowHeight * this.totalRowCount
      },
      totalWidth () {
        return this.columns.reduce((acc, column) => acc + column.pixelWidth, 0)
      },
      renderedRowCount () {
        return size(this.rows)
      },
      cachedRowCount () {
        return size(this.cachedRows)
      },
      firstVisibleRow () {
        return Math.floor(this.scrollTop / this.rowHeight)
      },
      lastVisibleRow () {
        return Math.min(Math.ceil((this.scrollTop + this.clientHeight) / this.rowHeight), this.totalRowCount)
      },
      visibleRowCount () {
        return this.lastVisibleRow - this.firstVisibleRow
      },
      rowsInCache () {
        let missingRows = false

        if (this.cachedRowCount === 0) { return false }

        for (let i = this.start; i < this.start + this.limit; ++i) {
          if (i < this.totalRowCount && !this.cachedRows[i]) {
            missingRows = true
            break
          }
        }

        return !missingRows
      },
      renderRows () {
        if (this.liveScroll) { return this.rows }

        let rows = []
        for (let i = this.start; i < this.start + this.limit; ++i) {
          if (i < this.totalRowCount) { rows.push(this.cachedRows[i] || {}) }
        }
        return rows
      },
      leftOfRenderCols () {
        if (!this.virtualScroll) { return [] }

        // this value should be an empty array since we're showing
        // all columns when scrolling horizontally
        if (this.isPotentialHorizontalScroll) { return [] }

        let left = (-1 * this.scrollLeft)
        return this.columns.filter(c => {
          const isOffscreenLeft = left + c.pixelWidth + 1 < 0
          left += c.pixelWidth + 1
          return isOffscreenLeft
        })
      },
      leftOfRenderColsWidth () {
        console.log('leftOfRenderColsWidth', this.leftOfRenderCols)
        this.leftOfRenderCols.reduce((sum, column) => sum + column.pixelWidth, 0)
      },
      renderCols () {
        if (!this.virtualScroll) { return this.columns }

        // horizontal scroll operations are far more performance with
        // all columns visible since this is a normal browser scroll event
        // and there are no expensive calculations that need to happen
        if (this.isPotentialHorizontalScroll) { return this.columns }

        // if we haven't yet initialized our column widths,
        // we need to return all columns so that the cells
        // are rendered and the cell contents can be measured
        if (!this.colWidthsInitialized) { return this.columns }

        let left = (-1 * this.scrollLeft)
        return this.columns.filter(c => {
          const isOffscreenLeft = left + c.pixelWidth + 1 < 0
          const isOffscreenRight = left > this.clientWidth
          left += c.pixelWidth + 1
          return !isOffscreenLeft && !isOffscreenRight
        })
      },
      firstVisibleColumn () {
        // return first(this.renderCols)
        if (!this.renderCols || !Array.isArray(this.renderCols)) {
          return null
        }
        const firstColumn = first(this.renderCols)
        this.$emit('first-visible-column-change', firstColumn, this.currentFirstColumn)
        this.currentFirstColumn = firstColumn
        return firstColumn
      },
      lastVisibleColumn () {
        // return last(this.renderCols)
        const lastVisibleColumn = last(this.renderCols)
        this.$emit('last-visible-column-change', lastVisibleColumn, this.currentLastVisibleColumn)
        this.currentLastVisibleColumn = lastVisibleColumn
        return lastVisibleColumn
      },
      visibleColumnCount () {
        // return size(this.renderCols)
        const visibleColumnCount = size(this.renderCols)
        this.$emit('visible-column-count-change', visibleColumnCount, this.currentVisibleColumnCount)
        this.currentVisibleColumnCount = visibleColumnCount
        return visibleColumnCount
      },
      totalColumnCount () {
        return size(this.columns)
      },
      resizeDelta () {
        return this.mousedownX === -1 ? 0 : this.mouseX - this.mousedownX
      },
      verticalScrollbarLeft () {
        return this.offsetLeft + this.clientWidth
      },
      verticalScrollbarRight () {
        return this.offsetLeft + this.offsetWidth
      },
      horizontalScrollbarTop () {
        return this.offsetTop + this.clientHeight
      },
      horizontalScrollbarBottom () {
        return this.offsetTop + this.offsetHeight
      },
      isPotentialHorizontalScroll () {
        if (this.isHorizontalScrollActive) { return true }

        // handle bottom-right corner: if we're closer to the vertical
        // scrollbar than we are to the horizontal scrollbar, return 'false'
        if (this.mouseX / this.verticalScrollbarLeft >
            this.mouseY / this.horizontalScrollbarTop) { return false }

        // for the rest of the viewport, return 'true' when we get close
        // to the horizontal scrollbar
        if (this.mouseY >= this.horizontalScrollbarTop * 2 / 3 &&
            this.mouseY < this.horizontalScrollbarBottom) {
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
        const metrics = assign({}, filteredData, computedData)
        this.$emit('metrics-change', metrics, this.currentMetrics)
        this.currentMetrics = metrics
        return assign({}, filteredData, computedData)
      }
    },
    mounted () {
      this.activeXhr = null
      this.cancelXhr = null
      this.defaultColWidths = {}

      // initialize container offset
      const offset = getOffset(this.$refs['container'])
      this.containerOffsetTop = offset.top
      this.containerOffsetLeft = offset.left

      // initialize the client dimensions
      this.clientHeight = this.$el.clientHeight
      this.clientWidth = this.$el.clientWidth

      // establish our debounced and throttled methods (we need these functions
      // to be members of this component since we've run into reference
      // issues using debounce() and throttle() directly on the method
      this.tryFetchDebounced = debounce(this.tryFetch, 50, { leading: false, trailing: true })
      console.log('[COMP] Grid.vue::mounted resizeRowHandle', this.resizeRowHandle)
      this.resizeRowHandleThrottled = throttle(this.resizeRowHandleFunc, 20)
      this.resizeColumnThrottled = throttle(this.resizeColumnHandleFunc, 20)
      // this.scrollVerticalThrottled = debounce(this.scrollVertical, 5, { leading: false, trailing: true })
      this.scrollVerticalThrottled = throttle(this.scrollVerticalHandleFunc, 5, { leading: false, trailing: true })

      // do our initial fetch
      this.tryFetch()

      // document-level mouse down
      this.onDocumentMousedown = (evt) => {
        this.mousedownX = evt.pageX
        this.mousedownY = evt.pageY
      }

      // document-level mouse up
      this.onDocumentMouseup = (evt) => {
        this.mousedownX = -1
        this.mousedownY = -1
        this.resizeCol = null
        this.resizeRowHandle = null
        this.isHorizontalScrollActive = false
        this.updateStyle('cursor', '')
        this.updateStyle('noselect', '')
      }

      // document-level mouse move
      this.onDocumentMousemove = (evt) => {
        this.mouseX = evt.pageX
        this.mouseY = evt.pageY

        if (!isNil(this.resizeRowHandle)) { this.resizeRowHandleThrottled() }

        if (!isNil(this.resizeCol)) { this.resizeColumnThrottled() }
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
        if (!isNil(this.activeXhr) && !isNil(this.cancelXhr)) {
          this.cancelXhr()

          // reset XHR variables
          this.activeXhr = null
          this.cancelXhr = null
        }

        // if all of the rows exist in our row cache, we're done
        if (this.rowsInCache) { return }

        const CancelToken = axios.CancelToken
        this.activeXhr = axios.get(fetchUrl, {
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
          const cacheRows = Object.assign(this.cachedRows, tempCachedRows)
          this.cachedRows = assign({}, cacheRows)

          // set our init flag to true so we don't get columns after this
          this.initialized = true

          // reset XHR variables
          this.activeXhr = null
          this.cancelXhr = null
        })
      },

      onStartRowHandleResize (col) {
        this.resizeRowHandle = { oldWidth: this.rowHandleWidth }
        this.updateStyle('cursor', 'html, body { cursor: ew-resize !important; }')
        this.updateStyle('noselect', 'html, body { -moz-user-select: none !important; user-select: none !important }')
      },

      onStartColumnResize (col) {
        this.resizeCol = cloneDeep(col)
        this.updateStyle('cursor', 'html, body { cursor: ew-resize !important; }')
        this.updateStyle('noselect', 'html, body { -moz-user-select: none !important; user-select: none !important }')
      },

      onResize (resizeEl) {
        this.offsetTop = this.containerOffsetTop + resizeEl.offsetTop
        this.offsetLeft = this.containerOffsetLeft + resizeEl.offsetLeft
        this.offsetHeight = resizeEl.offsetHeight
        this.offsetWidth = resizeEl.offsetWidth
        this.clientHeight = resizeEl.clientHeight
        this.clientWidth = resizeEl.clientWidth

        this.$nextTick(() => {
          // check to see if all of the visible rows have been queried for
          if (this.lastVisibleRow >= this.start + this.renderedRowCount) {
            this.start = this.firstVisibleRow
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

      scrollVerticalHandleFunc (val, oldVal) {
        this.scrollTop = val

        // scrolling down
        if (this.lastVisibleRow >= this.start + this.renderedRowCount) {
          this.start = this.firstVisibleRow
          this.tryFetchDebounced()
        }

        // scrolling up
        if (this.firstVisibleRow < this.start) {
          this.start = this.firstVisibleRow
          this.tryFetchDebounced()
        }
      },

      scrollHorizontal (val, oldVal) {
        this.isHorizontalScrollActive = true
        this.scrollLeft = val
      },

      resizeRowHandleFunc () {
        const oldWidth = this.resizeRowHandle.oldWidth
        let newWidth = oldWidth + this.resizeDelta
        newWidth = Math.max(ROW_HANDLE_MIN_WIDTH, newWidth)
        newWidth = Math.min(ROW_HANDLE_MAX_WIDTH, newWidth)
        this.rowHandleWidth = newWidth
      },

      resizeColumnHandleFunc () {
        var lookupCol = find(this.columns, { name: get(this.resizeCol, 'name') })
        if (!isNil(lookupCol)) {
          const tempCols = this.columns.map((col) => {
            if (get(col, 'name') === get(lookupCol, 'name')) {
              const oldWidth = get(this.resizeCol, 'pixelWidth', DEFAULT_COLUMN_WIDTH)
              let pixelWidth = oldWidth + this.resizeDelta
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
        if (this.colWidthsInitialized) { return }

        const minWidth = defaultTo(this.defaultColWidths[col.name], COLUMN_MIN_WIDTH)
        let newWidth = Math.max(minWidth, width + 20) // make columns a little wider than they need to be
        newWidth = Math.min(newWidth, COLUMN_MAX_WIDTH)
        this.defaultColWidths[col.name] = newWidth

        const isHeader = rowIndex === 'header'
        const isLastRow = rowIndex === this.renderedRowCount - 1

        if (isHeader || isLastRow) {
          const tempCols = this.columns.map((col) => {
            const pixelWidth = this.defaultColWidths[col.name]
            return assign({}, col, { pixelWidth })
          })

          this.columns = [...tempCols]
          this.$nextTick(() => { this.colWidthsInitialized = true })
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
