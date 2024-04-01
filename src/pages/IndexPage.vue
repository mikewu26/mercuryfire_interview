<template>
  <q-page class="row q-pt-xl">
    <div class="full-width q-px-xl">
      <form class="q-mb-xl">
        <div class="row inputs">
          <q-input v-model="formData.name" label="姓名" />
          <q-input v-model="formData.age" label="年齡" />
        </div>
        <div class="actions">
          <q-btn v-if="isEditing" @click="cancelEdit">取消編輯</q-btn>
          <q-btn type="submit" color="primary" @click="handleSubmit">{{
            isEditing ? '更新' : '新增'
          }}</q-btn>
        </div>
      </form>
      <q-table
        flat
        bordered
        ref="tableRef"
        :rows="tableData"
        :columns="(tableConfig as QTableProps['columns'])"
        row-key="id"
        hide-pagination
        separator="cell"
        :rows-per-page-options="[0]"
      >
        <template v-slot:header="props">
          <q-tr :props="props">
            <q-th v-for="col in props.cols" :key="col.name" :props="props">
              {{ col.label }}
            </q-th>
            <q-th></q-th>
          </q-tr>
        </template>

        <template v-slot:body="props">
          <q-tr :props="props">
            <q-td
              v-for="col in props.cols"
              :key="col.name"
              :props="props"
              style="min-width: 120px"
            >
              <div>{{ col.value }}</div>
            </q-td>
            <q-td class="text-right" auto-width v-if="tableButtons.length > 0">
              <q-btn
                @click="handleClickOption(btn, props.row)"
                v-for="(btn, index) in tableButtons"
                :key="index"
                size="sm"
                color="grey-6"
                round
                dense
                :icon="btn.icon"
                class="q-ml-md"
                padding="5px 5px"
              >
                <q-tooltip
                  transition-show="scale"
                  transition-hide="scale"
                  anchor="top middle"
                  self="bottom middle"
                  :offset="[10, 10]"
                >
                  {{ btn.label }}
                </q-tooltip>
              </q-btn>
            </q-td>
          </q-tr>
        </template>
        <template v-slot:no-data="{ icon }">
          <div
            class="full-width row flex-center items-center text-primary q-gutter-sm"
            style="font-size: 18px"
          >
            <q-icon size="2em" :name="icon" />
            <span> 無相關資料 </span>
          </div>
        </template>
      </q-table>
    </div>
  </q-page>
</template>

<script setup lang="ts">
import axios from 'axios';
import { QTableProps, useQuasar } from 'quasar';
import { onMounted, ref } from 'vue';
interface btnType {
  label: string;
  icon: string;
  status: string;
}
interface row {
  id: number;
  name: string;
  age: number;
}
const $q = useQuasar();
const isDeleteDialogOpen = ref(false);
const isEditing = ref(false);
const tableData = ref<row[]>([]);
const tableConfig = ref([
  {
    label: 'id',
    name: 'id',
    field: 'id',
    align: 'left',
    sortable: true,
    sortOrder: 'ad',
  },
  {
    label: '姓名',
    name: 'name',
    field: 'name',
    align: 'left',
    sortable: true,
  },
  {
    label: '年齡',
    name: 'age',
    field: 'age',
    align: 'left',
    sortable: true,
  },
]);
const tableButtons = ref([
  {
    label: '編輯',
    icon: 'edit',
    status: 'edit',
  },
  {
    label: '刪除',
    icon: 'delete',
    status: 'delete',
  },
]);
const rowIdClicked = ref();
const formData = ref({
  name: '',
  age: '',
});
function handleClickOption(btn: btnType, data: row) {
  const { status } = btn;
  const { id } = data;

  rowIdClicked.value = id;
  if (status === 'edit') {
    formData.value.name = data.name;
    formData.value.age = data.age.toString();
    isEditing.value = true;
  } else if (status === 'delete') {
    isDeleteDialogOpen.value = true;
    handleDeleteRow();
  } else {
    throw new Error('status not found');
  }
}
function cancelEdit() {
  isEditing.value = false;
  formData.value.name = '';
  formData.value.age = '';
}
function handleSubmit(e: Event) {
  e.preventDefault();
  // validate value
  if (!formData.value.name || !formData.value.age) {
    $q.notify({
      message: '請填寫完整資料',
      color: 'negative',
    });
    return;
  } else if (
    isNaN(Number(formData.value.age)) ||
    Number(formData.value.age) <= 0
  ) {
    $q.notify({
      message: '年齡請填數字且需大於 0',
      color: 'negative',
    });
    return;
  }
  if (isEditing.value) {
    editData({
      id: rowIdClicked.value,
      name: formData.value.name,
      age: Number(formData.value.age),
    });
  } else {
    createData({ name: formData.value.name, age: Number(formData.value.age) });
  }
}
function handleDeleteRow() {
  $q.dialog({
    title: '提示',
    message: `是否確定刪除編號 ${rowIdClicked.value.toString()} 的資料？`,
    cancel: {
      label: '取消',
      flat: true,
      color: 'black',
    },
    persistent: true,
    ok: {
      label: '確定',
      color: 'negative',
    },
  })
    .onOk(() => {
      deleteData(rowIdClicked.value).then(() => {
        isDeleteDialogOpen.value = false;
      });
    })
    .onCancel(() => {
      isDeleteDialogOpen.value = false;
    });
}
function readData() {
  axios.get('https://demo.mercuryfire.com.tw:49110/crudTest/a').then((res) => {
    tableData.value = res.data.result;
  });
}
function editData(data: row) {
  axios
    .patch('https://demo.mercuryfire.com.tw:49110/crudTest', data)
    .then(() => {
      tableData.value = tableData.value.map((item) => {
        if (item.id === data.id) {
          return data;
        }
        return item;
      });
      $q.notify({
        message: '編輯成功',
        color: 'positive',
      });
    });
}
function createData(data: Omit<row, 'id'>) {
  axios
    .post('https://demo.mercuryfire.com.tw:49110/crudTest', data)
    .then((res) => {
      tableData.value.push(res.data.result);
      $q.notify({
        message: '新增成功',
        color: 'positive',
      });
    });
}
function deleteData(id: number) {
  return axios
    .delete('https://demo.mercuryfire.com.tw:49110/crudTest/' + id)
    .then((res) => {
      $q.notify({
        message: '刪除成功',
        color: 'positive',
      });
      tableData.value = tableData.value.filter((item) => item.id !== id);
    });
}
onMounted(() => {
  readData();
});
</script>

<style lang="scss" scoped>
.q-table th {
  font-size: 20px;
  font-weight: bold;
}

.q-table tbody td {
  font-size: 18px;
}
form {
  max-width: 500px;
  border: 1px solid #e1e1e1;
  padding: 8px 16px;
  border-radius: 5px;
  .inputs {
    display: flex;
    justify-content: space-between;
    margin-bottom: 16px;
  }
  .actions {
    display: flex;
    justify-content: flex-end;
    align-items: center;
    gap: 8px;
  }
}
</style>
