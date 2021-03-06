<template>
  <div class="ef-mini-form">
    <div
      class="ef-mini-form__row"
      :style="`grid-template-columns: ${gridTemplateColumnsCalculated}`"
    >
      <EasyField
        v-for="(subfield, fieldIndex) in schemaLabels"
        :key="fieldIndex"
        class="ef-mini-form__sub-field"
        v-bind="subfield"
      />
    </div>
    <div
      class="ef-mini-form__row"
      v-for="(row, rowIndex) in cValue"
      :style="`grid-template-columns: ${gridTemplateColumnsCalculated}`"
      :key="rowIndex"
    >
      <EasyField
        v-for="(subfield, fieldIndex) in cSchema"
        :key="fieldIndex"
        class="ef-mini-form__sub-field"
        :rowIndex="rowIndex"
        :rowData="cValue[rowIndex]"
        :rowInput="params => setSubFieldValue({ id: params.id, value: params.value, rowIndex })"
        v-bind="subfield"
        :value="cValue[rowIndex][subfield.id]"
        @input="
          (val, origin) => setSubFieldValue({ id: subfield.id, value: val, rowIndex }, origin)
        "
        @keyup.native.delete="onDeleteKey(rowIndex, subfield.id)"
      />
    </div>
  </div>
</template>

<style lang="sass">
// $
@import '../../index.sass'

.ef-mini-form
  >.ef-mini-form__row
    display: grid
    justify-items: stretch
    align-items: center
    grid-gap: $sm
    margin-bottom: $sm
</style>

<script>
import merge from 'merge-anything'
import copy from 'copy-anything'
import { isNumber } from 'is-what'
import { getUsageDocs } from './sharedProps.js'

export default {
  name: 'EfMiniForm',
  inheritAttrs: false,
  desc: `${getUsageDocs('EfMiniForm')}

### description

With EfMiniForm you can pass a "schema" just like an EasyForm. The difference is that EfMiniForm is more like a (as the name says) "mini" form. 😃

The "schema" you specify is shown as a single row. New rows are added automatically on user input.`,
  props: {
    // prop categories: behavior content general model state style
    value: {
      category: 'model',
      type: Array,
      default: () => [],
    },
    // EF props:
    schema: {
      category: 'content',
      type: Array,
      desc:
        'This is the information on the columns you want to be shown. An array of objects just like an EasyForm.',
      default: () => [{ component: 'QInput' }],
      examples: [
        "[{label: 'Amount', id: 'amount', component: 'QInput', type: 'number'}, {label: 'Currency', id: 'curr', component: 'QSelect', options: [{label: 'USD', value: 'usd'}]}]",
      ],
    },
    attrsToPass: {
      category: 'content',
      type: Array,
      desc: `A list of prop (attribute) names to be passed on to each single EasyField generated in the mini form.

This is useful when you want to use Evaluated Props in the schema of the mine form but need information from the top level EasyForm.`,
      default: () => ['formData', 'formDataFlat', 'formId', 'mode', 'fieldInput', 'lang'],
      examples: [`['formData', 'mode', 'myCustomProp']`],
    },
    maxRows: {
      category: 'content',
      type: Number,
      desc: 'Allows to limit the max amount of rows.',
    },
    // props of which to inherit "description" etc. from EasyField:
    disable: { type: Boolean },
    readonly: { type: Boolean },
  },
  computed: {
    cValue: {
      get () {
        const { value, schema, disable, readonly, maxRows } = this
        const emptyRow = schema.reduce((carry, { id }) => ({ ...carry, [id]: undefined }), {})
        if (!disable && !readonly && (!isNumber(maxRows) || maxRows > value.length)) {
          return value.concat([emptyRow])
        }
        return value
      },
      set (val) {
        this.$emit('input', val)
      },
    },
    miniFormAttrsToPass () {
      const { attrsToPass, getPropOrAttrOrParentProp } = this
      return attrsToPass.reduce((carry, attrKey) => {
        carry[attrKey] = getPropOrAttrOrParentProp(attrKey)
        return carry
      }, {})
    },
    cSchema () {
      const { schema, disable, readonly, miniFormAttrsToPass } = this
      return schema.map(subfield => {
        return merge(miniFormAttrsToPass, { disable, readonly }, subfield, {
          label: '',
          subLabel: '',
        })
      })
    },
    schemaLabels () {
      const { schema, miniFormAttrsToPass } = this
      return schema.map(subfield => {
        return merge(miniFormAttrsToPass, subfield, { component: undefined })
      })
    },
    gridTemplateColumnsCalculated () {
      const { schema } = this
      return schema.reduce((total, field) => {
        const fr = Number(field.span)
        if (isNumber(fr)) return `${total} ${fr}fr`
        return `${total} ${field.span || '1fr'}`
      }, '')
    },
  },
  methods: {
    getPropOrAttrOrParentProp (propKey) {
      if (propKey in this) return this[propKey]
      if (propKey in this.$attrs) return this.$attrs[propKey]
      if (propKey in this.$parent) return this.$parent[propKey]
      return this.$parent.$parent[propKey]
    },
    deleteRow (rowIndex) {
      const { value } = this
      const allRows = copy(value)
      if (allRows[rowIndex] === undefined) return
      allRows.splice(rowIndex, 1)
      this.$emit('input', allRows)
    },
    setSubFieldValue ({ id, value: newValue, rowIndex }, origin) {
      // do not emit when the origin is from the default value initialisation
      if (origin === 'default') return
      const { value: oldValue } = this
      const allRows = copy(oldValue)
      if (allRows[rowIndex] === undefined) this.$set(allRows, rowIndex, {})
      allRows[rowIndex][id] = newValue
      this.$emit('input', allRows)
    },
    onDeleteKey (rowIndex, fieldId) {
      const { value, deleteRow } = this
      const allRows = value
      const row = allRows[rowIndex]
      if (!row) return
      if (Object.keys(row).every(key => row[key] === '' || row[key] === 0)) {
        deleteRow(rowIndex)
      }
    },
  },
}
</script>
