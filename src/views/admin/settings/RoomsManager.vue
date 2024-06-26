<script lang="ts" setup>
import { onMounted, ref } from "vue";
import { ElNotification, ElMessage } from "element-plus";
import { Search } from "@element-plus/icons-vue";
import { userStore } from "@/stores/user";
import { roomListApi, banRoomApi, unBanRoomApi, approveRoomApi } from "@/services/apis/admin";
import CopyButton from "@/components/CopyButton.vue";
import { RoomStatus, roomStatus } from "@/types/Room";
import { useTimeAgo } from "@vueuse/core";

const props = defineProps<{
  title: string;
}>();

const getStatus = (rawStatus: RoomStatus) => roomStatus[rawStatus];

const { token } = userStore();
const totalItems = ref(0);
const currentPage = ref(1);
const pageSize = ref(10);
const order = ref("desc");
const sort = ref("createdAt");
const keyword = ref("");
const search = ref("all");
const status = ref("");
const { state, execute: reqRoomListApi, isLoading: roomListLoading } = roomListApi();
const getRoomListApi = async () => {
  try {
    await reqRoomListApi({
      headers: {
        Authorization: token.value
      },
      params: {
        page: currentPage.value,
        max: pageSize.value,
        sort: sort.value,
        order: order.value,
        search: search.value,
        keyword: keyword.value,
        status: status.value
      }
    });
    if (state.value) {
      totalItems.value = state.value.total;
    }
  } catch (err: any) {
    console.error(err);
    ElNotification({
      title: "Failed To Get Room List",
      type: "error",
      message: err.response.data.error || err.message
    });
  }
};

// 封禁 / 解封 房间
const { execute: ban, isLoading: banLoading } = banRoomApi();
const { execute: unBan, isLoading: unBanLoading } = unBanRoomApi();
const banRoom = async (id: string, is: boolean) => {
  try {
    const config = {
      headers: {
        Authorization: token.value
      },
      data: {
        id: id
      }
    };
    is ? await ban(config) : await unBan(config);
    ElNotification({
      title: `${is ? "Ban" : "Unblock"} Success`,
      type: "success"
    });
    await getRoomListApi();
  } catch (err: any) {
    console.error(err);
    ElNotification({
      title: "Error",
      type: "error",
      message: err.response.data.error || err.message
    });
  }
};

// 允许创建
const approveCreate = async (id: string) => {
  try {
    await approveRoomApi().execute({
      headers: {
        Authorization: token.value
      },
      data: {
        id: id
      }
    });
    ElNotification({
      title: "Success",
      type: "success"
    });
    await getRoomListApi();
  } catch (err: any) {
    console.error(err);
    ElNotification({
      title: "Error",
      type: "error",
      message: err.response.data.error || err.message
    });
  }
};

onMounted(async () => {
  await getRoomListApi();
});
</script>

<template>
  <div class="card">
    <div class="card-title flex flex-wrap justify-between items-center">
      <div>
        {{ props.title }}
      </div>
      <div>
        <el-select
          v-model="status"
          placeholder="状态"
          style="width: 100px"
          @change="getRoomListApi()"
        >
          <el-option label="ALL" value="" />
          <el-option v-for="r in Object.values(roomStatus)" :label="r" :value="r.toLowerCase()" />
        </el-select>
        <el-input
          class="w-fit max-lg:w-full max-xl:my-2"
          v-model="keyword"
          placeholder="State"
          @keyup.enter="getRoomListApi()"
          required
        >
          <template #prepend>
            <el-select
              v-model="search"
              @change="getRoomListApi()"
              placeholder="Select"
              style="width: 90px"
            >
              <el-option label="All" value="all" />
              <el-option label="Name" value="name" />
              <el-option label="ID" value="roomId" />
            </el-select>
          </template>
          <template #append>
            <el-button :icon="Search" @click="getRoomListApi()" />
          </template>
        </el-input>
      </div>

      <div class="text-base max-xl:w-full">
        Sort By：<el-select
          v-model="sort"
          class="mr-2"
          placeholder="Sort By"
          @change="getRoomListApi()"
          style="width: 150px"
        >
          <el-option label="Room Name" value="name" />
          <el-option label="Created Time" value="createdAt" />
        </el-select>
        <button
          class="btn btn-dense"
          @click="
            order === 'desc' ? (order = 'asc') : (order = 'desc');
            getRoomListApi();
          "
        >
          {{ order === "asc" ? "👆" : "👇" }}
        </button>
      </div>
    </div>
    <div class="card-body">
      <el-table :data="state?.list" v-loading="roomListLoading" style="width: 100%">
        <el-table-column prop="roomName" label="Room Name" width="150" />
        <el-table-column prop="roomId" label="ID" width="120">
          <template #default="scope">
            <div class="flex overflow-hidden text-ellipsis max-w-[100px]">
              <span class="truncate">{{ scope.row.roomId }}</span>
              <CopyButton size="small" :value="scope.row.roomId" />
            </div>
          </template>
        </el-table-column>
        <el-table-column prop="creator" label="Creator">
          <template #default="scope">
            {{ scope.row.creator }}
          </template>
        </el-table-column>
        <el-table-column prop="peopleNum" label="Online Users" width="80">
          <template #default="scope">
            <el-tag disabled :type="scope.row.peopleNum > 0 ? 'success' : 'danger'">
              {{ scope.row.peopleNum }}
            </el-tag>
          </template>
        </el-table-column>
        <el-table-column prop="needPassword" label="Password Protection" width="100">
          <template #default="scope">
            <el-tag disabled :type="scope.row.needPassword ? 'danger' : 'success'">
              {{ scope.row.needPassword ? "Password" : "No Password" }}
            </el-tag>
          </template>
        </el-table-column>
        <el-table-column prop="createdAt" label="Creation time">
          <template #default="scope">
            {{ useTimeAgo(new Date(scope.row.createdAt)).value }}
          </template>
        </el-table-column>
        <el-table-column prop="status" label="Status" width="100">
          <template #default="scope">
            {{ getStatus(scope.row.status) }}
          </template>
        </el-table-column>
        <el-table-column fixed="right" label="Oprate">
          <template #default="scope">
            <el-button
              v-if="scope.row.status !== RoomStatus.Banned"
              type="danger"
              :loading="banLoading"
              @click="banRoom(scope.row.roomId, true)"
            >
              Ban
            </el-button>
            <el-button
              v-else
              type="warning"
              :loading="unBanLoading"
              @click="banRoom(scope.row.roomId, false)"
            >
              Unban
            </el-button>
            <el-button
              v-if="scope.row.status === RoomStatus.Pending"
              type="success"
              :loading="banLoading"
              @click="approveCreate(scope.row.roomId)"
            >
              Allow Creation
            </el-button>
          </template>
        </el-table-column>
      </el-table>
    </div>
    <div class="card-footer flex flex-wrap justify-between overflow-hidden">
      <el-button type="success" @click="getRoomListApi()" :loading="roomListLoading"
        >Update List</el-button
      >
      <el-pagination
        v-if="state?.list?.length != 0"
        v-model:current-page="currentPage"
        v-model:page-size="pageSize"
        :pager-count="5"
        layout="sizes, prev, pager, next, jumper"
        :total="totalItems"
        @size-change="getRoomListApi()"
        @current-change="getRoomListApi()"
        class="flex-wrap"
      />
    </div>
  </div>

  <userRooms ref="userRoomsDialog" />
</template>
