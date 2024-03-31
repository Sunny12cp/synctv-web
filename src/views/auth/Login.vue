<script setup lang="ts">
import { onMounted, ref } from "vue";
import { ElNotification, ElMessage } from "element-plus";
import { OAuth2Platforms, loginWithOAuth2, LoginApi } from "@/services/apis/auth";
import { userInfo } from "@/services/apis/user";
import { useRouteQuery } from "@vueuse/router";
import { strLengthLimit, getAppIcon } from "@/utils";
import { userStore } from "@/stores/user";
import router from "@/router/index";

const platforms: { [key: string]: { name: string; class: string } } = {
  github: {
    name: "Github",
    class: "btn-white"
  },
  microsoft: {
    name: "Microsoft",
    class: "btn-default"
  },
  google: {
    name: "Google",
    class: "btn-white"
  },
  "feishu-sso": {
    name: "飞书SSO",
    class: "btn-white"
  },
  authing: {
    name: "Authing",
    class: "btn-white"
  },
  xiaomi: {
    name: "Xiaomi",
    class: "btn-white"
  },
  baidu: {
    name: "Baidu",
    class: "btn-white"
  },
  "baidu-netdisk": {
    name: "Baidu-Netdisk",
    class: "btn-white"
  },
  gitee: {
    name: "Gitee",
    class: "btn-error"
  },
  gitlab: {
    name: "GitLab",
    class: "btn-error"
  },
  qq: {
    name: "QQ",
    class: "btn-default"
  }
};

const formData = ref({
  username: localStorage.getItem("uname") || "",
  password: localStorage.getItem("password") || ""
});
const savePwd = ref(false);

const redirect = useRouteQuery("redirect");
console.log("redirect: ", (redirect.value as string) ?? "");

const { getUserInfo: updateUserInfo, updateToken } = userStore();
const { execute: reqLoginApi, state: loginData } = LoginApi();
const login = async () => {
  if (formData.value?.username === "" || formData.value?.password === "") {
    ElNotification({
      title: "Error",
      message: "Please Fill It Completely",
      type: "error"
    });
    return;
  }
  try {
    for (const key in formData.value) {
      strLengthLimit(key, 32);
    }
    await reqLoginApi({
      data: formData.value
    });
    if (!loginData.value)
      return ElNotification({
        title: "Error",
        message: "Invalid Token",
        type: "error"
      });

    updateToken(loginData.value.token);
    localStorage.setItem("uname", formData.value.username);
    localStorage.setItem("password", savePwd.value ? formData.value.password : "");

    const state = await userInfo().execute({
      headers: {
        Authorization: loginData.value?.token ?? ""
      }
    });

    if (state.value) {
      updateUserInfo(state.value);
      localStorage.setItem("uname", state.value.username);
      ElNotification({
        title: "Success",
        type: "success"
      });

      router.replace((redirect.value as string) || "/");
    }
  } catch (err: any) {
    console.error(err);
    ElNotification({
      title: "Error",
      message: err.response.data.error || err.message,
      type: "error"
    });
  }
};

const { execute: reqOAuth2PlatformsApi, state: OAuth2Platforms_ } = OAuth2Platforms();
const getOAuth2Platforms = async () => {
  try {
    await reqOAuth2PlatformsApi();
  } catch (err: any) {
    console.error(err);
    ElNotification({
      title: "Error",
      message: err.response.data.error || err.message,
      type: "error"
    });
  }
};

const { state, execute } = loginWithOAuth2();
const useOAuth2 = async (platform: string) => {
  try {
    await execute({
      data: {
        redirect: (redirect.value as string) ?? ""
      },
      url: "/oauth2/login/" + platform
    });
    if (state.value) window.location.href = state.value.url;
  } catch (err: any) {
    console.error(err);
    ElNotification({
      title: "Error",
      message: err.response.data.error || err.message,
      type: "error"
    });
  }
};

onMounted(async () => {
  await getOAuth2Platforms();
});
</script>

<template>
  <div class="room">
    <form @submit.prevent="" class="login-box">
      <input
        class="l-input"
        type="text"
        v-model="formData.username"
        placeholder="Username"
        required
      />
      <br />
      <input
        class="l-input"
        type="password"
        v-model="formData.password"
        placeholder="Password"
        required
      />
      <br />
      <div class="text-sm"><b>Notice：</b>All input boxes can only input a maximum of 32 characters</div>
      <div>
        <input class="w-auto" type="checkbox" v-model="savePwd" />
        <label title="Save the plain text to this machine~">&nbsp;Remember PASSWORD</label>
      </div>
      <button class="btn m-[10px]" @click="login">Log in</button>
    </form>
    <br />
    <div
      v-if="OAuth2Platforms_?.enabled && OAuth2Platforms_.enabled.length > 0"
      class="sm:w-96 w-full m-auto"
    >
      <h4 class="text-[18px] font-bold">Log in using a third-party platform</h4>
      <button
        v-for="item in OAuth2Platforms_?.enabled"
        :class="`inline-flex items-center btn ${
          platforms[item] ? platforms[item].class : 'btn-black'
        } m-[10px] hover:px-[10px]`"
        @click="useOAuth2(item)"
      >
        <el-image class="w-4 mr-2 rounded-lg" :src="getAppIcon(item)"> </el-image>
        {{ platforms[item] ? platforms[item].name : item }}
      </button>
    </div>
  </div>
</template>

<style lang="less" scoped>
.room {
  text-align: center;
  margin-top: 5vmax;

  .login-box {
    @apply sm:w-96 w-full m-auto;

    input {
      width: 70%;

      &:hover {
        padding: 10px 15px;
        width: 74%;
      }
    }

    .btn {
      width: 70%;
      padding: 10px 15px;

      &:hover {
        padding: 12px 15px;
      }
    }
  }
}
</style>
