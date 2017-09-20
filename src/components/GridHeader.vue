<template>
  <div class="relative nowrap">
    <grid-header-cell
      class="dib v-top overflow-hidden ba bg-near-white relative tc vg-th"
      v-for="(col, index) in columns"
      :key="index"
      :col="col"
      :value="getColumnName(col)"
      :width="col.pixelWidth || 0"
      :style="'height: '+(rowHeight+1)+'px'"
      @column-resizer-mousedown="onColumnResizerMousedown"
      @determine-auto-width="onHeaderCellDetermineWidth"
    ></grid-header-cell>
  </div>
</template>

<script>
  import {
    DEFAULT_ROW_HEIGHT
  } from '../constants'
  import get from 'lodash/get'
  import GridHeaderCell from './GridHeaderCell.vue'

  export default {
    props: {
      rowHeight: {
        type: Number,
        default: DEFAULT_ROW_HEIGHT
      },
      columns: {
        type: Array,
        required: true
      }
    },
    components: {
      GridHeaderCell
    },
    methods: {
      getColumnName (col) {
        return get(col, 'name', '')
      },
      onColumnResizerMousedown (col) {
        this.$emit('start-column-resize', col)
      },
      onHeaderCellDetermineWidth (col, width) {
        this.$emit('determine-cell-auto-width', 'header', col, width)
      }
    }
  }
</script>
