<template>
  <div class="setting">
    <div class="menu-item">
      <el-switch
          v-model="isDark"
          style="--el-switch-on-color:#18222c"
      >
        <template #active-action>
          <el-icon>
            <Moon/>
          </el-icon>
        </template>
        <template #inactive-action>
          <el-icon>
            <Sunny color="#000"/>
          </el-icon>
        </template>
      </el-switch>
    </div>
    
    <!-- Динамический переключатель языка -->
    <el-dropdown class="menu-item" @command="changeLang">
      <div class="title lang-trigger">
        <span class="lang-flag">{{ currentLangFlag }}</span>
        <span class="lang-name">{{ currentLangName }}</span>
        <el-icon>
          <ArrowDown />
        </el-icon>
      </div>
      <template #dropdown>
        <el-dropdown-menu>
          <el-dropdown-item 
            v-for="(v, k) in appStore.setting.langs" 
            :key="k" 
            :command="k"
            :class="{ 'is-active': appStore.setting.lang === k }"
          >
            <span class="lang-flag">{{ getLangFlag(k) }}</span>
            <span>{{ v.name }}</span>
          </el-dropdown-item>
        </el-dropdown-menu>
      </template>
    </el-dropdown>
    
    <el-dropdown class="menu-item">
      <div class="title">
        <span class="nickname">{{ user.username }}</span>
        <el-icon>
          <ArrowDown />
        </el-icon>
      </div>
      <template #dropdown>
        <el-dropdown-menu>
          <el-dropdown-item @click="showChangePwd">{{ T('ChangePassword') }}</el-dropdown-item>
          <el-dropdown-item @click="logout">{{ T('Logout') }}</el-dropdown-item>
        </el-dropdown-menu>
      </template>
    </el-dropdown>
    <changePwdDialog v-model:visible="changePwdVisible"></changePwdDialog>
  </div>
</template>

<script setup>
import { useUserStore } from '@/store/user'
import { useAppStore } from '@/store/app'
import changePwdDialog from '@/components/changePwdDialog.vue'
import { ref, computed } from 'vue'
import { T } from '@/utils/i18n'
import { useDark } from '@vueuse/core'
import { Sunny, Moon, ArrowDown } from '@element-plus/icons'

const userStore = useUserStore()
const user = userStore
const appStore = useAppStore()

// Флаги для языков
const langFlags = {
  'ru': '🇷🇺',
  'en': '🇬🇧',
  'zh-CN': '🇨🇳',
  'fr': '🇫🇷',
  'es': '🇪🇸',
  'ko': '🇰🇷',
  'zh-TW': '🇹🇼'
}

// Названия языков
const langNames = {
  'ru': 'Русский',
  'en': 'English',
  'zh-CN': '中文',
  'fr': 'Français',
  'es': 'Español',
  'ko': '한국어',
  'zh-TW': '中文繁体'
}

const currentLangFlag = computed(() => {
  return langFlags[appStore.setting.lang] || '🌐'
})

const currentLangName = computed(() => {
  return langNames[appStore.setting.lang] || appStore.setting.lang
})

const getLangFlag = (lang) => {
  return langFlags[lang] || '🌐'
}

const logout = () => {
  userStore.logout()
  window.location.reload()
}

const changePwdVisible = ref(false)
const showChangePwd = () => {
  changePwdVisible.value = true
}

const changeLang = (lang) => {
  appStore.changeLang(lang)
}

const isDark = useDark()
</script>

<style lang="scss" scoped>
.setting {
  margin-left: auto;
  display: flex;
  align-items: center;
  justify-content: space-around;

  .menu-item {
    margin-left: 15px;

    * {
      outline: none;
    }
  }

  .title {
    color: #fff;
    display: flex;
    align-items: center;
    justify-content: space-around;
    cursor: pointer;
    
    .nickname {
      padding: 0 10px;
    }
  }
  
  .lang-trigger {
    gap: 4px;
    
    .lang-flag {
      font-size: 18px;
    }
    
    .lang-name {
      font-size: 14px;
    }
  }
}

:deep(.el-dropdown-menu__item) {
  display: flex;
  align-items: center;
  gap: 8px;
  
  .lang-flag {
    font-size: 16px;
  }
}

:deep(.is-active) {
  background-color: var(--el-color-primary-light-9);
  color: var(--el-color-primary);
}
</style>
