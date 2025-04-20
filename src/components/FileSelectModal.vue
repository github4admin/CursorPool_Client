<script setup lang="ts">
  import { NModal, NSpace, NIcon, useMessage, NButton, useThemeVars } from 'naive-ui'
  import { useCursorStore } from '@/stores'
  import { watch, computed } from 'vue'
  import { useTheme } from '../composables/theme'
  import {
    DocumentOutline,
    InformationCircleOutline,
    RefreshOutline,
    Warning,
  } from '@vicons/ionicons5'
  import { open } from '@tauri-apps/plugin-shell'

  // 获取主题变量
  const { isDarkMode } = useTheme()
  const themeVars = useThemeVars()

  // 计算主题相关样式
  const cardStyle = computed(() => ({
    backgroundColor: isDarkMode.value ? themeVars.value.cardColor : '#f5f5f7',
  }))

  // 获取 Cursor Store
  const cursorStore = useCursorStore()
  // 获取消息组件
  const message = useMessage()

  // 监听文件选择状态变化，提供消息反馈
  watch(
    () => cursorStore.showSelectFileModal,
    (newValue) => {
      if (newValue) {
        // 模态框显示时重置错误
        cursorStore.fileSelectError = ''
      }
    },
  )

  // 处理文件选择
  const handleSelectPath = async () => {
    await cursorStore.handleSelectCursorPath()

    // 处理成功状态
    if (!cursorStore.showSelectFileModal && !cursorStore.fileSelectError) {
      message.success('文件选择成功，系统已找到并保存Cursor路径')

      // 检查是否有待处理操作
      if (cursorStore.pendingAction) {
        message.loading(`正在执行${cursorStore.pendingAction.type}操作...`)

        // 等待操作完成
        setTimeout(() => {
          if (!cursorStore.fileSelectError) {
            if (cursorStore.pendingAction?.type === 'applyHook') {
              message.success('Hook应用成功！')
            } else if (cursorStore.pendingAction?.type === 'restoreHook') {
              message.success('Hook恢复成功！')
            }
          }
        }, 1000)
      }
    } else if (cursorStore.fileSelectError) {
      // 显示错误消息
      message.error(cursorStore.fileSelectError)
    }
  }

  const handleOpenDocs = () => {
    open('https://docs.52ai.org/troubleshooting/windows/injection')
  }

  // 处理注入正在运行的Cursor
  const handleInjectRunningCursor = async () => {
    message.loading('正在尝试注入运行中的Cursor...')
    try {
      await cursorStore.injectRunningCursor()
      message.success('成功注入正在运行的Cursor!')

      // 关闭模态框
      cursorStore.showSelectFileModal = false
    } catch (error) {
      // 显示错误消息
      const errorMsg = error instanceof Error ? error.message : String(error)
      message.error(errorMsg || '注入失败，请尝试手动选择文件')
      cursorStore.fileSelectError = errorMsg || '注入失败，请尝试手动选择文件'
    }
  }

  // 关闭模态框
  const closeModal = () => {
    cursorStore.showSelectFileModal = false
  }
</script>

<template>
  <!-- 文件选择模态框 -->
  <n-modal
    v-model:show="cursorStore.showSelectFileModal"
    preset="card"
    title="选择Cursor路径"
    :bordered="false"
    style="width: 550px"
    @close="closeModal"
  >
    <n-space vertical>
      <div
        class="text-base"
        :style="{ color: themeVars.textColor1 }"
      >
        需要设置Cursor路径以完成注入操作，请选择以下方式之一：
      </div>

      <div style="margin-top: 12px">
        <n-space
          vertical
          :size="12"
        >
          <div
            class="option-item"
            :style="cardStyle"
          >
            <div
              class="option-title"
              :style="{ color: themeVars.textColor1 }"
            >
              <n-icon
                size="20"
                class="option-icon"
                :color="themeVars.primaryColor"
              >
                <DocumentOutline />
              </n-icon>
              <b>选择Cursor程序</b>
            </div>
            <div
              class="option-desc"
              :style="{ color: themeVars.textColor3 }"
            >
              直接选择cursor.exe程序文件，系统会自动查找main.js的位置
            </div>
          </div>

          <div
            class="option-item"
            :style="cardStyle"
          >
            <div
              class="option-title"
              :style="{ color: themeVars.textColor1 }"
            >
              <n-icon
                size="20"
                class="option-icon"
                :color="themeVars.primaryColor"
              >
                <RefreshOutline />
              </n-icon>
              <b>注入运行中的Cursor</b>
            </div>
            <div
              class="option-desc"
              :style="{ color: themeVars.textColor3 }"
            >
              检测当前正在运行的Cursor实例并尝试直接注入
            </div>
          </div>

          <div
            class="option-item"
            :style="cardStyle"
          >
            <div
              class="option-title"
              :style="{ color: themeVars.textColor1 }"
            >
              <n-icon
                size="20"
                class="option-icon"
                :color="themeVars.primaryColor"
              >
                <InformationCircleOutline />
              </n-icon>
              <b>查看帮助文档</b>
            </div>
            <div
              class="option-desc"
              :style="{ color: themeVars.textColor3 }"
            >
              如果你不确定如何操作，可以查看详细的帮助文档
            </div>
          </div>
        </n-space>
      </div>

      <div
        v-if="cursorStore.fileSelectError"
        :class="['error-container', isDarkMode ? 'dark' : 'light']"
      >
        <n-icon
          size="16"
          style="margin-right: 6px"
        >
          <Warning />
        </n-icon>
        {{ cursorStore.fileSelectError }}
      </div>

      <div style="margin-top: 20px; display: flex; justify-content: flex-end; gap: 12px">
        <n-button
          type="primary"
          @click="handleOpenDocs"
        >
          打开文档
        </n-button>
        <n-button
          type="primary"
          :loading="cursorStore.operationLoading"
          @click="handleInjectRunningCursor"
        >
          注入正在运行的Cursor
        </n-button>
        <n-button
          type="primary"
          :loading="cursorStore.fileSelectLoading"
          @click="handleSelectPath"
        >
          选择文件
        </n-button>
      </div>
    </n-space>
  </n-modal>
</template>

<style scoped>
  .option-item {
    padding: 12px;
    border-radius: 6px;
    transition: background-color 0.2s;
    border: 1px solid rgba(0, 0, 0, 0.1);
  }

  .option-item:hover {
    filter: brightness(0.95);
  }

  .option-title {
    display: flex;
    align-items: center;
    font-size: 14px;
    margin-bottom: 6px;
  }

  .option-icon {
    margin-right: 8px;
  }

  .option-desc {
    margin-left: 28px;
    font-size: 13px;
  }

  .error-container {
    margin-top: 16px;
    padding: 10px;
    border-radius: 4px;
    display: flex;
    align-items: center;
  }

  .error-container.light {
    color: #d03050;
    background-color: rgba(208, 48, 80, 0.1);
  }

  .error-container.dark {
    color: #e88080;
    background-color: rgba(232, 128, 128, 0.1);
  }
</style>
