<template>
  <La1Card :Title="pageTitle" :Icon="pageIcon">
    <template #actions>
      <div class="row justify-end">
        <div v-if="permissions.create" class="col-12 col-md-5">
          <q-btn class="full-width" label="Adicionar" icon="fas fa-plus" color="positive" @click="showModal = true">
            <q-tooltip>Adicionar nova categoria</q-tooltip>
          </q-btn>
        </div>
      </div>
    </template>
    <DataTable :Name="tableName" :DataURL="ENDPOINTS.CATEGORY" v-model="Datatable" :Columns="columns"
      :RowActions="rowActions">
    </DataTable>

    <!-- Modal -->
    <La1Modal :Title="`${!!edditingItemKey ? '' : 'Nova '}Categoria de Transação`" Icon="fas fa-tags" Persistent
      v-model="showModal" :Actions="modalActions" @hide="resetInput" :HideActions="readonly">
      <!-- Content -->
      <InputField type="text" Label="Nome" Icon="fas fa-dollar" v-model="input.ds_title" :readonly="readonly"
        :Error="inputError.ds_title" @focus="inputError.ds_title = false">
      </InputField>
      <InputField type="textarea" Label="Descrição" Icon="fas fa-info-circle" v-model="input.tx_description"
        :readonly="readonly">
      </InputField>
      <!-- Audit Data -->
      <IamAuditInfo v-if="!!edditingItemKey && readonly" :input="audit"></IamAuditInfo>
    </La1Modal>
  </La1Card>
</template>
<script>
import ENDPOINTS from '../ENDPOINTS';

// ------- Page Config:
const page_title = 'Categorias';
const page_icon = 'fas fa-tags'
const table_name = 'analytics-finances-transactions-category';
// -------

export default {
  name: 'finances-components-categories',

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

      ENDPOINTS
    }
  },

  computed: {
    permissions() {
      return {
        create: this.$getService('iam/permissions').validatePermissions({ 'FNC_CATEGORY': 'C' }),
        update: this.$getService('iam/permissions').validatePermissions({ 'FNC_CATEGORY': 'U' }),
        delete: this.$getService('iam/permissions').validatePermissions({ 'FNC_CATEGORY': 'D' }),
      }
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
      if (!this.$getService('toolcase/utils').validateForm(this.input, this.inputError)) { return false };

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
          await this.$getService('toolcase/http').put(`${ENDPOINTS.CATEGORY}/${this.edditingItemKey}`, data);
          this.$getService('toolcase/utils').notify({
            message: 'A categoria foi atualizada com sucesso',
            type: 'positive',
            position: 'top-right'
          })
        } else {
          //CREATE
          await this.$getService('toolcase/http').post(ENDPOINTS.CATEGORY, data)
          this.$getService('toolcase/utils').notify({
            message: 'A categoria foi criada com sucesso',
            type: 'positive',
            position: 'top-right'
          });
        }
        await this.Datatable.reload();
        this.showModal = false;
      } catch (error) {
        this.$getService('toolcase/utils').notifyError(error);
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
        await this.$getService('toolcase/http').delete(`${ENDPOINTS.CATEGORY}/${data.ds_key}`);
        this.$getService('toolcase/utils').notify({
          message: 'A categoria foi excluída com sucesso',
          type: 'positive',
          position: 'top-right'
        });
        this.Datatable.reload();
      } catch (error) {
        this.$getService('toolcase/utils').notifyError(error);
        console.error("An error occurred while attempting to delete the object.", error);
      } finally {
        // Finalizing the loading event
        this.$emit('loaded', 'item-remove');
      };
    },
  }
}
</script>