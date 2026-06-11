<script lang="ts" setup>
import { Setting } from "@element-plus/icons-vue"

const show = ref(false)
const { t } = useI18n()
</script>

<template>
  <el-tooltip :content="t('布局配置')" placement="bottom" effect="dark">
    <div class="handle-button" aria-label="Settings" @click="show = true">
      <el-icon :size="20">
        <Setting />
      </el-icon>
    </div>
  </el-tooltip>
  <el-drawer v-model="show" size="300px" :with-header="false">
    <slot />
  </el-drawer>
</template>

<style lang="scss" scoped>
.handle-button {
  width: 40px;
  height: 48px;
  cursor: pointer;
  color: var(--v3-navigationbar-text-color);
  display: flex;
  align-items: center;
  justify-content: center;

  &:hover {
    background-color: rgb(0 0 0 / 6%);
  }
}
</style>
