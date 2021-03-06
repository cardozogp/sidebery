<template lang="pug">
.Tab(:is-active="tab.active"
  :data-status="tab.status"
  :data-no-fav="!favicon"
  :data-audible="tab.audible"
  :data-muted="tab.mutedInfo.muted"
  :data-pinned="tab.pinned"
  :is-selected="selected"
  :discarded="tab.discarded"
  :updated="updated"
  :is-parent="tab.isParent"
  :folded="tab.folded"
  :invisible="tab.invisible"
  :lvl="tab.lvl"
  :close-btn="$store.state.showTabRmBtn"
  :style="{ transform: 'translateY(' + position + 'px)' }"
  :title="tooltip"
  @contextmenu.prevent.stop=""
  @mousedown="onMouseDown"
  @mouseup="onMouseUp"
  @mouseleave="onMouseLeave"
  @dblclick.prevent.stop="onDoubleClick"): .lvl-wrapper
  .loaded-fx
  .drag-layer(draggable="true"
    @dragstart="onDragStart"
    @dragenter="onDragEnter"
    @dragleave="onDragLeave")
  .audio(@mousedown.stop="", @click="$store.dispatch('remuteTabs', [tab.id])")
    svg.-loud: use(xlink:href="#icon_loud")
    svg.-mute: use(xlink:href="#icon_mute")
  .fav(:loading="loading")
    .placeholder: svg: use(:xlink:href="favPlaceholder")
    img(:src="favicon", @load.passive="onFaviconLoad")
    .exp(@dblclick.prevent.stop="", @mousedown.stop="onExp"): svg: use(xlink:href="#icon_expand")
    .update-badge
    .ok-badge
      svg: use(xlink:href="#icon_ok")
    .err-badge
      svg: use(xlink:href="#icon_err")
    .loading-spinner
      each n in [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12]
        .spinner-stick(class='spinner-stick-' + n)
    .child-count(v-if="childCount && tab.folded") {{childCount}}
  .close(v-if="$store.state.showTabRmBtn", @mousedown.stop="onCloseClick", @mouseup.stop="")
    svg: use(xlink:href="#icon_remove")
  .t-box
    .title {{tab.title}}
    .loading
      svg.-a: use(xlink:href="#icon_load")
      svg.-b: use(xlink:href="#icon_load")
      svg.-c: use(xlink:href="#icon_load")
</template>


<script>
import Store from '../../store'
import State from '../../store.state'
import EventBus from '../../event-bus'

const PNG_RE = /(\.png)([?#].*)?$/i
const JPG_RE = /(\.jpe?g)([?#].*)?$/i
const PDF_RE = /(\.pdf)([?#].*)?$/i
const GROUP_RE = /\/group\/group\.html/

export default {
  props: {
    position: Number,
    childCount: Number,
    tab: {
      type: Object,
      default: () => ({}),
    },
  },

  data() {
    return {
      menu: false,
      loading: false,
      selected: false,
    }
  },

  computed: {
    updated() {
      return !!State.updatedTabs[this.tab.id]
    },

    favicon() {
      if (this.tab.status === 'loading') return State.favicons[this.tab.host]
      else return this.tab.favIconUrl || State.favicons[this.tab.host]
    },

    tooltip() {
      try {
        return `${this.tab.title}\n${decodeURI(this.tab.url)}`
      } catch (err) {
        return `${this.tab.title}\n${this.tab.url}`
      }
    },

    favPlaceholder() {
      if (this.tab.url.startsWith('moz-extension:') && GROUP_RE.test(this.tab.url)) {
        return '#icon_group'
      }
      if (PNG_RE.test(this.tab.url)) return '#icon_png'
      if (JPG_RE.test(this.tab.url)) return '#icon_jpg'
      if (PDF_RE.test(this.tab.url)) return '#icon_pdf'
      if (this.tab.url.startsWith('file:')) return '#icon_local_file'
      if (this.tab.url.startsWith('about:preferences')) return '#icon_pref'
      if (this.tab.url.startsWith('about:addons')) return '#icon_addons'
      if (this.tab.url.startsWith('about:performance')) return '#icon_perf'
      return '#icon_ff'
    },
  },

  created() {
    EventBus.$on('tabLoadingStart', this.loadingStart)
    EventBus.$on('tabLoadingEnd', this.loadingEnd)
    EventBus.$on('tabLoadingOk', this.loadingOk)
    EventBus.$on('tabLoadingErr', this.loadingErr)
    EventBus.$on('tabLoaded', this.onLoaded)
    EventBus.$on('selectTab', this.onTabSelection)
    EventBus.$on('deselectTab', this.onTabDeselection)
    EventBus.$on('openTabMenu', this.onTabMenu)
  },

  beforeDestroy() {
    EventBus.$off('tabLoadingStart', this.loadingStart)
    EventBus.$off('tabLoadingEnd', this.loadingEnd)
    EventBus.$off('tabLoadingOk', this.loadingOk)
    EventBus.$off('tabLoadingErr', this.loadingErr)
    EventBus.$off('tabLoaded', this.onLoaded)
    EventBus.$off('selectTab', this.onTabSelection)
    EventBus.$off('deselectTab', this.onTabDeselection)
    EventBus.$off('openTabMenu', this.onTabMenu)
  },

  methods: {
    /**
     * Double click handler
     */
    onDoubleClick() {
      let dc = State.tabDoubleClick
      if (dc === 'reload') Store.dispatch('reloadTabs', [this.tab.id])
      if (dc === 'duplicate') Store.dispatch('duplicateTabs', [this.tab.id])
      if (dc === 'pin') Store.dispatch('repinTabs', [this.tab.id])
      if (dc === 'mute') Store.dispatch('remuteTabs', [this.tab.id])
      if (dc === 'clear_cookies') Store.dispatch('clearTabsCookies', [this.tab.id])
      if (dc === 'exp' && this.tab.isParent) Store.dispatch('toggleBranch', this.tab.id)
      if (dc === 'new_after') Store.dispatch('createTabAfter', this.tab.id)
    },

    /**
     * Mousedown handler
     */
    onMouseDown(e) {
      if (e.button === 1) {
        this.close()
        e.preventDefault()
        e.stopPropagation()
      }

      if (e.button === 0) {
        // Activate tab (if nothing selected)
        if (!State.selected.length) {
          browser.tabs.update(this.tab.id, { active: true })
        }

        // Long-click action
        this.hodorL = setTimeout(() => {
          if (State.dragNodes) return
          let llc = State.tabLongLeftClick
          if (llc === 'reload') Store.dispatch('reloadTabs', [this.tab.id])
          if (llc === 'duplicate') Store.dispatch('duplicateTabs', [this.tab.id])
          if (llc === 'pin') Store.dispatch('repinTabs', [this.tab.id])
          if (llc === 'mute') Store.dispatch('remuteTabs', [this.tab.id])
          if (llc === 'clear_cookies') Store.dispatch('clearTabsCookies', [this.tab.id])
          if (llc === 'new_after') Store.dispatch('createTabAfter', this.tab.id)
          this.hodorL = null
        }, 250)
      }

      if (e.button === 2) {
        e.preventDefault()
        e.stopPropagation()
        this.$emit('start-selection', {
          type: 'tab',
          clientY: e.clientY,
          ctrlKey: e.ctrlKey,
          id: this.tab.id,
        })
        // Long-click action
        this.hodorR = setTimeout(() => {
          this.$emit('stop-selection')
          let lrc = State.tabLongRightClick
          if (lrc === 'reload') Store.dispatch('reloadTabs', [this.tab.id])
          if (lrc === 'duplicate') Store.dispatch('duplicateTabs', [this.tab.id])
          if (lrc === 'pin') Store.dispatch('repinTabs', [this.tab.id])
          if (lrc === 'mute') Store.dispatch('remuteTabs', [this.tab.id])
          if (lrc === 'clear_cookies') Store.dispatch('clearTabsCookies', [this.tab.id])
          if (lrc === 'new_after') Store.dispatch('createTabAfter', this.tab.id)
          this.hodorR = null
        }, 250)
      }
    },

    /**
     * Handle mouseup event
     */
    onMouseUp(e) {
      if (e.button === 0 && this.hodorL) {
        this.hodorL = clearTimeout(this.hodorL)
      }
      if (e.button === 2 && this.hodorR) {
        this.hodorR = clearTimeout(this.hodorR)

        // Select this tab
        if (this.tab.isParent && this.tab.folded) {
        // Select whole branch if tab is folded
          Store.commit('resetSelection')
          const toSelect = [this.tab.id]
          for (let tab of State.tabs) {
            if (toSelect.includes(tab.parentId)) toSelect.push(tab.id)
          }
          toSelect.map(id => EventBus.$emit('selectTab', id))
          State.selected = [...toSelect]
          Store.dispatch('openCtxMenu', { el: this.$el, node: this.tab })
        } else {
        // Select only current tab 
          Store.commit('closeCtxMenu')
          State.selected = [this.tab.id]
          this.selected = true
        }
      }
    },

    /**
     * Handle tab-selection event
     */
    onTabSelection(id) {
      if (this.tab.id === id) {
        this.selected = true
        this.hodorR = clearTimeout(this.hodorR)
      }
    },
  
    /**
     * Handle tab-deselection event
     */
    onTabDeselection(id) {
      if (!id) this.selected = false
      if (id && this.tab.id === id) this.selected = false
    },

    /**
     * Open tab[s] menu
     */
    onTabMenu(id) {
      if (id !== this.tab.id) return
      if (this.tab.invisible) return
      Store.dispatch('openCtxMenu', { el: this.$el, node: this.tab })
    },

    /**
     * Handle mouseleave event
     */
    onMouseLeave() {
      if (this.hodorL) this.hodorL = clearTimeout(this.hodorL)
      if (this.hodorR) this.hodorR = clearTimeout(this.hodorR)
    },

    /**
     * Handle dragstart event.
     */
    onDragStart(e) {
      if (!this.hodorL) return

      // Hide context menu (if any)
      if (State.ctxMenu) State.ctxMenu = null

      // Check what to drag
      const toDrag = [this.tab.id]
      const tabsToDrag = []
      if (!State.selected.length) tabsToDrag.push(this.tab)
      for (let tab of State.tabs) {
        if (toDrag.includes(tab.parentId)) {
          toDrag.push(tab.id)
          tabsToDrag.push(tab)
          continue
        }
        if (State.selected.includes(tab.id)) {
          toDrag.push(tab.id)
          tabsToDrag.push(tab)
        }
      }

      // Clear selected elements
      State.selected = []

      // Set drag info
      e.dataTransfer.setData('text/x-moz-text-internal', this.tab.url)
      e.dataTransfer.setData('text/uri-list', this.tab.url)
      e.dataTransfer.setData('text/plain', this.tab.url)
      e.dataTransfer.effectAllowed = 'move'
      const dragData = tabsToDrag.map(t => {
        return {
          ...JSON.parse(JSON.stringify(t)),
          type: 'tab',
          ctx: t.cookieStoreId,
          windowId: State.windowId,
          panel: State.panelIndex,
          incognito: State.private,
        }
      })
      EventBus.$emit('dragStart', dragData)
      Store.dispatch('broadcast', {
        name: 'outerDragStart',
        arg: dragData,
      })
    },

    /**
     * Handle dragenter event
     */
    onDragEnter() {
      if (this.tab.invisible) return
      if (this.dragEnterTimeout) clearTimeout(this.dragEnterTimeout)
      this.dragEnterTimeout = setTimeout(() => {
        if (!State.dragNodes) return
        browser.tabs.update(this.tab.id, { active: true })
        this.dragEnterTimeout = null
      }, 500)
    },

    /**
     * Handle dragleave event
     */
    onDragLeave() {
      if (this.dragEnterTimeout) {
        clearTimeout(this.dragEnterTimeout)
        this.dragEnterTimeout = null
      }
    },

    /**
     * If favicon is just url to some image,
     * wait until it is loaded, convert to base64 and
     * store result to cache.
     */
    onFaviconLoad(e) {
      if (!this.favicon) return
      if (this.favicon.startsWith('http')) {
        let canvas = document.createElement('canvas')
        let ctx = canvas.getContext('2d')
        canvas.width = e.target.naturalWidth
        canvas.height = e.target.naturalHeight
        ctx.imageSmoothingEnabled = false
        ctx.drawImage(e.target, 0, 0, e.target.naturalWidth, e.target.naturalHeight)
        let base64 = canvas.toDataURL('image/png')
        let hn = this.tab.url.split('/')[2]
        if (!hn) return
        Store.dispatch('setFavicon', { hostname: hn, icon: base64 })
      }
    },

    /**
     * Handle mousedown event on expand button
     */
    onExp(e) {
      // Fold/Expand branch
      if (e.button === 0) Store.dispatch('toggleBranch', this.tab.id)

      // Select whole branch and show menu
      if (e.button === 2) {
        Store.commit('resetSelection')
        const toSelect = [this.tab.id]
        for (let tab of State.tabs) {
          if (toSelect.includes(tab.parentId)) toSelect.push(tab.id)
        }
        toSelect.map(id => EventBus.$emit('selectTab', id))
        State.selected = [...toSelect]
        Store.dispatch('openCtxMenu', { el: this.$el, node: this.tab })
      }
    },

    /**
     * Handle click on close btn
     */
    onCloseClick(e) {
      if (e.button === 0) this.close()
      if (e.button === 1) this.close()
      if (e.button === 2) this.closeTree()
    },

    onLoaded(id) {
      if (id !== this.tab.id) return
      this.$el.classList.remove('-loaded')
      this.$el.offsetHeight
      this.$el.classList.add('-loaded')
      setTimeout(() => {this.$el.classList.remove('-loaded')}, 333)
    },

    /**
     * Close tab
     */
    close() {
      Store.dispatch('removeTabs', [this.tab.id])
    },

    /**
     * Close tabs tree
     */
    closeTree() {
      const toRemove = [this.tab.id]
      for (let tab of State.tabs) {
        if (toRemove.includes(tab.parentId)) toRemove.push(tab.id)
      }
      Store.dispatch('removeTabs', toRemove)
    },

    loadingStart(id) {
      if (id !== this.tab.id) return
      this.loading = true
      if (this.loadingTimer) {
        clearTimeout(this.loadingTimer)
        this.loadingTimer = null
      }
    },

    loadingEnd(id) {
      if (id !== this.tab.id) return
      this.loading = false
    },

    loadingOk(id) {
      if (id !== this.tab.id) return
      this.loading = 'ok'
      this.loadingTimer = setTimeout(() => {
        this.loadingEnd(id)
        this.loadingTimer = null
      }, 2000)
    },

    loadingErr(id) {
      if (id !== this.tab.id) return
      this.loading = 'err'
      this.loadingTimer = setTimeout(() => {
        this.loadingEnd(id)
        this.loadingTimer = null
      }, 2000)
    },

    // ??? remove
    height() {
      return this.$el.offsetHeight
    },
  },
}
</script>


<style lang="stylus">
@import '../../../styles/mixins'

.Tab
  box(absolute, flex)
  width: 100%
  height: var(--tabs-height)
  align-items: center
  z-index: 10
  transform: translateZ(0)
  transition: opacity var(--d-fast), transform var(--d-fast), z-index 0s .2s
  border: var(--tabs-border)
  box-shadow: var(--tabs-shadow)
  &:hover
    background-color: var(--tabs-bg-hover)
  &:hover
  &[is-active]:hover
    .fav
      opacity: .7
    .title
      color: var(--tabs-fg-hover)
    .close
      opacity: 1
  &:active
  &[is-active]:active
    background-color: var(--tabs-bg-active)
    .fav
      transition: none
      opacity: .5

  &[lvl="1"]
    padding-left: var(--tabs-indent)
  &[lvl="2"]
    padding-left: calc(var(--tabs-indent) * 2)
  &[lvl="3"]
    padding-left: calc(var(--tabs-indent) * 3)
  &[lvl="4"]
    padding-left: calc(var(--tabs-indent) * 4)
  &[lvl="5"]
    padding-left: calc(var(--tabs-indent) * 5)
  &[lvl="6"]
    padding-left: calc(var(--tabs-indent) * 6)
  &[lvl="7"]
    padding-left: calc(var(--tabs-indent) * 7)
  &[lvl="8"]
    padding-left: calc(var(--tabs-indent) * 8)
  &[lvl="9"]
    padding-left: calc(var(--tabs-indent) * 9)
  &[lvl="10"]
    padding-left: calc(var(--tabs-indent) * 10)

  &[is-parent] .fav:hover
    > .exp
      z-index: 1
      opacity: 1
      transform: scale(1, 1)

  &[is-parent][data-no-fav] .fav:hover
    > .placeholder
      opacity: .5

  &[is-parent]:not([data-no-fav]) .fav:hover
    > img
      opacity: .2

  &[is-parent][folded]
    .fav > .exp
      z-index: 1
      opacity: 1
      transform: scale(1, 1) rotateZ(-90deg)

  &[is-parent][folded][data-no-fav]
    .fav > .placeholder
      opacity: .5

  &[is-parent][folded]:not([data-no-fav])
    .fav > img
      opacity: .2

  &[is-active]
    background-color: var(--tabs-activated-bg)
    border: var(--tabs-activated-border)
    box-shadow: var(--tabs-activated-shadow)
    .fav
      opacity: 1
    .title
      color: var(--tabs-activated-fg)

  &[close-btn]:hover
    &[data-audible] .t-box
    &[data-muted] .t-box
      mask: linear-gradient(-90deg, transparent, transparent 42px, #000000 64px, #000000)
    .t-box
      mask: linear-gradient(-90deg, transparent, transparent 24px, #000000 48px, #000000)
  
  &[data-status="loading"]
    cursor: progress
    .title
      transform: translateX(11px)
    .loading
      opacity: 1
    .loading > svg.-a
      animation: tab-loading .8s infinite
    .loading > svg.-b
      animation: tab-loading .8s .07s infinite
    .loading > svg.-c
      animation: tab-loading .8s .14s infinite

  &[data-no-fav]
    .fav > .placeholder
      opacity: 1
      transform: translateY(0)
    .fav > img
      opacity: 0
      transform: translateY(-4px)
      transition: none

  &[data-audible]
    .audio
      opacity: 1
      z-index: 20
      transform: translateX(0)
    .t-box
      transform: translateX(16px)
    .t-box
      mask: linear-gradient(-90deg, transparent, transparent 16px, #000000 28px, #000000)

  &[data-muted]
    .audio
      opacity: .8
      z-index: 20
      transform: translateX(0)
      > svg.-loud
        opacity: 0
      > svg.-mute
        opacity: 1
    .t-box
      transform: translateX(16px)
    .t-box
      mask: linear-gradient(-90deg, transparent, transparent 16px, #000000 28px, #000000)

  &[is-selected]
  &[is-selected]:hover
  &[is-selected]:active
    z-index: 10
    background-color: var(--tabs-selected-bg)
    border: var(--tabs-selected-border)
    box-shadow: var(--tabs-selected-shadow)
    .title
      color: var(--tabs-selected-fg)
  
  &[discarded]
    opacity: .5

  &[updated] .fav
    > img
      mask: radial-gradient(
        circle at calc(100% - 2px) calc(100% - 2px),
        #00000032,
        #00000032 4px,
        #000000 5px,
        #000000
      )
    > .update-badge
      opacity: 1
      transform: scale(1, 1)

  &[invisible]
    opacity: 0
    z-index: -1

// --- Level Wrapper
.Tab .lvl-wrapper
  box(relative, flex)
  size(100%, same)
  align-items: center
  transition: opacity var(--d-fast), transform var(--d-fast)
  &:before
    content: ''
    box(absolute, none)
    size(3px, 3px)
    pos(calc(50% - 1px), 2px)
    border-radius: 50%
    opacity: .8
#root.-tabs-lvl-marks .Tab .lvl-wrapper:before
  box(block)
  box-shadow: calc(var(--tabs-indent) / -2) 0 0 0 var(--inactive-fg),
              calc(var(--tabs-indent) * -1.5) 0 0 0 var(--inactive-fg),
              calc(var(--tabs-indent) * -2.5) 0 0 0 var(--inactive-fg),
              calc(var(--tabs-indent) * -3.5) 0 0 0 var(--inactive-fg),
              calc(var(--tabs-indent) * -4.5) 0 0 0 var(--inactive-fg),
              calc(var(--tabs-indent) * -5.5) 0 0 0 var(--inactive-fg),
              calc(var(--tabs-indent) * -6.5) 0 0 0 var(--inactive-fg),
              calc(var(--tabs-indent) * -7.5) 0 0 0 var(--inactive-fg),
              calc(var(--tabs-indent) * -8.5) 0 0 0 var(--inactive-fg),
              calc(var(--tabs-indent) * -9.5) 0 0 0 var(--inactive-fg)

// --- Drag layer ---
.Tab .drag-layer
  box(absolute)
  size(100%, same)
  pos(0, 0)
  z-index: 15

// --- Audio ---
.Tab .audio
  box(absolute)
  pos(0, 24px)
  size(16px, 100%)
  z-index: 1
  opacity: 0
  transform: translateX(-100%)
  transition: opacity var(--d-fast), transform var(--d-fast)

  > svg
    box(absolute)
    pos(calc(50% - 5px), 6px)
    size(11px, same)
    fill: var(--tabs-fg)
    transition: opacity var(--d-fast)

  > svg.-mute
    opacity: 0

// --- Favicon ---
.Tab .fav
  box(relative)
  size(16px, same)
  flex-shrink: 0
  margin: 0 6px 0 7px
  opacity: 1
  z-index: 20
  transition: opacity var(--d-fast), transform var(--d-fast)
  &[loading="true"]
    cursor: progress
    > .loading-spinner
      opacity: 1
      for i in 0..12
        > .spinner-stick-{i}
          animation: loading-spin .6s (i*50)ms infinite
  &[loading="ok"]
    > .ok-badge
      opacity: 1
      transform: scale(1, 1)
  &[loading="err"]
    > .err-badge
      opacity: 1
      transform: scale(1, 1)
  &[loading]
    > img
      mask: radial-gradient(
        circle at calc(100% - 2px) calc(100% - 2px),
        #00000032,
        #00000032 6px,
        #000000 7px,
        #000000
      )

.Tab .fav > .placeholder
  box(absolute)
  size(16px, same)
  pos(0, 0)
  opacity: 0
  transform: translateY(4px)
  transition: opacity var(--d-fast), transform var(--d-fast)
  > svg
    box(absolute)
    size(100%, same)
    pos(0, 0)
    fill: var(--favicons-placehoder-bg)

.Tab .fav > img
  box(absolute)
  size(100%, same)
  transition: opacity var(--d-fast), transform var(--d-fast)

.Tab .fav > .exp
  box(absolute)
  size(calc(100% + 8px), same)
  pos(-4px, -4px)
  opacity: 0
  z-index: -1
  cursor: pointer
  transform: scale(0.5, 0.5)
  transition: opacity var(--d-fast), transform var(--d-fast)
  > svg
    box(absolute)
    pos(5px, same)
    size(14px, same)
    fill: var(--bookmarks-folder-open-fg)

.Tab .fav > .loading-spinner
  box(absolute)
  size(10px, same)
  pos(b: -4px, r: -4px)
  border-radius: 50%
  opacity: 0
  transition: opacity var(--d-norm)

  > .spinner-stick
    box(absolute)
    size(1px, 4px)
    pos(calc(50% - 2px), calc(50% - 1px))
    opacity: 0

    &:before
      box(absolute)
      pos(4px, 0)
      size(100%, same)
      background-color: var(--tabs-loading-fg)
      content: ''
  for i in 0..12
    > .spinner-stick-{i}
      transform: rotateZ((i * 30)deg)
      animation: none

.Tab .fav > .update-badge
  box(absolute)
  size(6px, same)
  pos(b: -1px, r: -1px)
  border-radius: 50%
  background-color: var(--tabs-update-badge-bg)
  opacity: 0
  transform: scale(0.7, 0.7)
  transition: opacity var(--d-norm), transform var(--d-norm)

.Tab .fav > .ok-badge
.Tab .fav > .err-badge
  box(absolute)
  size(10px, same)
  pos(b: -3px, r: -3px)
  border-radius: 50%
  opacity: 0
  transform: scale(0.7, 0.7)
  transition: opacity var(--d-norm), transform var(--d-norm)
  > svg
    box(absolute)
    size(100%, same)

.Tab .fav > .ok-badge > svg
  fill: var(--true-fg)

.Tab .fav > .err-badge > svg
  fill: var(--false-fg)

.Tab .fav > .child-count
  box(absolute)
  size(8px)
  pos(b: -5px, r: -3px)
  font: var(--tabs-count-font)
  text-align: center
  color: var(--tabs-fg)

// --- Context highlight
.Tab .ctx
  box(absolute)
  pos(b: 10px, l: 0px)
  size(2px, 10px)
  z-index: 2000
  box-shadow: 0 0 2px 0 #00000024

.Tab .loaded-fx
  box(absolute)
  size(100%, same)
  pos(0, 0)
  background-image: linear-gradient(90deg, #00000000, var(--tabs-loading-fg))
  opacity: 0
  transform: translateX(-100%)
.Tab.-loaded .loaded-fx
  animation: tab-loaded .3s

// --- Title box ---
.Tab .t-box
  box(relative)
  size(100%)
  transition: opacity var(--d-fast), transform var(--d-fast)
  overflow: hidden
  mask: linear-gradient(-90deg, transparent, #000000 12px, #000000)

// Title
.Tab .title
  box(relative)
  font: var(--tabs-font)
  color: var(--tabs-fg)
  padding: 0 1px
  transition: color .2s
  white-space: nowrap
  overflow: hidden
  transition: transform var(--d-fast), color var(--d-fast), mask var(--d-fast)

// Loading
.Tab .loading
  box(absolute)
  pos(calc(50% - 8px), 0)
  size(7px, 16px)
  opacity: 0
  transition: transform var(--d-fast), opacity var(--d-fast)
  > svg
    box(absolute)
    pos(3px, 0)
    size(7px, 4px)
    fill: var(--tabs-loading-fg)
    opacity: 0
  > svg.-b
    pos(7px)
  > svg.-c
    pos(11px)

// --- CLose button ---
.Tab .close
  box(absolute)
  pos(0, r: 0)
  size(31px)
  height: var(--tabs-height)
  cursor: pointer
  z-index: 20
  opacity: 0
  &:hover > svg
    fill: #ea4335
  &:active > svg
    transition: none
    fill: #fa5335
  > svg
    box(absolute)
    pos(calc(50% - 8px), same)
    size(17px, same)
    fill: #a63626
    transition: fill var(--d-fast)

// --- Animations ---
@keyframes tab-loading
  0%
    opacity: 1
  100%
    opacity: 0

@keyframes tab-loaded
  0%
    opacity: .8
    transform: translateX(-100%)
  100%
    opacity: 0
    transform: translateX(5px)
</style>
