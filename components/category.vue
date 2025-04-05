<template>
  <Card :Title="pageTitle" :Icon="pageIcon">
    <template #actions>
      <div class="row justify-end">
        <div v-if="permissions.create" class="col-12 col-md-5">
          <q-btn class="full-width" label="Adicionar" icon="fas fa-plus" color="positive" @click="showModal = true">
            <q-tooltip>Adicionar nova categoria</q-tooltip>
          </q-btn>
        </div>
      </div>
    </template>
    <DataTable :Name="tableName" :DataURL="endpoint" v-model="Datatable" :Columns="columns" :RowActions="rowActions">
    </DataTable>

    <!-- Modal -->
    <Modal :Title="`${!!edditingItemKey ? '' : 'Nova '}Categoria de Transação`" Icon="fas fa-tags" Persistent
      v-model="showModal" :Actions="modalActions" @hide="resetInput" :HideActions="readonly">
      <!-- Content -->
      <InputField type="text" Label="Nome" Icon="fas fa-dollar" v-model="input.ds_title" :readonly="readonly"
        :Error="inputError.ds_title" @focus="inputError.ds_title = false">
      </InputField>
      <InputField type="textarea" Label="Descrição" Icon="fas fa-info-circle" v-model="input.tx_description"
        :readonly="readonly">
      </InputField>
      <!-- Audit Data -->
      <Auditinfo v-if="!!edditingItemKey && readonly" :input="audit"></Auditinfo>
    </Modal>
  </Card>
</template>
<script>
// Services:
import { auth, permissions } from 'src/modules/lambdatt-ui-iam/services.js'
import { ENDPOINTS } from 'src/services/endpoints'

// ------- Page Config:
const page_title = 'Categorias';
const page_icon = 'fas fa-tags'
const entity_endpoint = ENDPOINTS.FIN.CATEGORY;
const table_name = 'analytics-finances-transactions-category';
// -------

// Components:
import Auditinfo from 'src/modules/lambdatt-ui-iam/components/auditinfo.vue'

export default {
  name: 'components-analytics-finances-transactions-category',

  components: {
    Auditinfo,
  },

  data() {
    return {
      pageTitle: page_title,
      pageIcon: page_icon,
      input: {
        ds_title: null,
        tx_description: null,
      },
      inputError: {
        ds_title: null,
      },
      // Audit
      audit: {
        dtCreated: null,
        dtUpdated: null,
        userCreated: null,
        userUpdated: null,
      },
      // Datatable:
      Datatable: {},
      // Options
      readonly: false,
      // Modal
      showModal: false,
      edditingItemKey: null,
    }
  },

  computed: {
    permissions() {
      return {
        create: permissions.validatePermissions({ 'FIN_TRANSACTIONS_CATEGORY': 'C' }),
        update: permissions.validatePermissions({ 'FIN_TRANSACTIONS_CATEGORY': 'U' }),
        delete: permissions.validatePermissions({ 'FIN_TRANSACTIONS_CATEGORY': 'D' }),
      }
    },

    endpoint() {
      return entity_endpoint;
    },

    tableName() {
      return table_name;
    },

    columns() {
      return [
        { label: 'Título', field: 'ds_title', sort: 7, align: 'center' },
        { label: 'Descrição', field: 'tx_description', sort: 8, align: 'center' },
      ]
    },

    rowActions() {
      return [
        { icon: 'fas fa-eye', label: 'Visualizar', tooltip: 'Ver Detalhes', fn: (row) => { this.readonly = true; this.editItem(row) } },
        { icon: 'fas fa-edit', label: 'Editar', hide: !this.permissions.update || this.readonly, tooltip: 'Editar Informações', fn: this.editItem },
        { icon: 'fas fa-trash-alt', label: 'Excluir', hide: !this.permissions.delete || this.readonly, tooltip: 'Excluir', fn: this.remove },
      ]
    },

    modalActions() {
      return [
        {
          label: 'Excluir', color: 'negative', hide: !this.permissions.delete || this.edditingItemKey === null, icon: 'fas fa-trash-alt',
          fn: () => { this.showModal = false; this.remove({ ds_key: this.edditingItemKey }) }
        },
        { label: 'Salvar', color: 'green', icon: 'save', fn: this.save },
      ]
    },
  },

  methods: {
    editItem(item) {
      this.showModal = true;
      setTimeout(() => {
        this.edditingItemKey = item.ds_key;
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

    resetInput() {
      for (let k in this.input) {
        this.input[k] = null;
        if (k in this.inputError) this.inputError[k] = false;
      }
      for (let k in this.control) { this.control[k] = null; }
      this.readonly = false;
      this.edditingItemKey = null;
    },

    async save() {
      // Validation
      if (!this.$utils.validateForm(this.input, this.inputError)) { return false };

      // Emitting the loading event
      this.$emit('load', 'item-save');

      // Setting data for upload
      let data = new FormData();
      for (let item in this.input) {
        data.set(item, this.input[item]);
      }

      // Api Request
      try {
        if (!!this.edditingItemKey) {
          // UPDATE
          await this.$http.put(`${ENDPOINTS.FIN.CATEGORY}/${this.edditingItemKey}`, data);
          this.$utils.notify({
            message: 'A categoria foi atualizada com sucesso',
            type: 'positive',
            position: 'top-right'
          })
        } else {
          //CREATE
          await this.$http.post(ENDPOINTS.FIN.CATEGORY, data)
          this.$utils.notify({
            message: 'A categoria foi criada com sucesso',
            type: 'positive',
            position: 'top-right'
          });
        }
        await this.Datatable.reload();
        this.showModal = false;
      } catch (error) {
        this.$utils.notifyError(error);
        console.error('An error occurred while attempting to create/update the object.', error);
      } finally {
        // Finalizing the loading event
        this.$emit('loaded', 'item-save');
      }
    },

    async remove(data) {
      // Confimation
      if (!confirm('Deseja excluir as informações?')) return false;

      // Emitting the loading event
      this.$emit('load', 'item-remove');

      // Api Request
      try {
        await this.$http.delete(`${ENDPOINTS.FIN.CATEGORY}/${data.ds_key}`);
        this.$utils.notify({
          message: 'A categoria foi excluída com sucesso',
          type: 'positive',
          position: 'top-right'
        });
        this.Datatable.reload();
      } catch (error) {
        this.$utils.notifyError(error);
        console.error("An error occurred while attempting to delete the object.", error);
      } finally {
        // Finalizing the loading event
        this.$emit('loaded', 'item-remove');
      };
    },
  }
}
</script>