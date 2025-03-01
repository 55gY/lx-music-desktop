<template lang="pug">
#container(v-if="isProd && !isLinux" :class="theme" @mouseenter="enableIgnoreMouseEvents" @mouseleave="dieableIgnoreMouseEvents")
  core-aside#left
  #right
    core-toolbar#toolbar
    core-view#view
    core-player#player
  core-icons
  material-version-modal(v-show="version.showModal")
#container(v-else :class="theme")
  core-aside#left
  #right
    core-toolbar#toolbar
    core-view#view
    core-player#player
  core-icons
  material-version-modal(v-show="version.showModal")
</template>

<script>
import dnscache from 'dnscache'
import { mapMutations, mapGetters, mapActions } from 'vuex'
import { rendererOn, rendererSend } from '../common/ipc'
import { isLinux } from '../common/utils'
import music from './utils/music'
import { throttle, openUrl } from './utils'
window.ELECTRON_DISABLE_SECURITY_WARNINGS = process.env.ELECTRON_DISABLE_SECURITY_WARNINGS
dnscache({
  enable: true,
  ttl: 21600,
  cachesize: 1000,
})

export default {
  data() {
    return {
      isProd: process.env.NODE_ENV === 'production',
      isLinux,
      globalObj: {
        apiSource: 'test',
        proxy: {},
      },
      updateTimeout: null,
    }
  },
  computed: {
    ...mapGetters(['electronStore', 'setting', 'theme', 'version']),
    ...mapGetters('list', ['defaultList', 'loveList']),
    ...mapGetters('download', {
      downloadList: 'list',
      downloadStatus: 'downloadStatus',
    }),
  },
  created() {
    this.saveSetting = throttle(n => {
      this.electronStore.set('setting', n)
    })
    this.saveDefaultList = throttle(n => {
      this.electronStore.set('list.defaultList', n)
    }, 500)
    this.saveLoveList = throttle(n => {
      this.electronStore.set('list.loveList', n)
    }, 500)
    this.saveDownloadList = throttle(n => {
      this.electronStore.set('download.list', n)
    }, 1000)
  },
  mounted() {
    document.body.classList.add(this.isLinux ? 'noTransparent' : 'transparent')
    this.init()
  },
  watch: {
    setting: {
      handler(n) {
        this.saveSetting(n)
      },
      deep: true,
    },
    defaultList: {
      handler(n) {
        this.saveDefaultList(n)
      },
      deep: true,
    },
    loveList: {
      handler(n) {
        this.saveLoveList(n)
      },
      deep: true,
    },
    downloadList: {
      handler(n) {
        this.saveDownloadList(n)
      },
      deep: true,
    },
    'globalObj.apiSource'(n) {
      if (n != this.setting.apiSource) {
        this.setSetting(Object.assign({}, this.setting, {
          apiSource: n,
        }))
      }
    },
  },
  methods: {
    ...mapActions(['getVersionInfo']),
    ...mapMutations(['setNewVersion', 'setVersionModalVisible', 'setDownloadProgress']),
    ...mapMutations('list', ['initList']),
    ...mapMutations('download', ['updateDownloadList']),
    ...mapMutations(['setSetting']),
    init() {
      document.body.addEventListener('click', this.handleBodyClick, true)
      if (this.isProd && !isLinux) {
        document.body.addEventListener('mouseenter', this.dieableIgnoreMouseEvents)
        document.body.addEventListener('mouseleave', this.enableIgnoreMouseEvents)
      }
      rendererOn('update-available', (e, info) => {
        // this.showUpdateModal(true)
        // console.log(info)
        this.setVersionModalVisible({ isDownloading: true })
        this.getVersionInfo().catch(() => ({
          version: info.version,
          desc: info.releaseNotes,
        })).then(body => {
          // console.log(body)
          this.setNewVersion(body)
          this.$nextTick(() => {
            this.setVersionModalVisible({ isShow: true })
          })
        })
      })
      rendererOn('update-error', (event, err) => {
        // console.log(err)
        this.clearUpdateTimeout()
        this.setVersionModalVisible({ isError: true })
        this.$nextTick(() => {
          this.showUpdateModal()
        })
      })
      rendererOn('update-progress', (event, progress) => {
        // console.log(progress)
        this.setDownloadProgress(progress)
      })
      rendererOn('update-downloaded', info => {
        // console.log(info)
        this.clearUpdateTimeout()
        this.setVersionModalVisible({ isDownloaded: true })
        this.$nextTick(() => {
          this.showUpdateModal()
        })
      })
      rendererOn('update-not-available', () => {
        this.clearUpdateTimeout()
        this.setNewVersion({
          version: this.version.version,
        })
      })
      // 更新超时定时器
      this.updateTimeout = setTimeout(() => {
        this.updateTimeout = null
        this.setVersionModalVisible({ isTimeOut: true })
        this.$nextTick(() => {
          this.showUpdateModal()
        })
      }, 60 * 30 * 1000)

      this.initData()
      this.globalObj.apiSource = this.setting.apiSource
      this.globalObj.proxy = Object.assign({}, this.setting.network.proxy)
      window.globalObj = this.globalObj

      // 初始化音乐sdk
      music.init()
    },
    enableIgnoreMouseEvents() {
      if (isLinux) return
      rendererSend('setIgnoreMouseEvents', false)
      // console.log('content enable')
    },
    dieableIgnoreMouseEvents() {
      if (isLinux) return
      // console.log('content disable')
      rendererSend('setIgnoreMouseEvents', true)
    },

    initData() { // 初始化数据
      this.initPlayList() // 初始化播放列表
      this.initDownloadList() // 初始化下载列表
    },
    initPlayList() {
      let defaultList = this.electronStore.get('list.defaultList')
      let loveList = this.electronStore.get('list.loveList')
      this.initList({ defaultList, loveList })
    },
    initDownloadList() {
      let downloadList = this.electronStore.get('download.list')
      if (downloadList) {
        downloadList.forEach(item => {
          if (item.status == this.downloadStatus.RUN || item.status == this.downloadStatus.WAITING) {
            item.status = this.downloadStatus.PAUSE
            item.statusText = '暂停下载'
          }
        })
        this.updateDownloadList(downloadList)
      }
    },
    showUpdateModal() {
      (this.version.newVersion && this.version.newVersion.history ? Promise.resolve(this.version.newVersion) : this.getVersionInfo().then(body => {
        this.setNewVersion(body)
        return body
      })).catch(() => {
        this.setVersionModalVisible({ isUnknow: true })
        let result = {
          version: '0.0.0',
          desc: null,
        }
        this.setNewVersion(result)
        return result
      }).then(result => {
        if (result.version === this.version.version) return
        // console.log(this.version)
        this.$nextTick(() => {
          this.setVersionModalVisible({ isShow: true })
        })
      })
    },
    clearUpdateTimeout() {
      if (!this.updateTimeout) return
      clearTimeout(this.updateTimeout)
      this.updateTimeout = null
    },
    handleBodyClick(event) {
      if (event.target.tagName != 'A') return
      if (event.target.host == window.location.host) return
      event.preventDefault()
      if (/^https?:\/\//.test(event.target.href)) openUrl(event.target.href)
    },
  },
  beforeDestroy() {
    this.clearUpdateTimeout()
    if (this.isProd) {
      document.body.removeEventListener('mouseenter', this.dieableIgnoreMouseEvents)
      document.body.removeEventListener('mouseleave', this.enableIgnoreMouseEvents)
    }
    document.body.removeEventListener('click', this.handleBodyClick)
  },
}
</script>

<style lang="less">
@import './assets/styles/index.less';
@import './assets/styles/layout.less';

body {
  user-select: none;
  height: 100vh;
  box-sizing: border-box;
}

.transparent {
  padding: @shadow-app;
  #container {
    box-shadow: 0 0 @shadow-app rgba(0, 0, 0, 0.5);
    border-radius: 4px;
    background-color: transparent;
  }
}
.noTransparent {
  background-color: #fff;
}

#container {
  position: relative;
  display: flex;
  height: 100%;
  overflow: hidden;
  background: @color-theme-bgimg @color-theme-bgposition no-repeat;
  background-size: @color-theme-bgsize;
  transition: background-color @transition-theme;
  background-color: @color-theme;
}

#left {
  flex: none;
  width: @width-app-left;
}
#right {
  flex: auto;
  display: flex;
  flex-flow: column nowrap;
  transition: background-color @transition-theme;
  background-color: @color-theme_2;
}
#toolbar, #player {
  flex: none;
}
#view {
  flex: auto;
  height: 0;
}

each(@themes, {
  #container.@{value} {
    background-color: ~'@{color-@{value}-theme}';
    background-image: ~'@{color-@{value}-theme-bgimg}';
    background-size: ~'@{color-@{value}-theme-bgsize}';
    background-position: ~'@{color-@{value}-theme-bgposition}';
    #right {
      background-color: ~'@{color-@{value}-theme_2}';
    }
  }
})
</style>

