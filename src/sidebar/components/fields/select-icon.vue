<template lang="pug">
.SelectIconField
  .label {{t(label)}}
  select-icon-input(
    :value="value"
    :opts="opts"
    :fill="optFill"
    @input="select")
</template>


<script>
import SelectIconInput from '../inputs/select-icon'

export default {
  components: {
    SelectIconInput
  },

  props: {
    value: String,
    label: String,
    optFill: String,
    opts: Array,
  },

  data() {
    return {}
  },

  methods: {
    select(option) {
      this.$emit('input', option)
    },
  },
}
</script>


<style lang="stylus">
@import '../../../styles/mixins'

.SelectIconField
  box(relative)
  padding: 2px 0
  margin: 0 12px 0 16px
  cursor: pointer
  &:hover
    > .label
      color: var(--label-fg-hover)
  &.-no-top-margin
    margin-top: 0

.SelectIconField > .label
  box(relative)
  text(s: rem(14))
  color: var(--label-fg)
  transition: color var(--d-fast)
</style>
