<script setup lang="ts">
import { logApi } from "@/api/logs";
import type { LogFilter, RequestLog } from "@/types/models";
import { maskKey } from "@/utils/display";
import { EyeOffOutline, EyeOutline, Search } from "@vicons/ionicons5";
import {
  NButton,
  NDataTable,
  NDatePicker,
  NEllipsis,
  NIcon,
  NInput,
  NInputGroup,
  NSelect,
  NSpace,
  NSpin,
  NTag,
} from "naive-ui";
import { computed, h, onMounted, reactive, ref, watch } from "vue";

interface LogRow extends RequestLog {
  is_key_visible: boolean;
}

// Data
const loading = ref(false);
const logs = ref<LogRow[]>([]);
const currentPage = ref(1);
const pageSize = ref(15);
const total = ref(0);
const totalPages = computed(() => Math.ceil(total.value / pageSize.value));

// Filters
const filters = reactive({
  group_name: "",
  key_value: "",
  is_success: "" as "true" | "false" | "",
  status_code: "",
  source_ip: "",
  error_contains: "",
  start_time: null as number | null,
  end_time: null as number | null,
});

const successOptions = [
  { label: "全部状态", value: "" },
  { label: "成功", value: "true" },
  { label: "失败", value: "false" },
];

// Fetch data
const loadLogs = async () => {
  loading.value = true;
  try {
    const params: LogFilter = {
      page: currentPage.value,
      page_size: pageSize.value,
      group_name: filters.group_name || undefined,
      key_value: filters.key_value || undefined,
      is_success: filters.is_success === "" ? undefined : filters.is_success === "true",
      status_code: filters.status_code ? parseInt(filters.status_code, 10) : undefined,
      source_ip: filters.source_ip || undefined,
      error_contains: filters.error_contains || undefined,
      start_time: filters.start_time ? new Date(filters.start_time).toISOString() : undefined,
      end_time: filters.end_time ? new Date(filters.end_time).toISOString() : undefined,
    };

    const res = await logApi.getLogs(params);
    if (res.code === 0 && res.data) {
      logs.value = res.data.items.map(log => ({ ...log, is_key_visible: false }));
      total.value = res.data.pagination.total_items;
    } else {
      logs.value = [];
      total.value = 0;
      window.$message.error(res.message || "加载日志失败", {
        keepAliveOnHover: true,
        duration: 5000,
        closable: true,
      });
    }
  } catch (_error) {
    window.$message.error("加载日志请求失败");
  } finally {
    loading.value = false;
  }
};

const formatDateTime = (timestamp: string) => {
  if (!timestamp) {
    return "-";
  }
  const date = new Date(timestamp);
  return date.toLocaleString("zh-CN", { hour12: false }).replace(/\//g, "-");
};

const toggleKeyVisibility = (row: LogRow) => {
  row.is_key_visible = !row.is_key_visible;
};

// Columns definition
const createColumns = () => [
  {
    title: "时间",
    key: "timestamp",
    width: 180,
    render: (row: LogRow) => formatDateTime(row.timestamp),
  },
  {
    title: "状态",
    key: "is_success",
    width: 80,
    render: (row: LogRow) =>
      h(
        NTag,
        { type: row.is_success ? "success" : "error", size: "small", round: true },
        { default: () => (row.is_success ? "成功" : "失败") }
      ),
  },
  {
    title: "类型",
    key: "is_stream",
    width: 80,
    render: (row: LogRow) =>
      h(
        NTag,
        { type: row.is_stream ? "info" : "default", size: "small", round: true },
        { default: () => (row.is_stream ? "流式" : "非流") }
      ),
  },
  { title: "状态码", key: "status_code", width: 80 },
  { title: "耗时(ms)", key: "duration_ms", width: 100 },
  { title: "重试", key: "retries", width: 60 },
  { title: "分组", key: "group_name", width: 120 },
  {
    title: "Key",
    key: "key_value",
    width: 200,
    render: (row: LogRow) =>
      h(NSpace, { align: "center", wrap: false }, () => [
        h(
          NEllipsis,
          { style: "max-width: 150px" },
          { default: () => (row.is_key_visible ? row.key_value : maskKey(row.key_value || "")) }
        ),
        h(
          NButton,
          { size: "tiny", text: true, onClick: () => toggleKeyVisibility(row) },
          {
            icon: () =>
              h(NIcon, null, { default: () => h(row.is_key_visible ? EyeOffOutline : EyeOutline) }),
          }
        ),
      ]),
  },
  {
    title: "请求路径",
    key: "request_path",
    render: (row: LogRow) =>
      h(NEllipsis, { style: "max-width: 200px" }, { default: () => row.request_path }),
  },
  {
    title: "上游地址",
    key: "upstream_addr",
    render: (row: LogRow) =>
      h(NEllipsis, { style: "max-width: 200px" }, { default: () => row.upstream_addr }),
  },
  { title: "源IP", key: "source_ip", width: 130 },
  {
    title: "错误信息",
    key: "error_message",
    render: (row: LogRow) =>
      h(NEllipsis, { style: "max-width: 250px" }, { default: () => row.error_message || "-" }),
  },
  {
    title: "User Agent",
    key: "user_agent",
    render: (row: LogRow) =>
      h(NEllipsis, { style: "max-width: 200px" }, { default: () => row.user_agent }),
  },
];

const columns = createColumns();

// Lifecycle and Watchers
onMounted(loadLogs);
watch([currentPage, pageSize], loadLogs);

const handleSearch = () => {
  currentPage.value = 1;
  loadLogs();
};

const resetFilters = () => {
  filters.group_name = "";
  filters.key_value = "";
  filters.is_success = "";
  filters.status_code = "";
  filters.source_ip = "";
  filters.error_contains = "";
  filters.start_time = null;
  filters.end_time = null;
  handleSearch();
};

function changePage(page: number) {
  currentPage.value = page;
}

function changePageSize(size: number) {
  pageSize.value = size;
  currentPage.value = 1;
}
</script>

<template>
  <div class="log-table-container">
    <n-space vertical>
      <!-- 工具栏 -->
      <div class="toolbar">
        <div class="toolbar-left">
          <n-space>
            <n-select
              v-model:value="filters.is_success"
              :options="successOptions"
              size="small"
              style="width: 120px"
              @update:value="handleSearch"
            />
            <n-date-picker
              v-model:value="filters.start_time"
              type="datetime"
              clearable
              size="small"
              placeholder="开始时间"
              style="width: 180px"
            />
            <n-date-picker
              v-model:value="filters.end_time"
              type="datetime"
              clearable
              size="small"
              placeholder="结束时间"
              style="width: 180px"
            />
          </n-space>
        </div>
        <div class="toolbar-right">
          <n-space>
            <n-input
              v-model:value="filters.group_name"
              placeholder="分组名"
              size="small"
              clearable
              style="width: 120px"
              @keyup.enter="handleSearch"
            />
            <n-input
              v-model:value="filters.key_value"
              placeholder="Key 片段"
              size="small"
              clearable
              style="width: 150px"
              @keyup.enter="handleSearch"
            />
            <n-input-group>
              <n-input
                v-model:value="filters.error_contains"
                placeholder="错误信息"
                size="small"
                clearable
                style="width: 150px"
                @keyup.enter="handleSearch"
              />
              <n-button ghost size="small" :disabled="loading" @click="handleSearch">
                <n-icon :component="Search" />
              </n-button>
            </n-input-group>
            <n-button size="small" @click="resetFilters">重置</n-button>
          </n-space>
        </div>
      </div>
      <div class="table-main">
        <!-- 表格 -->
        <div class="table-container">
          <n-spin :show="loading">
            <n-data-table :columns="columns" :data="logs" :bordered="false" remote size="small" />
          </n-spin>
        </div>

        <!-- 分页 -->
        <div class="pagination-container">
          <div class="pagination-info">
            <span>共 {{ total }} 条记录</span>
            <n-select
              v-model:value="pageSize"
              :options="[
                { label: '15条/页', value: 15 },
                { label: '30条/页', value: 30 },
                { label: '50条/页', value: 50 },
                { label: '100条/页', value: 100 },
              ]"
              size="small"
              style="width: 100px; margin-left: 12px"
              @update:value="changePageSize"
            />
          </div>
          <div class="pagination-controls">
            <n-button
              size="small"
              :disabled="currentPage <= 1"
              @click="changePage(currentPage - 1)"
            >
              上一页
            </n-button>
            <span class="page-info">第 {{ currentPage }} 页，共 {{ totalPages }} 页</span>
            <n-button
              size="small"
              :disabled="currentPage >= totalPages"
              @click="changePage(currentPage + 1)"
            >
              下一页
            </n-button>
          </div>
        </div>
      </div>
    </n-space>
  </div>
</template>

<style scoped>
.log-table-container {
  /* background: white; */
  /* border-radius: 8px; */
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.05);
  display: flex;
  flex-direction: column;
  /* height: 100%; */
}
.toolbar {
  background: white;
  border-radius: 8px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px;
  border-bottom: 1px solid #f0f0f0;
}
.table-main {
  background: white;
  border-radius: 8px;
  overflow: hidden;
}
.table-container {
  /* background: white;
  border-radius: 8px; */
  flex: 1;
  overflow: hidden;
  position: relative;
}
.empty-container {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}
.pagination-container {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px;
  border-top: 1px solid #f0f0f0;
}
.pagination-info {
  display: flex;
  align-items: center;
  gap: 12px;
  font-size: 13px;
  color: #666;
}
.pagination-controls {
  display: flex;
  align-items: center;
  gap: 12px;
}
.page-info {
  font-size: 13px;
  color: #666;
}
</style>
