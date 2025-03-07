<!--
  - Create.vue
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
  <div>
    <alert :message="errorMessage" type="danger"/>
    <alert :message="successMessage" type="success"/>
    <form @submit="submitTransaction" autocomplete="off">
      <SplitPills :transactions="transactions" :count="transactions.length"/>
      <div class="tab-content">
        <SplitForm
            v-for="(transaction, index) in this.transactions"
            v-bind:key="index"
            :count="transactions.length"
            :custom-fields="customFields"
            :date="date"
            :destination-allowed-types="destinationAllowedTypes"
            :index="index"
            :source-allowed-types="sourceAllowedTypes"
            :submitted-transaction="submittedTransaction"
            :transaction="transaction"
            :transaction-type="transactionType"
            v-on:uploaded-attachments="uploadedAttachment($event)"
            v-on:selected-attachments="selectedAttachment($event)"
            v-on:set-marker-location="storeLocation($event)"
            v-on:set-account="storeAccountValue($event)"
            v-on:set-date="storeDate($event)"
            v-on:set-field="storeField($event)"
            v-on:remove-transaction="removeTransaction($event)"
        />
      </div>

      <div class="row">
        <!-- group title -->
        <div class="col-xl-6 col-lg-6 col-md-12 col-sm-12 col-xs-12">
          <div v-if="transactions.length > 1" class="card">
            <div class="card-body">
              <div class="row">
                <div class="col">
                  <TransactionGroupTitle v-model="this.groupTitle" :errors="this.groupTitleErrors" v-on:set-group-title="storeGroupTitle($event)"/>
                </div>
              </div>
            </div>
          </div>
        </div>
        <div class="col-xl-6 col-lg-6 col-md-12 col-sm-12 col-xs-12">
          <!-- buttons -->
          <div class="card card-primary">
            <div class="card-body">
              <div class="row">
                <div class="col">
                  <div class="text-xs d-none d-lg-block d-xl-block">
                    &nbsp;
                  </div>
                  <button type="button" class="btn btn-outline-primary btn-block" @click="addTransactionArray"><span class="far fa-clone"></span> {{
                      $t('firefly.add_another_split')
                    }}
                  </button>
                </div>
                <div class="col">
                  <div class="text-xs d-none d-lg-block d-xl-block">
                    &nbsp;
                  </div>
                  <button :disabled="!enableSubmit" class="btn btn-success btn-block" @click="submitTransaction">
                    <span v-if="enableSubmit"><span class="far fa-save"></span> {{ $t('firefly.store_transaction') }}</span>
                    <span v-if="!enableSubmit"><span class="fas fa-spinner fa-spin"></span></span>
                  </button>
                </div>
              </div>
              <div class="row">
                <div class="col">
                  &nbsp;
                </div>
                <div class="col">
                  <div class="form-check">
                    <input id="createAnother" v-model="createAnother" class="form-check-input" type="checkbox">
                    <label class="form-check-label" for="createAnother">
                      <span class="small">{{ $t('firefly.create_another') }}</span>
                    </label>
                  </div>
                  <div class="form-check">
                    <input id="resetFormAfter" v-model="resetFormAfter" :disabled="!createAnother" class="form-check-input" type="checkbox">
                    <label class="form-check-label" for="resetFormAfter">
                      <span class="small">{{ $t('firefly.reset_after') }}</span>
                    </label>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </form>
  </div>
</template>

<script>
import Alert from '../partials/Alert';
import SplitPills from "./SplitPills";
import TransactionGroupTitle from "./TransactionGroupTitle";
import SplitForm from "./SplitForm";
import {mapGetters, mapMutations} from "vuex";

export default {
  name: "Create",
  components: {
    SplitForm,
    Alert,
    SplitPills,
    TransactionGroupTitle,
  },
  /**
   * Grab some stuff from the API, add the first transaction.
   */
  created() {
    // set transaction type:
    let pathName = window.location.pathname;
    let parts = pathName.split('/');
    let type = parts[parts.length - 1];

    // set a basic date-time string:
    let date = new Date;
    this.date = [date.getFullYear(), ('0' + (date.getMonth() + 1)).slice(-2), ('0' + date.getDate()).slice(-2)].join('-') + 'T00:00';

    //console.log('Date is set to "' + this.date + '"');

    this.setTransactionType(type[0].toUpperCase() + type.substring(1));
    this.getExpectedSourceTypes();
    this.getAccountToTransaction();
    this.getCustomFields();
    this.addTransaction();
  },
  data() {
    return {
      // error or success message
      errorMessage: '',
      successMessage: '',

      // custom fields to show, useful for components:
      customFields: {},

      // states for the form (makes sense right)
      enableSubmit: true,
      createAnother: false,
      resetFormAfter: false,

      // things the process is done working on (3 phases):
      submittedTransaction: false,
      submittedLinks: false,
      submittedAttachments: -1, // -1 (no attachments), 0 = uploading, 1 = uploaded

      // transaction was actually submitted?
      inError: false,

      // number of uploaded attachments
      // its an object because we count per transaction journal (which can have multiple attachments)
      // and array doesn't work right.
      submittedAttCount: {},

      // errors in the group title:
      groupTitleErrors: [],

      // group ID + title once submitted:
      returnedGroupId: 0,
      returnedGroupTitle: '',

      // meta data for accounts
      accountToTransaction: {},
      allowedOpposingTypes: {},
      sourceAllowedTypes: ['Asset account', 'Loan', 'Debt', 'Mortgage', 'Revenue account'],
      destinationAllowedTypes: ['Asset account', 'Loan', 'Debt', 'Mortgage', 'Expense account'],

      // date not in the store because it was buggy
      date: ''
    }
  },
  computed: {
    /**
     * Grabbed from the store.
     */
    ...mapGetters('transactions/create', ['transactionType', 'transactions', 'groupTitle', 'defaultErrors']),
    ...mapGetters('root', ['listPageSize'])
  },
  watch: {
    submittedAttachments: function () {
      this.finaliseSubmission();
    }
  },
  methods: {
    /**
     * Store related mutators used by this component.
     */
    ...mapMutations('transactions/create',
                    [
                      'setGroupTitle',
                      'addTransaction',
                      'deleteTransaction',
                      'setTransactionError',
                      'setTransactionType',
                      'resetErrors',
                      'updateField',
                      'resetTransactions',
                    ]
    ),
    addTransactionArray: function (event) {
      event.preventDefault();
      this.addTransaction();
    },
    /**
     * Removes a split from the array.
     */
    removeTransaction: function (payload) {
      // console.log('Triggered to remove transaction ' + payload.index);
      this.$store.commit('transactions/create/deleteTransaction', payload);
    },
    submitData: function (url, data) {
      return axios.post(url, data);
    },
    handleSubmissionResponse: function (response) {
      // console.log('In handleSubmissionResponse()');
      // save some meta data:
      this.returnedGroupId = parseInt(response.data.data.id);
      this.returnedGroupTitle = null === response.data.data.attributes.group_title ? response.data.data.attributes.transactions[0].description : response.data.data.attributes.group_title;
      let journals = [];

      // save separate journal ID's (useful ahead in the process):
      let result = response.data.data.attributes.transactions
      for (let i in result) {
        if (result.hasOwnProperty(i) && /^0$|^[1-9]\d*$/.test(i) && i <= 4294967294) {
          journals.push(parseInt(result[i].transaction_journal_id));
        }
      }

      return Promise.resolve({journals: journals});
    },
    submitLinks: function (response, submission) {
      let promises = [];
      // for
      for (let i in response.journals) {
        if (response.journals.hasOwnProperty(i) && /^0$|^[1-9]\d*$/.test(i) && i <= 4294967294) {
          let journalId = response.journals[i];
          let links = submission.transactions[i].links;
          for (let ii in links) {
            if (links.hasOwnProperty(ii) && /^0$|^[1-9]\d*$/.test(ii) && ii <= 4294967294) {
              let currentLink = links[ii];
              if (0 === currentLink.outward_id) {
                currentLink.outward_id = journalId;
              }
              if (0 === currentLink.inward_id) {
                currentLink.inward_id = journalId;
              }
              promises.push(axios.post('./api/v1/transaction_links', currentLink));
            }
          }
        }
      }
      if (0 === promises.length) {
        return Promise.resolve({response: 'from submitLinks'});
      }
      return Promise.all(promises);
    },
    submitAttachments: function (response, submission) {
      let anyAttachments = false;
      for (let i in response.journals) {
        if (response.journals.hasOwnProperty(i) && /^0$|^[1-9]\d*$/.test(i) && i <= 4294967294) {
          let journalId = response.journals[i];
          let hasAttachments = submission.transactions[i].attachments;
          // console.log('Decided that ' + journalId);
          // console.log(hasAttachments);
          if (hasAttachments) {
            // console.log('upload!');
            this.updateField({index: i, field: 'transaction_journal_id', value: journalId});
            // set upload trigger?
            this.updateField({index: i, field: 'uploadTrigger', value: true});
            //this.transactions[i].uploadTrigger = true;
            anyAttachments = true;
          }
        }
      }

      if (true === anyAttachments) {
        this.submittedAttachments = 0;
      }

      return Promise.resolve({response: 'from submitAttachments'});
    },
    selectedAttachment: function (payload) {
      this.updateField({index: payload.index, field: 'attachments', value: true});
    },
    finaliseSubmission: function () {
      // console.log('finaliseSubmission');
      if (0 === this.submittedAttachments) {
        // console.log('submittedAttachments = ' + this.submittedAttachments);
        return;
      }
      // console.log('In finaliseSubmission');
      if (false === this.createAnother) {
        window.location.href = (window.previousURL ?? '/') + '?transaction_group_id=' + this.returnedGroupId + '&message=created';
        return;
      }
      //console.log('Is in error?');
      //console.log(this.inError);
      if (false === this.inError) {
        // show message:
        this.errorMessage = '';
        this.successMessage = this.$t('firefly.transaction_stored_link', {ID: this.returnedGroupId, title: this.returnedGroupTitle});
      }
      // console.log('here we are');
      // enable flags:
      this.enableSubmit = true;
      this.submittedTransaction = false;
      this.submittedAttachments = -1;

      // reset attachments + errors
      if (!this.resetFormAfter) {
        for (let i in this.transactions) {
          if (this.transactions.hasOwnProperty(i) && /^0$|^[1-9]\d*$/.test(i) && i <= 4294967294) {
            if (this.transactions.hasOwnProperty(i)) {
              //this.
              // console.log('Reset attachment #' + i);
              this.updateField({index: i, field: 'transaction_journal_id', value: 0});
              this.updateField({index: i, field: 'errors', value: this.defaultErrors})
            }
          }
        }
      }
      // reset the form:
      if (this.resetFormAfter) {
        this.resetTransactions();
        this.addTransaction();
      }
      return Promise.resolve({response: 'from finaliseSubmission'});
    },
    handleSubmissionError: function (error) {
      //console.log('in handleSubmissionError');
      // oh noes Firefly III has something to bitch about.
      this.enableSubmit = true;

      // but report an error because error:
      this.inError = true;
      this.parseErrors(error.response.data);
    },
    /**
     * Actually submit the transaction to Firefly III. This is a fairly complex beast of a thing because multiple things
     * need to happen in the right order.
     */
    submitTransaction: function (event) {
      event.preventDefault();
      // console.log('submitTransaction()');
      // disable the submit button:
      this.enableSubmit = false;

      // assume nothing breaks
      this.inError = false;

      // remove old warnings etc.
      this.successMessage = '';
      this.errorMessage = '';

      // convert the data so its ready to be submitted:
      const url = './api/v1/transactions';
      const data = this.convertData();

      this.submitData(url, data)
          .then(this.handleSubmissionResponse)
          .then(response => {
                  return Promise.all([this.submitLinks(response, data), this.submitAttachments(response, data)]);
                }
          )
          .then(this.finaliseSubmission)
          .catch(this.handleSubmissionError);

    },
    /**
     * When a attachment component is done uploading it ends up here. We create an object where we count how many
     * attachment components have reported back they're done uploading. Of course if they have nothing to upload
     * they will be pretty fast in reporting they're done.
     *
     * Once the number of components matches the number of splits we know all attachments have been uploaded.
     */
    uploadedAttachment: function (journalId) {
      this.submittedAttachments = 0;
      // console.log('Triggered uploadedAttachment(' + journalId + ')');
      let key = 'str' + journalId;
      this.submittedAttCount[key] = 1;
      let count = Object.keys(this.submittedAttCount).length;
      // console.log('Count is now ' + count);
      // console.log('Length is ' + this.transactions.length);
      if (count === this.transactions.length) {
        // console.log('Got them all!');
        // mark the attachments as stored:
        this.submittedAttachments = 1;
      }
    },
    /**
     * Responds to changed location.
     */
    storeLocation: function (payload) {
      let zoomLevel = payload.hasMarker ? payload.zoomLevel : null;
      let lat = payload.hasMarker ? payload.lat : null;
      let lng = payload.hasMarker ? payload.lng : null;
      this.updateField({index: payload.index, field: 'zoom_level', value: zoomLevel});
      this.updateField({index: payload.index, field: 'latitude', value: lat});
      this.updateField({index: payload.index, field: 'longitude', value: lng});
    },
    /**
     * Responds to changed account.
     */
    storeAccountValue: function (payload) {
      this.updateField({index: payload.index, field: payload.direction + '_account_id', value: payload.id});
      this.updateField({index: payload.index, field: payload.direction + '_account_type', value: payload.type});
      this.updateField({index: payload.index, field: payload.direction + '_account_name', value: payload.name});

      this.updateField({index: payload.index, field: payload.direction + '_account_currency_id', value: payload.currency_id});
      this.updateField({index: payload.index, field: payload.direction + '_account_currency_code', value: payload.currency_code});
      this.updateField({index: payload.index, field: payload.direction + '_account_currency_symbol', value: payload.currency_symbol});

      //this.calculateTransactionType(payload.index);
    },
    storeField: function (payload) {
      this.updateField(payload);
    },
    storeDate: function (payload) {
      this.date = payload.date;
    },
    storeGroupTitle: function (value) {
      // console.log('set group title: ' + value);
      this.setGroupTitle({groupTitle: value});
    },

    /**
     * Submit transaction links.
     */
    submitTransactionLinks(data, response) {
      //console.log('submitTransactionLinks()');
      let promises = [];
      let result = response.data.data.attributes.transactions;
      let total = 0;
      for (let i in data.transactions) {
        if (data.transactions.hasOwnProperty(i) && /^0$|^[1-9]\d*$/.test(i) && i <= 4294967294) {
          let submitted = data.transactions[i];
          if (result.hasOwnProperty(i)) {
            // found matching created transaction.
            let received = result[i];
            // grab ID from received, loop "submitted" transaction links
            for (let ii in submitted.links) {
              if (submitted.links.hasOwnProperty(ii) && /^0$|^[1-9]\d*$/.test(ii) && ii <= 4294967294) {
                let currentLink = submitted.links[ii];
                total++;
                if (0 === currentLink.outward_id) {
                  currentLink.outward_id = received.transaction_journal_id;
                }
                if (0 === currentLink.inward_id) {
                  currentLink.inward_id = received.transaction_journal_id;
                }
                // submit transaction link:
                promises.push(axios.post('./api/v1/transaction_links', currentLink).then(response => {
// See reference nr. 4
                }));
              }
            }
          }
        }
      }
      if (0 === total) {
        this.submittedLinks = true;
        return;
      }
      Promise.all(promises).then(function () {
        this.submittedLinks = true;
      });
    },
    parseErrors: function (errors) {
      for (let i in this.transactions) {
        if (this.transactions.hasOwnProperty(i)) {
          this.resetErrors({index: i});
        }
      }

      this.successMessage = '';
      this.errorMessage = this.$t('firefly.errors_submission');
      if (typeof errors.errors === 'undefined') {
        this.successMessage = '';
        this.errorMessage = errors.message;
      }

      let payload;
      let transactionIndex;
      let fieldName;

      // fairly basic way of exploding the error array.
      for (const key in errors.errors) {
        // console.log('Error index: "' + key + '"');
        if (errors.errors.hasOwnProperty(key)) {
          if (key === 'group_title') {
            this.groupTitleErrors = errors.errors[key];
            continue;
          }
          if (key !== 'group_title') {
            // lol dumbest way to explode "transactions.0.something" ever.
            transactionIndex = parseInt(key.split('.')[1]);

            fieldName = key.split('.')[2];

            // set error in this object thing.
            // console.log('The errors in key "' + key + '" are');
            // console.log(errors.errors[key]);
            switch (fieldName) {
              case 'amount':
              case 'description':
              case 'date':
              case 'tags':
                payload = {index: transactionIndex, field: fieldName, errors: errors.errors[key]};
                this.setTransactionError(payload);
                break;
              case 'budget_id':
                payload = {index: transactionIndex, field: 'budget', errors: errors.errors[key]};
                this.setTransactionError(payload);
                break;
              case 'bill_id':
                payload = {index: transactionIndex, field: 'bill', errors: errors.errors[key]};
                this.setTransactionError(payload);
                break;
              case 'piggy_bank_id':
                payload = {index: transactionIndex, field: 'piggy_bank', errors: errors.errors[key]};
                this.setTransactionError(payload);
                break;
              case 'category_name':
                payload = {index: transactionIndex, field: 'category', errors: errors.errors[key]};
                this.setTransactionError(payload);
                break;
              case 'source_name':
              case 'source_id':
                payload = {index: transactionIndex, field: 'source', errors: errors.errors[key]};
                this.setTransactionError(payload);
                break;
              case 'destination_name':
              case 'destination_id':
                payload = {index: transactionIndex, field: 'destination', errors: errors.errors[key]};
                this.setTransactionError(payload);
                break;
              case 'foreign_amount':
              case 'foreign_currency':
                payload = {index: transactionIndex, field: 'foreign_amount', errors: errors.errors[key]};
                this.setTransactionError(payload);
                break;
            }
          }
          // unique some things
          if (typeof this.transactions[transactionIndex] !== 'undefined') {
            //this.transactions[transactionIndex].errors.source = Array.from(new Set(this.transactions[transactionIndex].errors.source));
            //this.transactions[transactionIndex].errors.destination = Array.from(new Set(this.transactions[transactionIndex].errors.destination));
          }

        }
      }
    },

    /**
     *
     */
    convertData: function () {
      //console.log('now in convertData');
      let data = {
        'transactions': []
      };
      //console.log('Group title is: "' + this.groupTitle + '"');
      if (this.groupTitle.length > 0) {
        data.group_title = this.groupTitle;
        //console.log('1) data.group_title is now "'+data.group_title+'"');
      }

      for (let i in this.transactions) {
        if (this.transactions.hasOwnProperty(i) && /^0$|^[1-9]\d*$/.test(i) && i <= 4294967294) {
          data.transactions.push(this.convertSplit(i, this.transactions[i]));
        }
      }
      if (data.transactions.length > 1 && '' !== data.transactions[0].description && (null === data.group_title || '' === data.group_title)) {
        data.group_title = data.transactions[0].description;
        //console.log('2) data.group_title is now "'+data.group_title+'"');
      }

      // depending on the transaction type for this thing, we need to
      // make sure other splits match the data we submit.
      if (data.transactions.length > 1) {
        // console.log('This is a split!');
        data = this.synchronizeAccounts(data);
      }

      return data;
    },
    synchronizeAccounts: function (data) {
      // console.log('synchronizeAccounts: ' + this.transactionType);
      // make sure all splits have whatever is in split 0.
      // since its a transfer we can drop the name and use ID's only.
      for (let i in data.transactions) {
        if (data.transactions.hasOwnProperty(i) && /^0$|^[1-9]\d*$/.test(i) && i <= 4294967294) {
          // console.log('now at ' + i);

          // for transfers, overrule both the source and the destination:
          if ('transfer' === this.transactionType.toLowerCase()) {
            data.transactions[i].source_name = null;
            data.transactions[i].destination_name = null;
            if (i > 0) {
              data.transactions[i].source_id = data.transactions[0].source_id;
              data.transactions[i].destination_id = data.transactions[0].destination_id;
            }
          }
          // for deposits, overrule the destination and ignore the rest.
          if ('deposit' === this.transactionType.toLowerCase()) {
            data.transactions[i].destination_name = null;
            if (i > 0) {
              data.transactions[i].destination_id = data.transactions[0].destination_id;
            }
          }

          // for withdrawals, overrule the source and ignore the rest.
          if ('withdrawal' === this.transactionType.toLowerCase()) {
            data.transactions[i].source_name = null;
            if (i > 0) {
              data.transactions[i].source_id = data.transactions[0].source_id;
            }
          }
        }
      }
      return data;

    },

    /**
     *
     * @param key
     * @param array
     */
    convertSplit: function (key, array) {
      if ('' === array.destination_account_name) {
        array.destination_account_name = null;
      }
      if (0 === array.destination_account_id) {
        array.destination_account_name = null;
      }

      if ('' === array.source_account_name) {
        array.source_account_name = null;
      }
      if (0 === array.source_account_id) {
        array.source_account_id = null;
      }

      let currentSplit = {
        // basic
        description: array.description,
        date: this.date,
        type: this.transactionType.toLowerCase(),

        // account
        source_id: array.source_account_id ?? null,
        source_name: array.source_account_name ?? null,
        destination_id: array.destination_account_id ?? null,
        destination_name: array.destination_account_name ?? null,

        // amount:
        currency_id: array.currency_id,
        amount: array.amount,

        // meta data
        budget_id: array.budget_id,
        category_name: array.category,

        // optional date fields (6x):
        interest_date: array.interest_date,
        book_date: array.book_date,
        process_date: array.process_date,
        due_date: array.due_date,
        payment_date: array.payment_date,
        invoice_date: array.invoice_date,

        // other optional fields:
        internal_reference: array.internal_reference,
        external_url: array.external_url,
        notes: array.notes,
        external_id: array.external_id,

        // location:
        zoom_level: array.zoom_level,
        longitude: array.longitude,
        latitude: array.latitude,
        tags: [],

        // from thing:
        order: 0,
        reconciled: false,
        attachments: array.attachments,
      };

      if (0 !== array.tags.length) {
        for (let i in array.tags) {
          if (array.tags.hasOwnProperty(i) && /^0$|^[1-9]\d*$/.test(i) && i <= 4294967294) {
            // array.tags
            let current = array.tags[i];
            if (typeof current === 'object' && null !== current) {
              currentSplit.tags.push(current.text);
              // console.log('Add tag "' + current.text + '" from object.');
              continue;
            }
            if (typeof current === 'string') {
              currentSplit.tags.push(current);
              // console.log('Add tag "' + current + '" from string.');
              continue;
            }
            // console.log('Is neither.');
          }
        }
      }
      // console.log('Current split tags is now: ');
      // console.log(currentSplit.tags);

      // bills and piggy banks
      if (0 !== array.piggy_bank_id) {
        currentSplit.piggy_bank_id = array.piggy_bank_id;
      }
      if (0 !== array.bill_id) {
        currentSplit.bill_id = array.bill_id;
      }

      // foreign amount:
      if (0 !== array.foreign_currency_id && '' !== array.foreign_amount) {
        currentSplit.foreign_currency_id = array.foreign_currency_id;
      }
      if ('' !== array.foreign_amount) {
        currentSplit.foreign_amount = array.foreign_amount;
      }

      // do transaction type
      // let transactionType;
      // let firstSource;
      // let firstDestination;

      // get transaction type from first transaction
      //transactionType = this.transactionType ? this.transactionType.toLowerCase() : 'any';
      //console.log('Transaction type is now ' + transactionType);
      // if the transaction type is invalid, might just be that we can deduce it from
      // the presence of a source or destination account
      //firstSource = this.transactions[0].source_account_type;
      //firstDestination = this.transactions[0].destination_account_type;
      //console.log(this.transactions[0].source_account);
      //console.log(this.transactions[0].destination_account);
      //console.log('Type of first source is  ' + firstSource);
      //console.log('Type of first destination is  ' + firstDestination);

      // default to source:
      currentSplit.currency_id = array.source_account_currency_id;
      // if ('any' === transactionType && ['asset', 'Asset account', 'Loan', 'Debt', 'Mortgage'].includes(firstSource)) {
      //   transactionType = 'withdrawal';
      // }

      if ('Deposit' === this.transactionType) {
        //   transactionType = 'deposit';
        currentSplit.currency_id = array.destination_account_currency_id;
      }
      //currentSplit.type = transactionType;
      //console.log('Final type is ' + transactionType);

      let links = [];
      for (let i in array.links) {
        if (array.links.hasOwnProperty(i) && /^0$|^[1-9]\d*$/.test(i) && i <= 4294967294) {
          let current = array.links[i];
          let linkTypeParts = current.link_type_id.split('-');
          let inwardId = 'outward' === linkTypeParts[1] ? 0 : parseInt(current.transaction_journal_id);
          let outwardId = 'inward' === linkTypeParts[1] ? 0 : parseInt(current.transaction_journal_id);
          let newLink = {
            link_type_id: parseInt(linkTypeParts[0]),
            inward_id: inwardId,
            outward_id: outwardId,
          };
          links.push(newLink);
        }
      }
      currentSplit.links = links;
      if (null === currentSplit.source_id) {
        delete currentSplit.source_id;
      }
      if (null === currentSplit.source_name) {
        delete currentSplit.source_name;
      }
      if (null === currentSplit.destination_id) {
        delete currentSplit.destination_id;
      }
      if (null === currentSplit.destination_name) {
        delete currentSplit.destination_name;
      }

      // console.log('Current split is: ');
      // console.log(currentSplit);

      // return it.
      return currentSplit;
    },
    /**
     * Get API value.
     */
    getAllowedOpposingTypes: function () {
      axios.get('./api/v1/configuration/firefly.allowed_opposing_types')
          .then(response => {
            // console.log('opposing types things.');
            // console.log(response.data.data.value);
            this.allowedOpposingTypes = response.data.data.value;
          });
    },
    getExpectedSourceTypes: function () {
      axios.get('./api/v1/configuration/firefly.expected_source_types')
          .then(response => {
            //console.log('getExpectedSourceTypes.');
            this.sourceAllowedTypes = response.data.data.value.source[this.transactionType];
            this.destinationAllowedTypes = response.data.data.value.destination[this.transactionType];

            // console.log('sourceAllowedTypes');
            // console.log(this.sourceAllowedTypes);

            // console.log('Source allowed types for ' + this.transactionType + ' is: ');
            // console.log(this.sourceAllowedTypes);

            // console.log('Destination allowed types for ' + this.transactionType + ' is: ');
            // console.log(this.destinationAllowedTypes);

            //this.allowedOpposingTypes = response.data.data.value;
          });
    },
    /**
     * Get API value.
     */
    getAccountToTransaction: function () {
      axios.get('./api/v1/configuration/firefly.account_to_transaction')
          .then(response => {
            this.accountToTransaction = response.data.data.value;
          });
    },
    /**
     * This method grabs the users preferred custom transaction fields. It's used when configuring the
     * custom date selects that will be available. It could be something the component does by itself,
     * thereby separating concerns. This is on my list. If it changes to a per-component thing, then
     * it should be done via the create.js Vue store because multiple components are interested in the
     * user's custom transaction fields.
     */
    getCustomFields: function () {
      axios.get('./api/v1/preferences/transaction_journal_optional_fields').then(response => {
        this.customFields = response.data.data.attributes.data;
      });
    },
    setDestinationAllowedTypes: function (value) {
      // console.log('Create::setDestinationAllowedTypes');
      // console.log(value);
      if (0 === value.length) {
        this.destinationAllowedTypes = this.defaultDestinationAllowedTypes;
        //console.log('empty so back to defaults');
        return;
      }
      this.destinationAllowedTypes = value;
    },
    setSourceAllowedTypes(value) {
      // console.log('Create::setSourceAllowedTypes');
      // console.log(value);
      if (0 === value.length) {
        this.sourceAllowedTypes = this.defaultSourceAllowedTypes;
        // console.log('empty so back to defaults');
        return;
      }
      this.sourceAllowedTypes = value;
    }
  },

}
</script>

