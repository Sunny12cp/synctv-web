<script lang="ts" setup>
import { computed, onMounted, ref } from "vue";
import { ElNotification, ElMessage } from "element-plus";
import { Search } from "@element-plus/icons-vue";
import { userStore } from "@/stores/user";
import {
  userListApi,
  banUserApi,
  unBanUserApi,
  addAdminApi,
  delAdminApi,
  approveUserApi,
  delUserApi
} from "@/services/apis/admin";
import CopyButton from "@/components/CopyButton.vue";
import userRooms from "@/components/admin/dialogs/userRooms.vue";
import newUser from "@/components/admin/dialogs/newUser.vue";
import { ROLE, role } from "@/types/User";

const props = defineProps<{
  title: string;
}>();

const userRoomsDialog = ref<InstanceType<typeof userRooms>>();
const newUserDialog = ref<InstanceType<typeof newUser>>();
const getRole = (rawRole: ROLE) => role[rawRole];
const roles = computed(() => {
  const v = Object.values(role);
  return v.filter((r) => r !== role[ROLE.Visitor] && r !== role[ROLE.Unknown]);
});

const { token } = userStore();
const totalItems = ref(0);
const currentPage = ref(1);
const pageSize = ref(10);
const order = ref("desc");
const sort = ref("createdAt");
const keyword = ref("");
const search = ref("all");
const role_ = ref("");
const { state, execute: reqUserListApi, isLoading: userListLoading } = userListApi();
const getUserListApi = async () => {
  try {
    await reqUserListApi({
      headers: {
        Authorization: token.value
      },
      params: {
        page: currentPage.value,
        max: pageSize.value,
        sort: sort.value,
        order: order.value,

        role: role_.value,
        search: search.value,
        keyword: keyword.value
      }
    });
    if (state.value) {
      totalItems.value = state.value.total;
    }
  } catch (err: any) {
    console.error(err);
    ElNotification({
      title: "Failed to obtain user list",
      type: "error",
      message: err.response.data.error || err.message
    });
  }
};

// 封禁 / 解封 用户
const banUser = async (id: string, is: boolean) => {
  try {
    const config = {
      headers: {
        Authorization: token.value
      },
      data: {
        id: id
      }
    };
    is ? await banUserApi().execute(config) : await unBanUserApi().execute(config);
    ElNotification({
      title: `${is ? "Ban" : "Unblock"} Success`,
      type: "success"
    });
    await getUserListApi();
  } catch (err: any) {
    console.error(err);
    ElNotification({
      title: "Error",
      type: "error",
      message: err.response.data.error || err.message
    });
  }
};

// 设管理
const setAdmin = async (id: string, is: boolean) => {
  try {
    const config = {
      headers: {
        Authorization: token.value
      },
      data: {
        id: id
      }
    };
    is ? await addAdminApi().execute(config) : await delAdminApi().execute(config);
    ElNotification({
      title: "Success",
      type: "success"
    });
    await getUserListApi();
  } catch (err: any) {
    console.error(err);
    ElNotification({
      title: "Error",
      type: "error",
      message: err.response.data.error || err.message
    });
  }
};

// 查看用户房间
const getUserRoom = async (id: string) => {
  userRoomsDialog.value?.openDialog(id);
};

// 允许用户注册
const { execute: approveUser, isLoading: approveLoading } = approveUserApi();
const approve = async (id: string) => {
  try {
    await approveUser({
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
    await getUserListApi();
  } catch (err: any) {
    console.error(err);
    ElNotification({
      title: "Error",
      type: "error",
      message: err.response.data.error || err.message
    });
  }
};

// 删用户
const delUser = async (id: string) => {
  try {
    await delUserApi().execute({
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
    await getUserListApi();
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
  await getUserListApi();
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
          v-model="role_"
          class="max-xl:my-2 max-lg:w-full"
          placeholder="Rights Groups"
          style="width: 90px"
          @change="getUserListApi()"
        >
          <el-option label="ALL" value="" />
          <el-option v-for="r in roles" :label="r" :value="r.toLowerCase()" />
        </el-select>
        <el-input
          class="w-fit max-lg:w-full"
          v-model="keyword"
          placeholder="Search"
          @keyup.enter="getUserListApi()"
          required
        >
          <template #prepend>
            <el-select
              v-model="search"
              @change="getUserListApi()"
              placeholder="Select"
              style="width: 90px"
            >
              <el-option label="All" value="all" />
              <el-option label="Name" value="name" />
              <el-option label="ID" value="roomId" />
            </el-select>
          </template>
          <template #append>
            <el-button :icon="Search" @click="getUserListApi()" />
          </template>
        </el-input>
      </div>
      <el-button class="max-xl:mt-3" type="primary" @click="newUserDialog?.openDialog">
        Add user
      </el-button>
    </div>
    <div class="card-body">
      <el-table :data="state?.list" v-loading="userListLoading" style="width: 100%">
        <el-table-column prop="username" label="Username" width="200" />
        <el-table-column prop="id" label="ID" width="120">
          <template #default="scope">
            <div class="flex overflow-hidden text-ellipsis max-w-[100px]">
              <span class="truncate">{{ scope.row.id }}</span>
              <CopyButton size="small" :value="scope.row.id" />
            </div>
          </template>
        </el-table-column>
        <el-table-column prop="role" label="Role" width="70">
          <template #default="scope">
            {{ getRole(scope.row.role) }}
          </template>
        </el-table-column>
        <el-table-column prop="roomList" label="Room List" width="120">
          <template #default="scope">
            <el-button type="primary" plain @click="getUserRoom(scope.row.id)"> 查看 </el-button>
          </template>
        </el-table-column>
        <el-table-column prop="createdAt" label="Creation Time" width="160">
          <template #default="scope">
            {{ new Date(scope.row.createdAt).toLocaleString() }}
          </template>
        </el-table-column>
        <el-table-column fixed="right" label="Operate">
          <template #default="scope">
            <el-button
              v-if="scope.row.role === ROLE.Pending"
              type="success"
              @click="approve(scope.row.id)"
              :loading="approveLoading"
            >
              By registering
            </el-button>
            <div v-else>
              <el-button
                v-if="scope.row.role === ROLE.Banned"
                type="warning"
                @click="banUser(scope.row.id, false)"
              >
                Unblock
              </el-button>

              <div v-else class="phone-button">
                <el-button type="danger" plain @click="banUser(scope.row.id, true)">
                  Ban
                </el-button>

                <el-button
                  v-if="scope.row.role < ROLE.Admin"
                  type="primary"
                  @click="setAdmin(scope.row.id, true)"
                >
                  Set as Admin
                </el-button>

                <el-button v-else type="warning" @click="setAdmin(scope.row.id, false)">
                  Unmanage
                </el-button>

                <el-popconfirm title="Are you sure you want to delete this user?" @confirm="delUser(scope.row.id)">
                  <template #reference>
                    <el-button type="danger"> Delete </el-button>
                  </template>
                </el-popconfirm>
              </div>
            </div>
          </template>
        </el-table-column>
      </el-table>
    </div>
    <div class="card-footer flex flex-wrap justify-between overflow-hidden">
      <el-button type="success" @click="getUserListApi()" :loading="userListLoading">
        Update List
      </el-button>

      <div class="flex flex-wrap">
        <div class="text-base mr-2 max-xl:my-2">
          Sort By：<el-select
            v-model="sort"
            class="mr-2"
            placeholder="Sort By"
            @change="getUserListApi()"
            style="width: 150px"
          >
            <el-option label="Username" value="username" />
            <el-option label="Creation Time" value="createdAt" />
          </el-select>
          <button
            class="btn btn-dense"
            @click="
              order === 'desc' ? (order = 'asc') : (order = 'desc');
              getUserListApi();
            "
          >
            {{ order === "asc" ? "👆" : "👇" }}
          </button>
        </div>

        <el-pagination
          v-model:current-page="currentPage"
          v-model:page-size="pageSize"
          :pager-count="5"
          layout="sizes, prev, pager, next, jumper"
          :total="totalItems"
          @size-change="getUserListApi()"
          @current-change="getUserListApi()"
          class="flex-wrap"
        />
      </div>
    </div>
  </div>

  <userRooms ref="userRoomsDialog" />
  <newUser ref="newUserDialog" @update-user-list="getUserListApi()" />
</template>

<style lang="less" scoped>
@media (max-width: 1540px) {
  .phone-button {
    .el-button {
      @apply m-0 mb-2 mr-2;
    }
  }
}
</style>
