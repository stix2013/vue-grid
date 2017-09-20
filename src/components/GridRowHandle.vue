<template>
  <div class="absolute nowrap" :style="rowStyle">
    <div class="h-100 lh-1 light-silver tr vg-td-inner" :style="innerRowHandleStyle">{{rowIndex+1}}</div>
  </div>
</template>

<script>
  import {
    DEFAULT_ROW_HEIGHT,
    DEFAULT_ROW_HANDLE_WIDTH
  } from '../constants'

  export default {
    props: {
      rowIndex: {
        type: Number,
        required: true
      },
      rowHeight: {
        type: Number,
        default: DEFAULT_ROW_HEIGHT
      },
      rowHandleWidth: {
        type: Number,
        default: DEFAULT_ROW_HANDLE_WIDTH
      }
    },
    computed: {
      rowStyle () {
        return {
          'top': (this.rowIndex * this.rowHeight) + 'px',
          'height': (this.rowHeight + 1) + 'px'
        }
      },
      innerRowHandleStyle () {
        return {
          'width': this.rowHandleWidth + 'px'
        }
      }
    }
  }
</script>
