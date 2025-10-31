<template>
  <div id="tags-view-container" class="tags-view-container">
    <scroll-pane ref="scrollPaneRef" class="tags-view-wrapper" @scroll="handleScroll">
      <router-link v-for="tag in visitedViews" :key="tag.path" :data-path="tag.path"
        :class="{ 'active': isActive(tag), 'has-icon': tagsIcon }"
        :to="{ path: tag.path ? tag.path : '', query: tag.query, fullPath: tag.fullPath ? tag.fullPath : '' }"
        class="tags-view-item" :style="activeStyle(tag)" @click.middle="!isAffix(tag) ? closeSelectedTag(tag) : ''"
        @contextmenu.prevent="openMenu(tag, $event)">
        <svg-icon v-if="tagsIcon && tag.meta && tag.meta.icon && tag.meta.icon !== '#'" :icon-class="tag.meta.icon" />
        <span class="tags-view-item-title">{{ tag.title }}</span>
        <span v-if="!isAffix(tag)" @click.prevent.stop="closeSelectedTag(tag)" class="tags-view-item-close">
          <close class="el-icon-close" />
        </span>
      </router-link>
    </scroll-pane>
    <ul v-show="visible" :style="{ left: left + 'px', top: top + 'px' }" class="contextmenu">
      <li @click="refreshSelectedTag(selectedTag)"><refresh-right style="width: 1em; height: 1em" /> 刷新页面</li>
      <li v-if="!isAffix(selectedTag)" @click="closeSelectedTag(selectedTag)">
        <close style="width: 1em; height: 1em" /> 关闭当前
      </li>
      <li @click="closeOthersTags"><circle-close style="width: 1em; height: 1em" /> 关闭其他</li>
      <li v-if="!isFirstView()" @click="closeLeftTags">
        <back style="width: 1em; height: 1em" /> 关闭左侧
      </li>
      <li v-if="!isLastView()" @click="closeRightTags">
        <right style="width: 1em; height: 1em" /> 关闭右侧
      </li>
      <li @click="closeAllTags(selectedTag)"><circle-close style="width: 1em; height: 1em" /> 全部关闭</li>
    </ul>
  </div>
</template>

<script setup lang="ts">
import ScrollPane from './ScrollPane.vue';
import { getNormalPath } from '@/utils/ruoyi';
import { useSettingsStore } from '@/store/modules/settings';
import { usePermissionStore } from '@/store/modules/permission';
import { useTagsViewStore } from '@/store/modules/tagsView';
import { RouteRecordRaw, RouteLocationNormalized } from 'vue-router';

const visible = ref(false);
const top = ref(0);
const left = ref(0);
const selectedTag = ref<RouteLocationNormalized>();
const affixTags = ref<RouteLocationNormalized[]>([]);
const scrollPaneRef = ref<InstanceType<typeof ScrollPane>>();

const { proxy } = getCurrentInstance() as ComponentInternalInstance;
const route = useRoute();
const router = useRouter();

const visitedViews = computed(() => useTagsViewStore().getVisitedViews());
const routes = computed(() => usePermissionStore().getRoutes());
const theme = computed(() => useSettingsStore().theme);
const tagsIcon = computed(() => useSettingsStore().tagsIcon)

watch(route, () => {
  addTags();
  moveToCurrentTag();
});
watch(visible, (value) => {
  if (value) {
    document.body.addEventListener('click', closeMenu);
  } else {
    document.body.removeEventListener('click', closeMenu);
  }
});

const isActive = (r: RouteLocationNormalized): boolean => {
  return r.path === route.path;
};
const activeStyle = (tag: RouteLocationNormalized) => {
  if (!isActive(tag)) return {};
  return {
    'background-color': 'color-mix(in srgb, var(--el-color-primary) 15%, transparent)',
    'border-color': 'color-mix(in srgb, var(--el-color-primary) 15%, transparent)',
    'color': 'var(--el-menu-active-color)'
  };
};
const isAffix = (tag: RouteLocationNormalized) => {
  return tag?.meta && tag?.meta?.affix;
};
const isFirstView = () => {
  try {
    return selectedTag.value.fullPath === '/index' || selectedTag.value.fullPath === visitedViews.value[1].fullPath;
  } catch (err) {
    return false;
  }
};
const isLastView = () => {
  try {
    return selectedTag.value.fullPath === visitedViews.value[visitedViews.value.length - 1].fullPath;
  } catch (err) {
    return false;
  }
};
const filterAffixTags = (routes: RouteRecordRaw[], basePath = '') => {
  let tags: RouteLocationNormalized[] = [];

  routes.forEach((route) => {
    if (route.meta && route.meta.affix) {
      const tagPath = getNormalPath(basePath + '/' + route.path);
      tags.push({
        hash: '',
        matched: [],
        params: undefined,
        query: undefined,
        redirectedFrom: undefined,
        fullPath: tagPath,
        path: tagPath,
        name: route.name as string,
        meta: { ...route.meta }
      });
    }
    if (route.children) {
      const tempTags = filterAffixTags(route.children, route.path);
      if (tempTags.length >= 1) {
        tags = [...tags, ...tempTags];
      }
    }
  });
  return tags;
};
const initTags = () => {
  const res = filterAffixTags(routes.value);
  affixTags.value = res;
  for (const tag of res) {
    // Must have tag name
    if (tag.name) {
      useTagsViewStore().addVisitedView(tag);
    }
  }
};
const addTags = () => {
  const { name } = route;
  if (route.query.title) {
    route.meta.title = route.query.title as string;
  }
  if (name) {
    useTagsViewStore().addView(route as any);
  }
};
const moveToCurrentTag = () => {
  nextTick(() => {
    for (const r of visitedViews.value) {
      if (r.path === route.path) {
        scrollPaneRef.value?.moveToTarget(r);
        // when query is different then update
        if (r.fullPath !== route.fullPath) {
          useTagsViewStore().updateVisitedView(route);
        }
      }
    }
  });
};
const refreshSelectedTag = (view: RouteLocationNormalized) => {
  proxy?.$tab.refreshPage(view);
  if (route.meta.link) {
    useTagsViewStore().delIframeView(route);
  }
};
const closeSelectedTag = (view: RouteLocationNormalized) => {
  proxy?.$tab.closePage(view).then(({ visitedViews }: any) => {
    if (isActive(view)) {
      toLastView(visitedViews, view);
    }
  });
};
const closeRightTags = () => {
  proxy?.$tab.closeRightPage(selectedTag.value).then((visitedViews: RouteLocationNormalized[]) => {
    if (!visitedViews.find((i: RouteLocationNormalized) => i.fullPath === route.fullPath)) {
      toLastView(visitedViews);
    }
  });
};
const closeLeftTags = () => {
  proxy?.$tab.closeLeftPage(selectedTag.value).then((visitedViews: RouteLocationNormalized[]) => {
    if (!visitedViews.find((i: RouteLocationNormalized) => i.fullPath === route.fullPath)) {
      toLastView(visitedViews);
    }
  });
};
const closeOthersTags = () => {
  router.push(selectedTag.value).catch(() => { });
  proxy?.$tab.closeOtherPage(selectedTag.value).then(() => {
    moveToCurrentTag();
  });
};
const closeAllTags = (view: RouteLocationNormalized) => {
  proxy?.$tab.closeAllPage().then(({ visitedViews }) => {
    if (affixTags.value.some((tag) => tag.path === route.path)) {
      return;
    }
    toLastView(visitedViews, view);
  });
};
const toLastView = (visitedViews: RouteLocationNormalized[], view?: RouteLocationNormalized) => {
  const latestView = visitedViews.slice(-1)[0];
  if (latestView) {
    router.push(latestView.fullPath as string);
  } else {
    // now the default is to redirect to the home page if there is no tags-view,
    // you can adjust it according to your needs.
    if (view?.name === 'Dashboard') {
      // to reload home page
      router.replace({ path: '/redirect' + view?.fullPath });
    } else {
      router.push('/');
    }
  }
};
const openMenu = (tag: RouteLocationNormalized, e: MouseEvent) => {
  const menuMinWidth = 105;
  const offsetLeft = proxy?.$el.getBoundingClientRect().left || 0;
  const offsetTop = proxy?.$el.getBoundingClientRect().top || 0;
  const offsetWidth = proxy?.$el.offsetWidth || 0;
  const maxLeft = offsetWidth - menuMinWidth;
  const l = e.clientX - offsetLeft + 15;
  const t = e.clientY - offsetTop;

  if (l > maxLeft) {
    left.value = maxLeft;
  } else {
    left.value = l;
  }

  top.value = t;
  visible.value = true;
  selectedTag.value = tag;
};
const closeMenu = () => {
  visible.value = false;
};
const handleScroll = () => {
  closeMenu();
};

onMounted(() => {
  initTags();
  addTags();
});
</script>

<style lang="scss" scoped>
.tags-view-container {
  height: 34px;
  width: 100%;
  background-color: var(--el-bg-color);
  border-top: 1px solid var(--el-border-color-light);
  box-shadow: 0 1px 4px rgba(113, 128, 165, .1);
  position: relative;
  z-index: 10;

  .tags-view-wrapper {
    height: 100%;
    white-space: nowrap;
    position: relative;
    padding: 4px 10px 0 10px;

    .tags-view-item {
      display: inline-flex;
      align-items: center;
      position: relative;
      cursor: pointer;
      height: 26px;
      color: var(--tags-view-item-color, #495060);
      padding: 0 16px 0 12px;
      font-size: 12px;
      margin: 0 2px;
      background-color: var(--tags-view-item-bg, var(--el-bg-color));
      border: 1px solid var(--tags-view-item-border, var(--el-border-color-light));
      border-radius: 4px 4px 0 0;
      transition: all 0.3s cubic-bezier(0.645, 0.045, 0.355, 1);
      z-index: 1;
      overflow: hidden;

      &:hover {
        color: var(--el-color-primary);
        background-color: var(--tags-view-hover-bg, var(--el-fill-color-light));
        z-index: 2;
      }

      &:first-of-type {
        margin-left: 5px;
      }

      &:last-of-type {
        margin-right: 5px;
      }

      &.active {
        background-color: color-mix(in srgb, var(--el-color-primary) 15%, transparent);
        color: var(--el-menu-active-color);
        border-color: color-mix(in srgb, var(--el-color-primary) 15%, transparent);
        border-bottom-color: transparent;
        z-index: 3;
        margin-bottom: -1px;
      }
    }
  }

  .tags-view-item-title {
    margin-left: 4px;
    margin-right: 3px;
    position: relative;
    z-index: 1;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    max-width: 150px;
  }

  .tags-view-item-close {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 16px;
    height: 16px;
    border-radius: 50%;
    margin-left: 5px;
    position: relative;
    z-index: 1;

    .el-icon-close {
      width: 1em;
      height: 1em;
      transition: all 0.3s cubic-bezier(0.645, 0.045, 0.355, 1);
      transform-origin: center center;

      &:hover {
        background-color: #b4bccc;
        color: #fff;
        width: 12px !important;
        height: 12px !important;
      }
    }
  }

  .contextmenu {
    margin: 0;
    background: var(--el-bg-color);
    z-index: 3000;
    position: absolute;
    list-style-type: none;
    padding: 5px 0;
    border-radius: 4px;
    font-size: 12px;
    font-weight: 400;
    box-shadow: 2px 2px 3px 0 rgba(0, 0, 0, 0.3);
    border: 1px solid var(--el-border-color-light);

    li {
      margin: 0;
      padding: 7px 16px;
      cursor: pointer;
      display: flex;
      align-items: center;
      gap: 8px;

      &:hover {
        background: var(--el-fill-color-light);
      }
    }
  }
}

// 定义浅色和深色主题下的变量
:root {
  --tags-view-item-bg: var(--el-bg-color);
  --tags-view-item-border: var(--el-border-color-light);
  --tags-view-item-color: #495060;
  --tags-view-hover-bg: var(--el-fill-color-light);
  --tags-view-active-bg: color-mix(in srgb, var(--el-color-primary) 15%, transparent);
  --tags-view-active-border-color: color-mix(in srgb, var(--el-color-primary) 15%, transparent);
  --tags-view-active-color: var(--el-menu-active-color);
}

html.dark {
  --tags-view-item-bg: var(--el-bg-color);
  --tags-view-item-border: var(--el-border-color-light);
  --tags-view-item-color: #e5eaf3;
  --tags-view-hover-bg: var(--el-fill-color-dark);
  --tags-view-active-bg: rgba(255, 255, 255, 0.05);
  --tags-view-active-border-color: rgba(255, 255, 255, 0.05);
  --tags-view-active-color: #fff;
}
</style>

<style lang="scss">
//reset element css of el-icon-close
.tags-view-wrapper {
  .tags-view-item {
    .tags-view-item-close {
      .el-icon-close {
        &:before {
          transform: scale(0.6);
          display: inline-block;
        }
      }
    }
  }
}
</style>
