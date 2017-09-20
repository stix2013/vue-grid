<template>
  <div class="overflow-hidden ba absolute bg-near-white z-1 vg-th" :style="rowHandleStyle">
    <div class="h-100 lh-1 light-silver tr vg-th-inner" :style="innerRowHandleStyle"></div>

    <!-- row handle resize handle -->
    <div
      class="absolute top-0 bottom-0 right-0 cursor-resize-ew"
      :style="'width: '+columnResizeHandleWidth+'px'"
      @mousedown="onRowHandleResizerMousedown"
    ></div>
  </div>
</template>

<script>
  import {
    DEFAULT_ROW_HEIGHT,
    DEFAULT_ROW_HANDLE_WIDTH,
    COLUMN_RESIZE_HANDLE_WIDTH
  } from '../constants'

  export default {
    props: {
      'row-height': {
        type: Number,
        default: DEFAULT_ROW_HEIGHT
      },
      'row-handle-width': {
        type: Number,
        default: DEFAULT_ROW_HANDLE_WIDTH
      }
    },
    data () {
      return {
        columnResizeHandleWidth: COLUMN_RESIZE_HANDLE_WIDTH
      }
    },
    computed: {
      rowHandleStyle () {
        return {
          'height': (this.rowHeight + 1) + 'px'
        }
      },
      innerRowHandleStyle () {
        return {
          'width': this.rowHandleWidth + 'px'
        }
      }
    },
    methods: {
      onRowHandleResizerMousedown () {
        this.$emit('start-row-handle-resize')
      }
    }
  }
</script>
