<script setup lang="ts">
import { ref, onMounted } from "vue";
import { useVendorApi, statusList } from "@/hooks/admin/setting/useVendor";
import { getAppIcon } from "@/utils/index";
import VendorEditor from "@/components/admin/dialogs/editVendor.vue";
import { RefreshRight } from "@element-plus/icons-vue";

const props = defineProps<{
  title: string;
}>();
const vendorEditorDialog = ref<InstanceType<typeof VendorEditor>>();

const {
  vendorsListState,
  getVendorsList,
  getVendorsListLoading,
  currentPage,
  pageSize,
  selections,
  batchDelete,
  batchReconnect
} = useVendorApi();

const handleSelectionChange = (e: { info: { backend: { endpoint: string } } }[]) => {
  selections.value = e;
};

onMounted(async () => {
  await getVendorsList();
});
</script>

<template>
  <div class="card">
    <div class="card-title flex flex-wrap justify-between items-center">
      <div>
        {{ props.title }}
      </div>
      <div>
        <el-popconfirm
          v-if="selections.length >= 2"
          title="Are you sure you want to delete? "
          @confirm="batchDelete(selections)"
        >
          <template #reference>
            <el-button class="max-xl:mt-3" type="warning" @click=""> 批量删除 </el-button>
          </template>
        </el-popconfirm>

        <el-button
          v-if="selections.length >= 2"
          class="max-xl:mt-3"
          type="primary"
          @click="batchReconnect(selections)"
        >
          Batch Reconnection
        </el-button>

        <el-button
          class="max-xl:mt-3"
          type="primary"
          @click="vendorEditorDialog?.openDialog(false)"
        >
          Adding A Parser
        </el-button>
      </div>
    </div>
    <div class="card-body">
      <el-table
        :stripe="true"
        :data="vendorsListState?.list"
        @selection-change="handleSelectionChange($event)"
        style="width: 100%"
        table-layout="fixed"
      >
        <el-table-column type="selection" width="55" />
        <el-table-column prop="info.backend.endpoint" label="节点">
          <template #default="scope">
            <div style="display: flex; align-items: center">
              <el-image
                v-if="scope.row.info.backend.consul.serviceName"
                style="max-width: 15px"
                :src="getAppIcon('consul')"
              />
              <el-image
                v-if="scope.row.info.backend.etcd.serviceName"
                style="max-width: 15px"
                :src="getAppIcon('etcd')"
              />
              {{ scope.row.info.backend.endpoint }}
            </div>
          </template>
        </el-table-column>
        <el-table-column prop="info.backend.comment" label="备注" />
        <el-table-column prop="info.backend.usedBy.enabled" label="应用">
          <template #default="scope">
            <div v-if="scope.row.info.usedBy.enabled">
              <el-image
                v-if="scope.row.info.usedBy.alist"
                style="max-width: 15px"
                :src="getAppIcon('alist')"
              />
              <el-image
                v-if="scope.row.info.usedBy.bilibili"
                style="max-width: 15px"
                :src="getAppIcon('bilibili')"
              />
              <el-image
                v-if="scope.row.info.usedBy.emby"
                style="max-width: 15px"
                :src="getAppIcon('emby')"
              />
            </div>
            <div v-else>Disabled</div>
          </template>
        </el-table-column>
        <el-table-column prop="info.backend.jwtSecret" label="JWT密钥" />
        <el-table-column prop="info.backend.timeOut" label="超时时间" />
        <el-table-column prop="status" label="Current Status">
          <template #default="scope">
            <el-tag v-if="scope" :type="statusList[scope.row.status as 0 | 1 | 2 | 3 | 4].type">{{
              statusList[scope.row.status as 0 | 1 | 2 | 3 | 4].name
            }}</el-tag>
            <el-tag v-else type="info">Invalid State</el-tag>
            <el-tooltip v-if="scope.row.status > 0">
              <template #content>Reconnect</template>
              <el-button
                class="ml-2"
                :icon="RefreshRight"
                size="small"
                circle
                @click="batchReconnect(scope.row.info.backend.endpoint)"
              />
            </el-tooltip>
          </template>
        </el-table-column>

        <el-table-column prop="info.backend.endpoint" label="Operate">
          <template #default="scope">
            <el-button type="primary" @click="vendorEditorDialog?.openDialog(true, scope.row.info)"
              >Edit</el-button
            >
            <el-popconfirm
              title="Are You Sure You Want To Delete?"
              @confirm="batchDelete(scope.row.info.backend.endpoint)"
            >
              <template #reference>
                <el-button type="danger">Delete</el-button>
              </template>
            </el-popconfirm>
          </template>
        </el-table-column>
      </el-table>
    </div>
    <div class="card-footer flex flex-wrap justify-between overflow-hidden">
      <el-button type="success" @click="getVendorsList()" :loading="getVendorsListLoading"
        >Update List</el-button
      >
      <el-pagination
        v-if="vendorsListState?.list?.length != 0"
        v-model:current-page="currentPage"
        v-model:page-size="pageSize"
        :pager-count="10"
        layout="sizes, prev, pager, next, jumper"
        :total="vendorsListState?.total"
        @size-change="getVendorsList()"
        @current-change="getVendorsList()"
        class="flex-wrap"
      />
    </div>
  </div>
  <VendorEditor ref="vendorEditorDialog" @update-vendor-list="getVendorsList(true)" />
</template>
