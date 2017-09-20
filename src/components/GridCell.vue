<template>
  <div>
    <div class="h-100 lh-1 vg-td-inner" :class="cls" :style="'width: '+width+'px'">
      <span ref="content">{{value}}</span>
    </div>
  </div>
</template>
<script>
  import get from 'lodash/get'

  export default {
    props: {
      col: {
        type: Object,
        required: true
      },
      value: {
        required: true
      },
      width: {
        type: Number,
        required: true
      }
    },
    data () {
      return {
        contentWidth: 0
      }
    },
    computed: {
      cls () {
        switch (get(this.col, 'type', 'text')) {
          case 'bool':
          case 'boolean':
          case 'date':
          case 'datetime':
            return 'tc'
          case 'integer':
          case 'float':
          case 'number':
          case 'numeric':
            return 'tr'
        }

        return ''
      }
    },
    mounted () {
      const el = this.$refs.content
      this.contentWidth = el ? el.offsetWidth : 0
      this.$emit('determine-auto-width', this.col, this.contentWidth)
    }
  }
</script>
