<template>
  <q-card>
    <q-card-section class="page-title">Resumo de Transações</q-card-section>
    <q-separator></q-separator>
    <q-card-section>
      <!-- Filter Controls -->
      <div class="row">
        <div class="col-12 col-md-6">
          <InputField type="daterange" Label="Período" v-model="params.daterange" :Default="fullMonth"></InputField>
        </div>
        <div class="col-12 col-md-6">
          <InputField type="select" Label="Categoria" Icon="fas fa-tags" v-model="params.id_fnc_category"
            :Options="categories" clearable></InputField>
        </div>
      </div>

      <q-separator></q-separator>

      <!-- Expenses/Revenue Counters -->
      <div class="row q-pa-sm flex-center text-bold">
        <q-card class="col q-pa-md q-mr-sm">
          <div class="row">
            <div class="col-auto row flex-center">
              <q-icon name="trending_up" color="positive" size="md"></q-icon>
            </div>
            <div class="col q-px-sm">
              <div class="text-positive">R$ {{ totalRevenue.number_format(2, ',', '.') }}</div>
              <div>Receitas no período</div>
            </div>
          </div>
        </q-card>
        <q-card class="col q-pa-md">
          <div class="row">
            <div class="col-auto row flex-center">
              <q-icon name="trending_down" color="negative" size="md"></q-icon>
            </div>
            <div class="col q-px-sm">
              <div class="text-negative">R$ {{ totalExpenses.number_format(2, ',', '.') }}</div>
              <div>Despesas no período</div>
            </div>
          </div>
        </q-card>
      </div>
    </q-card-section>
    <q-card-section>
      <La1Card :Title="pageTitle" :Icon="pageIcon">
        <!-- Card Actions -->
        <template #actions>
          <div class="row justify-end">
            <div v-if="permissions.create" class="col-12 col-md-5">
              <BtnDropdown Label="Adicionar" Icon="fas fa-plus" Color="positive" :AvailableActions="dropdownActions">
              </BtnDropdown>
            </div>
          </div>
        </template>

        <!-- Table -->
        <SimpleTable dense :Name="tableName" :Data="data.cashflow" v-model="Table" :Columns="columns"
          :RowActions="rowActions" @search="loadData" Printable
          :IntervalRule="(p, c, n) => c?.dt_transaction != n?.dt_transaction || !!c === false"
          :CustomResources="tableExtraResources">
          <template #cell-value="row">
            <div class="text-center">
              <span :class="`text-bold ${row.data.vl_transaction_value < 0 ? 'text-negative' : 'text-positive'}`">
                {{ row.data.transactionValue }}
              </span>
            </div>
          </template>

          <template #interval-row="row">
            <div class="text-right q-pa-xs text-grey-8" :colspan="columns.length + 1">
              <div v-if="!!row.data.previous">
                <div>
                  <span class="text-caption text-bold">Saldo em {{ row.data.previous.dtTransaction }}:</span>
                </div>
                <div>
                  <span style="font-size:1.3em;"
                    :class="`text-${valueClass(Number(data.balances[row.data.previous.dt_transaction]))}`">R$
                    {{
                      Number(data.balances[row.data.previous.dt_transaction]).number_format(2, ',', '.') }}</span>
                </div>
                <q-separator></q-separator>
              </div>
              <div v-else>
                <div>
                  <span class="text-caption text-bold">Saldo anterior:</span>
                </div>
                <div>
                  <span style="font-size:1.3em;" :class="`text-${valueClass(Number(data.previousBalance))}`">R$
                    {{
                      Number(data.previousBalance).number_format(2, ',', '.') }}</span>
                </div>
              </div>
            </div>
          </template>
        </SimpleTable>

        <!-- Dialog: Edit/Delete options for repeated transacations: -->
        <q-dialog backdrop-filter="blur(4px) contrast(40%)" v-model="actionModeModal">
          <q-card>
            <q-card-section class="row items-center q-pb-none">
              <div class="text-h6">Modo de {{ actionName }}</div>
              <q-space />
              <q-btn icon="close" color="primary" flat round dense v-close-popup />
            </q-card-section>

            <q-card-section>
              <p>Esta transação possui repetições, como deseja proceder?</p>
            </q-card-section>

            <q-card-section>
              <q-option-group v-model="actionMode" :options="actionModes" color="primary" />
            </q-card-section>

            <q-card-actions align="right">
              <q-btn flat label="Cancelar" @click="actionConfirm = false" color="primary" v-close-popup />
              <q-btn flat label="Confirmar" @click="actionConfirm = true" color="primary" v-close-popup />
            </q-card-actions>
          </q-card>
        </q-dialog>

        <!-- Modal: Transaction Form -->
        <La1Modal :Title="`${!!edditingItemKey ? 'Dados da ' : 'Nova '}Transação`" Icon="fas fa-exchange" Persistent
          v-model="showModal" @hide="resetInput" :Actions="modalActions" :HideActions="readonly">
          <!-- Content -->
          <div class="row">
            <div class="col q-pa-sm">
              <q-btn-toggle spread v-model="input.do_direction"
                :toggle-color="`${input.do_direction == 'I' ? 'positive' : 'negative'}`"
                :class="`text-${input.do_direction == 'I' ? 'positive' : 'negative'}`" :disable="readonly" :options="[
                  { label: 'Receita', value: 'I', icon: 'fas fa-plus' },
                  { label: 'Despesa', value: 'O', icon: 'fas fa-minus' },
                ]" />
            </div>
            <div class="col-12">
              <InputField type="text" Label="Valor" Icon="fas fa-dollar" Mask="#.##" FillMask="0" ReverseFillMask
                v-model="input.vl_transaction_value" :readonly="readonly" :Error="inputError.vl_transaction_value"
                @focus="inputError.vl_transaction_value = false">
              </InputField>
            </div>
            <div class="col-12">
              <InputField type="select" Label="Categoria" Icon="fas fa-tags" v-model="input.id_fnc_category"
                :readonly="readonly" :Options="categories">
              </InputField>
            </div>
            <div class="col-12">
              <InputField type="text" Label="Descrição" Icon="fas fa-circle-info" v-model="input.tx_description"
                :readonly="readonly" :Error="inputError.tx_description" @focus="inputError.tx_description = false">
              </InputField>
            </div>
            <div class="col-12">
              <InputField type="date" Label="Data" v-model="input.dt_transaction" :readonly="readonly"
                :Error="inputError.dt_transaction" @focus="inputError.dt_transaction = false">
              </InputField>
            </div>
            <div class="col-12 q-pa-sm">
              <q-toggle v-model="controls.repeat" :disable="readonly" label="Repetir Transação"
                @update:model-value="resetRepetition"></q-toggle>
            </div>
          </div>
          <q-separator></q-separator>
          <div class="row">
            <div class="col-12">
              <InputField :disable="!controls.repeat" type="select" Label="Frequência" Icon="fas fa-calendar-alt"
                v-model="input.do_periodicity" :Options="periodicity" :readonly="readonly" :Error="periodicityError"
                @focus="periodicityError = false">
              </InputField>
            </div>
            <div class="col-12 q-pa-sm">
              <q-toggle v-model="input.isRecurrence" :disable="readonly || !controls.repeat"
                label="Repetir Indefinidamente" false-value="N" true-value="Y"
                @update:model-value="input.nr_installments = null"></q-toggle>
            </div>
            <div class="col-12">
              <InputField :disable="!controls.repeat || input.isRecurrence == 'Y'" type="number" Label="Nº de Parcelas"
                Icon="fas fa-layer-group" v-model="input.nr_installments" :Error="installmentsError"
                @focus="installmentsError = false" :readonly="readonly">
              </InputField>
            </div>
          </div>
          <q-separator></q-separator>
          <!-- Audit Data -->
          <IamAuditInfo v-if="!!edditingItemKey && readonly" :input="audit"></IamAuditInfo>
        </La1Modal>
      </La1Card>
    </q-card-section>
  </q-card>
</template>
<script>
import ENDPOINTS from '../ENDPOINTS';

// ------- Page Config:
const page_title = 'Fluxo de Caixa';
const page_icon = 'fas fa-exchange'
const table_name = 'analytics-finances-transactions';

const create_message = 'A transação foi criada com sucesso';
const update_message = 'A transação foi atualizada com sucesso';
const delete_message = 'A transação foi excluída com sucesso';
// -------

export default {
  name: 'finances-components-transactions',

  props: {
    AccountId: String,
  },

  data() {
    return {
      // Configs:
      pageTitle: page_title,
      pageIcon: page_icon,
      tableName: table_name,

      // Filters:
      params: {
        daterange: {
          from: null,
          to: null
        },
        id_fnc_category: null
      },

      // Misc controls:
      showModal: false,
      edditingItemKey: null,
      controls: {
        repeat: false,
      },
      readonly: false,

      // Transaction Form-related vars:
      input: {
        do_direction: 'I',
        dt_transaction: null,
        id_fnc_category: null,
        vl_transaction_value: null,
        tx_description: null,
        nr_installments: null,
        do_periodicity: null,
        isRecurrence: 'N',
        refDate: null
      },
      inputError: {
        dt_transaction: false,
        vl_transaction_value: false,
        tx_description: false,
      },
      periodicityError: false,
      installmentsError: false,

      // Action mode:
      actionModeModal: false,
      actionMode: 'single',
      actionConfirm: null,
      actionName: null,

      // Audit
      audit: {
        dtCreated: null,
        dtUpdated: null,
        userCreated: null,
        userUpdated: null,
      },

      // Data and Table:
      data: {
        cashflow: [],
        balances: {}
      },
      Table: {},

      // Select options lists:
      categories: [],
      periodicity: [
        { label: "Diário", value: 'D' },
        { label: "Semanal", value: 'W' },
        { label: "Mensal", value: 'M' },
        { label: "Anual", value: 'Y' },
      ],
    }
  },

  computed: {
    actionModes() {
      let modes;
      if (this.actionName.toLowerCase() == 'exclusão') {
        modes = [
          { label: `Aplicar ${this.actionName.toLowerCase()} somente para esta transação`, value: 'single' },
          { label: `Aplicar ${this.actionName.toLowerCase()} a partir desta transação.`, value: 'partial' },
          { label: `Aplicar ${this.actionName.toLowerCase()} para todas as transações`, value: 'all' },
        ]
      } else {
        modes = [
          { label: `Aplicar ${this.actionName.toLowerCase()} somente para esta transação`, value: 'single' },
          { label: `Aplicar ${this.actionName.toLowerCase()} a partir desta transação.`, value: 'partial' },
        ]
      }
      return modes
    },

    permissions() {
      return {
        create: this.$getService('iam/permissions').validatePermissions({ 'FNC_TRANSACTION': 'C' }),
        update: this.$getService('iam/permissions').validatePermissions({ 'FNC_TRANSACTION': 'U' }),
        delete: this.$getService('iam/permissions').validatePermissions({ 'FNC_TRANSACTION': 'D' }),
      }
    },

    columns() {
      return [
        {
          label: 'Data', field: 'dtTransaction', sort: 5, align: 'center', filter:
            { type: 'daterange', field: 'dt_transaction' }
        },
        { label: 'Descrição', field: 'tx_description', sort: 12, align: 'center', filter: 'text' },
        {
          label: 'Categoria', field: 'categoryName', sort: 3, align: 'center', filter:
            { type: 'select', options: this.categories, field: 'id_fnc_category' }
        },
        { label: 'Valor', field: 'transactionValue', sort: 9, align: 'center', filter: 'text', name: 'value' },
      ]
    },

    rowActions() {
      return [
        { icon: 'fas fa-eye', label: 'Visualizar', tooltip: 'Ver Detalhes', fn: (row) => { this.readonly = true; this.editItem(row) } },
        { icon: 'fas fa-edit', label: 'Editar', hide: !this.permissions.update || this.readonly, tooltip: 'Editar Informações', fn: this.editItem },
        { icon: 'fas fa-trash-alt', label: 'Excluir', hide: !this.permissions.delete || this.readonly, tooltip: 'Excluir', fn: this.remove },
      ]
    },

    dropdownActions() {
      return [
        { label: "Nova transação", icon: 'fas fa-exchange', tooltip: 'Registrar uma transação', fn: () => { this.showModal = true } },
      ]
    },

    modalActions() {
      return [
        {
          label: 'Excluir', color: 'negative', hide: !this.permissions.delete || this.edditingItemKey === null, icon: 'fas fa-trash-alt',
          fn: () => { this.showModal = false; this.remove({ ds_key: this.edditingItemKey }) }
        },
        { label: 'Salvar', color: 'positive', icon: 'save', fn: this.save },
      ]
    },

    tableExtraResources() {
      return [
        { label: 'Atualizar lista', icon: 'fas fa-refresh', fn: () => this.loadData() }
      ]
    },

    totalExpenses() {
      let onlyexpenses = this.data.cashflow
        .filter(entry => entry.vl_transaction_value < 0);

      if (onlyexpenses.length == 0) return 0;

      return onlyexpenses
        .map(entry => Number(entry.vl_transaction_value))
        .reduce((acc, cur) => acc + cur);
    },

    totalRevenue() {
      let onlyrevenue = this.data.cashflow
        .filter(entry => entry.vl_transaction_value >= 0);

      if (onlyrevenue.length == 0) return 0;

      return onlyrevenue
        .map(entry => Number(entry.vl_transaction_value))
        .reduce((acc, cur) => acc + cur);
    },

    fullMonth() {
      const date = new Date();
      const firstDay = new Date(date.getFullYear(), date.getMonth(), 1);
      const lastDay = new Date(date.getFullYear(), date.getMonth() + 1, 0);

      return {
        from: this.$getService('toolcase/utils').dateFormat(firstDay, 'y-m-d'),
        to: this.$getService('toolcase/utils').dateFormat(lastDay, 'y-m-d')
      };

    },

    filters() {
      var filters = {
        dtStart: this.params.daterange.from,
        dtEnd: this.params.daterange.to,
      };

      if (!!this.params.id_fnc_category)
        filters.id_fnc_category = this.params.id_fnc_category;

      return filters
    }
  },

  watch: {
    filters: {
      handler() {
        this.loadData()
      },
      deep: true
    }
  },

  methods: {
    // ======= CORE FUNCTIONS
    async loadData() {
      this.Table.loadState(true);

      try {
        var response = await this.$getService('toolcase/http').get(ENDPOINTS.LEDGER, this.filters);
        this.data = response.data;
      } catch (error) {
        this.Table.setError(error);
        this.$getService('toolcase/utils').notifyError(error);
        console.error("An error occurred while attempting to retrieve the object's data.", error);
      } finally {
        this.Table.loadState(false);
      }
    },

    async getCategories() {
      try {
        const response = await this.$getService('toolcase/http').get(ENDPOINTS.CATEGORY)
        if (response && response.data) {
          this.categories = response.data.map((cat) => ({
            label: cat.ds_title,
            value: cat.id_fnc_category,
          }));
        }
      } catch (error) {
        this.$getService('toolcase/utils').notifyError(error);
        console.error("An error occurred while attempting to retrieve the object's data.", error);
      }
    },

    editItem(_item) {
      var item = { ..._item };

      this.showModal = true;
      setTimeout(() => {
        this.edditingItemKey = item.ds_key;
        this.input.refDate = item.dt_transaction;

        if (item.isRepetition == 'Y')
          this.controls.repeat = true;

        item.vl_transaction_value = item.vl_transaction_value.toFixed(2);
        // Read Input-related fields
        for (let key in this.input) {
          if (key in item) {
            this.input[key] = item[key];
          }
        }

        // Read Audit-related fields
        for (let key in this.audit) {
          if (key in item) {
            this.audit[key] = item[key];
          }
        }
      }, 100);
    },

    async save() {
      var mode;
      if (this.controls.repeat == true && !!this.edditingItemKey) {
        let confirm = await this.selectMode('Alteração');
        mode = this.actionMode;
        this.resetActionMode();

        if (confirm === false) return false;
      }
      // Validation
      if (!this.$getService('toolcase/utils').validateForm(this.input, this.inputError) |
        !this.validateRepetition()) {
        return false
      };

      // Emitting the loading event
      this.Table.loadState(true);

      // Api Request
      try {
        let type = this.input.isRecurrence == 'Y' ? 'R' : 'T';
        let data = {
          id_fnc_account: this.AccountId,
          ...this.input
        };
        delete data.isRecurrence;
        if (type == 'R') delete data.nr_installments;

        if (!!this.edditingItemKey) {
          // UPDATE
          await this.$getService('toolcase/http').put(`${ENDPOINTS.ENTRY}/${this.edditingItemKey}/${mode}`, data);
          this.$getService('toolcase/utils').notify({
            message: update_message,
            type: 'positive',
            position: 'top-right'
          })
        } else {
          delete data.refDate;
          //CREATE
          await this.$getService('toolcase/http').post(`${ENDPOINTS.ENTRY}/${type}`, data);
          this.$getService('toolcase/utils').notify({
            message: create_message,
            type: 'positive',
            position: 'top-right'
          });
        }

        this.loadData();
        this.showModal = false;
      } catch (error) {
        this.$getService('toolcase/utils').notifyError(error);
        console.error(error);
        this.Table.loadState(false);
      }
    },

    resetActionMode() {
      this.actionModeModal = false;
      this.actionMode = 'single';
      this.actionConfirm = null;
      this.actionName = null;
    },

    selectMode(actionName) {
      this.actionName = actionName;

      return new Promise((resolve) => {
        this.actionModeModal = true;

        let terminate = () => {
          setTimeout(() => {
            if (this.actionConfirm !== null)
              resolve(this.actionConfirm)
            else terminate()
          }, 100);
        }

        terminate()
      });
    },

    async remove(data) {
      var mode;
      // Confimation
      if (data.isRepetition == 'Y') {
        let confirm = await this.selectMode('Exclusão');
        mode = this.actionMode;
        this.resetActionMode();

        if (confirm === false) return false;
      } else {
        if (!confirm('Deseja excluir as informações?')) return false;
      }

      // Emitting the loading event
      this.Table.loadState(true);

      // Api Request
      try {
        await this.$getService('toolcase/http').delete(`${ENDPOINTS.ENTRY}/${data.ds_key}/${mode}/${data.dt_transaction}`);
        this.$getService('toolcase/utils').notify({
          message: delete_message,
          type: 'positive',
          position: 'top-right'
        });

        this.loadData();
      } catch (error) {
        this.$getService('toolcase/utils').notifyError(error);
        console.error("An error occurred while attempting to delete the object.", error);
        this.Table.loadState(false);
      }
    },

    // ======= AUX FUNCTIONS
    validateRepetition() {
      var isValid = true;

      if (this.controls.repeat) {
        if (!this.input.do_periodicity) {
          this.periodicityError = true;
          isValid = false;
        }

        if (!this.input.nr_installments && this.input.isRecurrence == 'N') {
          this.installmentsError = true;
          isValid = false;
        }
      }

      return isValid;
    },

    resetRepetition() {
      this.input.isRecurrence = 'N';
      this.input.nr_installments = null;
      this.input.do_periodicity = null;
      this.periodicityError = false;
      this.installmentsError = false;
    },

    resetInput() {
      this.edditingItemKey = null;
      this.readonly = false;

      this.input = {
        do_direction: 'I',
        dt_transaction: null,
        id_fnc_category: null,
        vl_transaction_value: null,
        tx_description: null,
        nr_installments: null,
        do_periodicity: null,
        isRecurrence: 'N',
        refDate: null
      }

      this.inputError = {
        dt_transaction: false,
        vl_transaction_value: false,
        tx_description: false,
      };

      this.controls.repeat = false;
      this.periodicityError = false;
      this.installmentsError = false;
    },

    valueClass(val) {
      if (val > 0) return 'positive';
      else if (val == 0) return 'grey-7';
      else return 'negative';
    }
  },

  async mounted() {
    await this.getCategories();
    await this.loadData();
  }
}
</script>
<style scoped>
.page-title {
  font-size: 28px;
  font-weight: 500;
}
</style>