<!--
  - DashboardListMedium.vue
  - Copyright (c) 2020 james@firefly-iii.org
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
  <table class="table table-striped table-sm">
    <caption style="display:none;">{{ $t('firefly.transaction_table_description') }}</caption>
    <thead>
    <tr>
      <th class="text-left" scope="col">{{ $t('firefly.description') }}</th>
      <th scope="col">{{ $t('firefly.opposing_account') }}</th>
      <th class="text-right" scope="col">{{ $t('firefly.amount') }}</th>
    </tr>
    </thead>
    <tbody>
    <tr v-for="transaction in this.transactions">
      <td>
        <a :href="'transactions/show/' + transaction.id " :title="transaction.date">
          <span v-if="transaction.attributes.transactions.length > 1">{{ transaction.attributes.group_title }}</span>
          <span v-if="1===transaction.attributes.transactions.length">{{ transaction.attributes.transactions[0].description }}</span>
        </a>
      </td>
      <td>
                <span v-for="tr in transaction.attributes.transactions">
                    <a v-if="'withdrawal' === tr.type" :href="'accounts/show/' + tr.destination_id">{{ tr.destination_name }}</a>
                    <a v-if="'deposit' === tr.type" :href="'accounts/show/' + tr.source_id">{{ tr.source_name }}</a>
                    <a v-if="'transfer' === tr.type && parseInt(tr.source_id) === account_id" :href="'accounts/show/' + tr.destination_id">{{ tr.destination_name }}</a>
                    <a v-if="'transfer' === tr.type && parseInt(tr.destination_id) === account_id" :href="'accounts/show/' + tr.source_id">{{ tr.source_name }}</a>
                    <br/>
                </span>
      </td>
      <td style="text-align:right;">
                <span v-for="tr in transaction.attributes.transactions">
                     <span v-if="'withdrawal' === tr.type" class="text-danger">
                        {{ Intl.NumberFormat(locale, {style: 'currency', currency: tr.currency_code}).format(tr.amount * -1) }}<br>
                     </span>
                    <span v-if="'deposit' === tr.type" class="text-success">
                        {{ Intl.NumberFormat(locale, {style: 'currency', currency: tr.currency_code}).format(tr.amount) }}<br>
                     </span>
                    <span v-if="'transfer' === tr.type && parseInt(tr.source_id) === account_id" class="text-info">
                        {{ Intl.NumberFormat(locale, {style: 'currency', currency: tr.currency_code}).format(tr.amount * -1) }}<br>
                    </span>
                    <span v-if="'transfer' === tr.type && parseInt(tr.destination_id) === account_id" class="text-info">
                        {{ Intl.NumberFormat(locale, {style: 'currency', currency: tr.currency_code}).format(tr.amount) }}<br>
                    </span>
                </span>
      </td>
    </tr>
    </tbody>
  </table>
</template>

<script>
export default {
  name: "DashboardListMedium",
  data() {
    return {
      locale: 'en-US'
    }
  },
  created() {
    this.locale = localStorage.locale ?? 'en-US';
  },
  props: {
    transactions: {
      type: Array,
      default: function () {
        return [];
      }
    },
    account_id: {
      type: Number,
      default: function () {
        return 0;
      }
    },
  }
}
</script>

