<template>
  <footer id="footer" @click.stop>
    <div class="copyright">
      <span class="site-name">{{ siteName }}</span>
      <span class="year">{{ fullYear }}</span>
      <span class="anthor" @click="jumpTo(SITE_REPO)">
        {{ siteAnthor }}
      </span>
      <span v-if="icp" class="icp" @click="jumpTo('https://beian.miit.gov.cn')">
        {{ icp }}
      </span>
      <span class="about" @click="aboutSiteModal = true">关于</span>
    </div>
    <!-- 关于 -->
    <n-modal
      preset="card"
      :bordered="false"
      v-model:show="aboutSiteModal"
      transform-origin="center"
    >
      <div class="about-modal">
        <div class="about">
          <span class="name">{{ siteName }}</span>
          <span class="version">v {{ packageJson.version }}</span>
        </div>
        <div class="based">
          基于
          <span class="link" @click="jumpTo(UPSTREAM_REPO)">
            Snavigation
          </span>
          修改而来
        </div>
        <div class="desc">
          <n-space class="link" justify="center">
            <n-button strong secondary @click="jumpTo(SITE_REPO)">
              Github
            </n-button>
          </n-space>
        </div>
      </div>
    </n-modal>
  </footer>
</template>

<script setup>
import { ref } from "vue";
import { setStore } from "@/stores";
import { NModal, NButton, NSpace } from "naive-ui";
import packageJson from "@/../package.json";

const set = setStore();

// 站点数据
const icp = import.meta.env.VITE_ICP;
const siteName = import.meta.env.VITE_SITE_TITLE;
const siteAnthor = import.meta.env.VITE_SITE_ANTHOR;
const copyrightLink = import.meta.env.VITE_SITE_COPYRIGHTLINK;

// 版权年份区间
const YEAR_START = 2024;
const YEAR_END = 2026;
const fullYear =
  YEAR_START === YEAR_END ? `${YEAR_START}` : `${YEAR_START}-${YEAR_END}`;

// 上游项目（本站基于其修改而来）
const UPSTREAM_REPO = "https://github.com/imsyy/Snavigation";
// 本站仓库（优先取 .env 配置，回退到默认地址）
const SITE_REPO = copyrightLink || "https://github.com/Regionnor/Snavigation";

// 关于弹窗数据
const aboutSiteModal = ref(false);

// 跳转
const jumpTo = (url) => {
  if (set.urlJumpType === "href") {
    window.location.href = url;
  } else if (set.urlJumpType === "open") {
    window.open(url, "_blank");
  }
};
</script>

<style lang="scss" scoped>
#footer {
  position: absolute;
  display: flex;
  align-items: center;
  justify-content: center;
  bottom: 0;
  height: 50px;
  width: 100%;
  color: var(--main-text-color);
  z-index: 1;
  .copyright {
    display: flex;
    align-items: center;
    font-size: 13px;
    span {
      margin: 0 2px;
      opacity: 0.6;
      transition: opacity 0.3s;
      &::before {
        opacity: 0.6;
        transition: none;
      }
    }
    .year {
      &::before {
        content: "@";
        opacity: 1;
        margin-right: 4px;
      }
    }
    .icp {
      &::before {
        content: "|";
        margin-right: 4px;
      }
    }
    .about {
      &::before {
        content: "|";
        margin-right: 4px;
      }
    }
    .anthor,
    .icp,
    .about {
      cursor: pointer;
      &:hover {
        opacity: 1;
      }
    }
  }
}
.about-modal {
  margin-bottom: 10px;
  .about {
    display: flex;
    flex-direction: column;
    align-items: center;
    width: 100%;
    .name {
      font-size: 26px;
      font-weight: bold;
      margin-bottom: 4px;
    }
    .version {
      opacity: 0.6;
      font-size: 16px;
    }
  }
  .based {
    margin-top: 12px;
    font-size: 13px;
    opacity: 0.7;
    text-align: center;
    .link {
      cursor: pointer;
      text-decoration: underline;
      opacity: 1;
      color: var(--main-text-color);
      &:hover {
        opacity: 0.8;
      }
    }
  }
  .desc {
    margin-top: 20px;
  }
}
</style>
