<template>
  <q-page class="row q-pt-xl">
    <!-- Loading -->
    <q-inner-loading :showing="loading" color="primary">
      <q-spinner-facebook size="50px" color="primary" />
      <div class="text-primary q-mt-sm">Loading...</div>
    </q-inner-loading>
    <div class="full-width q-px-xl">
      <!-- 表單區域：用於新增和編輯資料 -->
      <q-form ref="formRef" @submit.prevent="handleSubmit" class="q-mb-xl">
        <!-- 姓名輸入框 -->
        <q-input
          v-model="tempData.name"
          label="姓名"
          :rules="nameRules"
          lazy-rules
        />
        <!-- 年齡輸入框 -->
        <q-input
          v-model.number="tempData.age"
          label="年齡"
          type="number"
          :rules="ageRules"
          lazy-rules
        />
        <!-- 提交按鈕：根據編輯狀態顯示不同文字 -->
        <q-btn
          :label="isEditing ? '更新' : '新增'"
          color="primary"
          class="q-mt-md"
          type="submit"
        />
        <!-- 取消按鈕：只ˇ在編輯模式顯示 -->
        <q-btn
          v-if="isEditing"
          label="取消"
          color="grey"
          class="q-mt-md q-ml-sm"
          @click="resetForm"
        />
      </q-form>

      <!-- 搜尋區域：用於過濾表格數據 -->
      <div class="row items-center q-mb-md">
        <q-input
          v-model="searchText"
          dense
          placeholder="搜尋姓名..."
          class="col-12 col-md-4"
          @update:model-value="handleSearch"
        >
          <template v-slot:append>
            <q-icon name="search" />
          </template>
        </q-input>
      </div>

      <!-- 表格區域：顯示所有數據 -->
      <q-table
        flat
        bordered
        ref="tableRef"
        :rows="blockData"
        :columns="(tableConfig as QTableProps['columns'])"
        row-key="id"
        hide-pagination
        separator="cell"
        :rows-per-page-options="[0]"
        :loading="loading"
      >
        <!-- 表格標題 -->
        <template v-slot:header="props">
          <q-tr :props="props">
            <q-th v-for="col in props.cols" :key="col.name" :props="props">
              {{ col.label }}
            </q-th>
            <q-th></q-th>
          </q-tr>
        </template>

        <!-- 表格內容 -->
        <template v-slot:body="props">
          <q-tr :props="props">
            <!-- 數據列 -->
            <q-td
              v-for="col in props.cols"
              :key="col.name"
              :props="props"
              style="min-width: 120px"
            >
              <div>{{ col.value }}</div>
            </q-td>
            <!-- 操作按鈕列 -->
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
                <!-- 按鈕提示 -->
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

        <!-- 無數據時 -->
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
import { QTableProps, useQuasar, QForm, QSpinnerFacebook } from 'quasar';
import { ref, onMounted } from 'vue';

const $q = useQuasar();
// 表單引用，用於表單驗證
const formRef = ref<QForm | null>(null);

const baseURL = 'https://dahua.metcfire.com.tw/api/CRUDTest';
const loading = ref(false);
const isEditing = ref(false);
const searchText = ref('');
// 原始數據備份，用於搜尋功能
const originalData = ref<DataType[]>([]);

interface btnType {
  label: string;
  icon: string;
  status: string;
}

interface DataType {
  id?: string;
  name: string;
  age: number | null;
}

// 表格數據
const blockData = ref<DataType[]>([]);

// 表格列配置
const tableConfig = ref([
  {
    label: '姓名',
    name: 'name',
    field: 'name',
    align: 'left',
  },
  {
    label: '年齡',
    name: 'age',
    field: 'age',
    align: 'left',
    sortable: true,
  },
]);

// 表格操作按鈕配置
const tableButtons = ref<btnType[]>([
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

// 暫存數據，用於表單操作
const tempData = ref<DataType>({
  name: '',
  age: null,
});

// 錯誤處理函數
const handleError = (error: any) => {
  console.error('API Error:', {
    message: error.message,
    response: error.response?.data,
    status: error.response?.status,
  });
  try {
    $q.notify({
      color: 'negative',
      message: '操作失敗: ' + (error.response?.data?.message || error.message),
      position: 'top',
    });
  } catch (notifyError) {
    // 如果 notify 失敗，使用原生 alert
    alert('操作失敗: ' + (error.response?.data?.message || error.message));
  }
};

// 獲取數據列表
const fetchData = async () => {
  try {
    loading.value = true;

    const { data } = await axios.get(`${baseURL}/a`);
    originalData.value = data;
    blockData.value = data;
  } catch (error) {
    handleError(error);
  } finally {
    loading.value = false;
  }
};

// 處理搜尋
const handleSearch = () => {
  if (!searchText.value) {
    // 如果搜尋框為空，顯示所有數據
    blockData.value = originalData.value;
    return;
  }

  // 過濾符合搜尋條件的數據
  blockData.value = originalData.value.filter(
    (item) => item.name.toLowerCase().includes(searchText.value.toLowerCase()) // 忽略大小寫
  );
};

// 表單驗證邏輯
const nameRules = [(val: string) => !!val || '姓名不得空白'];

const ageRules = [
  (val: number) => !!val || '年齡不得空白',
  (val: number) => val > 0 || '請輸入正整數',
  (val: number) => Number.isInteger(Number(val)) || '請輸入整數',
];

// 重置表單
const resetForm = () => {
  tempData.value = {
    name: '',
    age: null,
  };
  isEditing.value = false;
  if (formRef.value) {
    formRef.value.resetValidation();
  }
};

// 處理表單提交
const handleSubmit = async () => {
  if (!formRef.value) return;

  try {
    // 驗證表單
    const isValid = await formRef.value.validate();
    if (!isValid) return;

    loading.value = true;

    const isEdit = isEditing.value;

    if (isEdit) {
      // 更新現有數據
      await axios.patch(baseURL, {
        id: tempData.value.id,
        name: tempData.value.name,
        age: Number(tempData.value.age),
      });
    } else {
      // 新增數據
      await axios.post(baseURL, {
        name: tempData.value.name,
        age: Number(tempData.value.age),
      });
    }

    await fetchData();
    resetForm();
    $q.notify({
      color: 'positive',
      message: `${isEdit ? '更新' : '新增'}成功`,
      position: 'top',
    });
  } catch (error) {
    handleError(error);
  } finally {
    loading.value = false;
  }
};

// 處理刪除操作
const handleDelete = async (id: string) => {
  try {
    loading.value = true;
    await axios.delete(`${baseURL}/${id}`);
    await fetchData();
    $q.notify({
      color: 'positive',
      message: '刪除成功',
      position: 'top',
    });
  } catch (error) {
    handleError(error);
  } finally {
    loading.value = false;
  }
};

// 處理表格按鈕點擊
const handleClickOption = (btn: btnType, data: DataType) => {
  if (btn.status === 'edit') {
    // 進入編輯模式
    tempData.value = { ...data };
    isEditing.value = true;
  } else if (btn.status === 'delete' && data.id) {
    // 顯示刪除確認對話框
    try {
      $q.dialog({
        class: 'custom-dialog',
        title: '提示',
        message: '是否確定刪除該筆資料?',
        cancel: {
          label: '取消',
          flat: true,
          color: 'grey',
        },
        ok: {
          label: '確定',
          flat: true,
          color: 'grey-8',
        },
        persistent: true,
      }).onOk(async () => {
        await handleDelete(data.id as string);
      });
    } catch (error) {
      console.error('Dialog error:', error);
    }
  }
};

onMounted(() => {
  fetchData();
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
</style>
