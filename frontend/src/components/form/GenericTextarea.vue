<!--
  - GenericTextarea.vue
  - Copyright (c) 2021 james@firefly-iii.org
  -
  - This file is part of Firefly III (https://github.com/firefly-iii).
  -
  - This program is free software: you can redistribute it and/or modify
  - it under the terms of the GNU Affero General Public License as
  - published by the Free Software Foundation, either version 3 of the
  - License, or (at your option) any later version.
  -
  - This program is distributed in the hope that it will be useful,
  - but WITHOUT ANY WARRANTY; without even the implied warranty of
  - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
  - GNU Affero General Public License for more details.
  -
  - You should have received a copy of the GNU Affero General Public License
  - along with this program.  If not, see <https://www.gnu.org/licenses/>.
  -->

<template>
  <div class="form-group">
    <div class="text-xs d-none d-lg-block d-xl-block">
      {{ title }}
    </div>
    <div class="input-group">
      <textarea
          v-model="localValue"
          :class="errors.length > 0 ? 'form-control is-invalid' : 'form-control'"
          :placeholder="title"
          :disabled=disabled
          :name="fieldName"
      >{{ localValue }}</textarea>
    </div>
    <span v-if="errors.length > 0">
      <span v-for="error in errors" class="text-danger small">{{ error }}<br/></span>
    </span>
  </div>
</template>

<script>
export default {
name: "GenericTextarea",
  props: {
    title: {
      type: String,
      default: ''
    },
    disabled: {
      type: Boolean,
      default: false
    },
    value: {
      type: String,
      default: ''
    },
    fieldName: {
      type: String,
      default: ''
    },
    errors: {
      type: Array,
      default: function () {
        return [];
      }
    },
  },
  data() {
    return {
      localValue: this.value
    }
  },
  watch: {
    localValue: function (value) {
      this.$emit('set-field', {field: this.fieldName, value: value});
    },
    value: function(value) {
      this.localValue = value;
    }
  }
}
</script>

