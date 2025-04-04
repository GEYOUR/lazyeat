<script setup lang="ts">
import AppMenu from "@/components/Menu.vue";
import pyApi from "@/py_api";
import use_app_store from "@/store/app";
import Home from "@/view/Home.vue";
import { getVersion } from "@tauri-apps/api/app";
import { getCurrentWindow } from "@tauri-apps/api/window";
import { LazyStore } from "@tauri-apps/plugin-store";
import {
  restoreStateCurrent,
  saveWindowState,
  StateFlags,
} from "@tauri-apps/plugin-window-state";
import { ElAside, ElContainer, ElMain } from "element-plus";
import { onMounted, ref, watch } from "vue";

const appVersion = ref("");
const ready = ref(false);
const is_dev = import.meta.env.DEV;

onMounted(async () => {
  appVersion.value = await getVersion();
  ready.value = await pyApi.ready();

  const timer = setInterval(async () => {
    ready.value = await pyApi.ready();
    if (ready.value) {
      clearInterval(timer);
    }
  }, 5000);

  await getCurrentWindow().onCloseRequested(async () => {
    await saveWindowState(StateFlags.ALL);

    if (!is_dev) {
      await pyApi.shutdown();
    }
  });
});

// 窗口恢复上一次状态
onMounted(() => {
  restoreStateCurrent(StateFlags.ALL);
});

// app_store 数据加载
const app_store = use_app_store();
const app_store_json = new LazyStore("settings.json");
onMounted(async () => {
  const config_data = await app_store_json.get("config");
  console.log("config_data", config_data);
  if (config_data) {
    Object.assign(app_store.config, JSON.parse(JSON.stringify(config_data)));
  }
});

watch(
  () => app_store.config,
  async (value) => {
    await app_store_json.set("config", value);
    app_store_json.save();
  },
  { deep: true }
);

// 通知
import {
  isPermissionGranted,
  requestPermission,
} from "@tauri-apps/plugin-notification";

onMounted(async () => {
  let permissionGranted = await isPermissionGranted();
  if (!permissionGranted) {
    const permission = await requestPermission();
    permissionGranted = permission === "granted";
  }
});

// 使用默认浏览器打开 iframe 中的 <a> 标签
import { openUrl } from "@tauri-apps/plugin-opener";
function setupIframeListener(event: Event) {
  const iframe = event.target as HTMLIFrameElement;
  iframe.contentWindow?.addEventListener("click", async (e) => {
    const link = (e.target as HTMLElement).closest("a");
    if (link && link.href) {
      e.preventDefault();
      await openUrl(link.href);
    }
  });
}
</script>

<template>
  <n-spin :show="!ready" size="large">
    <template #description> 手势识别模块加载中... </template>

    <el-container v-if="ready">
      <el-aside width="200px">
        <div class="aside-header">
          <img
            style="width: 30px; height: 30px"
            src="/lazyeat.png"
            alt="logo"
            class="logo"
          />
          <span class="logo-text">Lazyeat {{ appVersion }}</span>
        </div>
        <AppMenu />
      </el-aside>
      <el-container>
        <el-main>
          <n-card class="ad-container">
            <iframe
              :src="
                is_dev
                  ? '/lazyeat-ad/html/index.html'
                  : 'https://lazyeat-ad.pages.dev/'
              "
              width="100%"
              height="100%"
              @load="setupIframeListener"
            ></iframe>
          </n-card>
          <Home />
        </el-main>
      </el-container>
    </el-container>
  </n-spin>
</template>

<style lang="scss">
.n-spin-container {
  height: 100%;
  width: 100%;
  overflow: auto;
}

.n-spin-content {
  height: 100%;
  width: 100%;
  overflow: auto;
}
</style>

<style scoped lang="scss">
.el-container {
  height: 100%;
  width: 100%;
}

.el-aside {
  background-color: #f5f7fa;
  border-right: 1px solid #e6e6e6;
}

.aside-header {
  height: 60px;
  display: flex;
  align-items: center;
  padding: 0 20px;
  border-bottom: 1px solid #e6e6e6;
}

.logo {
  width: 30px;
  height: 30px;
  margin-right: 10px;
}

.logo-text {
  font-size: 16px;
}

// 广告区域
.ad-container {
  height: 220px;
  background-color: transparent;
  margin-bottom: 24px;

  iframe {
    border: none;
  }

  :deep(.n-card__content) {
    padding: 0 !important;
    padding-top: 0 !important;
  }
}
</style>
