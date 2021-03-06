<template lang="pug">
.CtxMenuBuilder
  scroll-box(ref="scrollBox")
    .box
      section.section
        .title {{t('menu.editor.tabs_title')}}

        .menu-group(v-for="(group, gi) in inlineTabsMenu")
          .group-title {{t('menu.editor.inline_group_title')}}
          .opt(v-for="(opt, i) in group", :title="t(tabsOpts[opt])")
            .opt-title {{t(tabsOpts[opt])}}
            .opt-btn(@click="downTabOpt(opt, i, gi)"): svg: use(xlink:href="#icon_expand")
            .opt-btn.-up(v-if="gi > 0 || group.length > 1", @click="upTabOpt(opt, i, gi)")
              svg: use(xlink:href="#icon_expand")

        .menu-group(v-if="listTabsMenu.length")
          .group-title {{t('menu.editor.list_title')}}
          .opt(v-for="(opt, i) in listTabsMenu", :title="t(tabsOpts[opt])")
            .opt-title {{t(tabsOpts[opt])}}
            .opt-btn(@click="downTabOpt(opt, i, -1)"): svg: use(xlink:href="#icon_expand")
            .opt-btn.-up(@click="upTabOpt(opt, i, -1)"): svg: use(xlink:href="#icon_expand")

        .menu-group(v-if="disabledTabsMenu.length")
          .group-title {{t('menu.editor.disabled_title')}}
          .opt(v-for="opt in disabledTabsMenu", :title="t(tabsOpts[opt])")
            .opt-title {{t(tabsOpts[opt])}}
            .opt-btn.-up(@click="restoreTabOpt(opt)"): svg: use(xlink:href="#icon_expand")

        .btn(@click="resetTabsMenu") {{t('menu.editor.reset')}}

      section.section
        .title {{t('menu.editor.bookmarks_title')}}

        .menu-group(v-for="(group, gi) in inlineBookmarksMenu")
          .group-title {{t('menu.editor.inline_group_title')}}
          .opt(v-for="(opt, i) in group", :title="t(bookmarksOpts[opt])")
            .opt-title {{t(bookmarksOpts[opt])}}
            .opt-btn(@click="downBookmarkOpt(opt, i, gi)"): svg: use(xlink:href="#icon_expand")
            .opt-btn.-up(v-if="gi > 0 || group.length > 1", @click="upBookmarkOpt(opt, i, gi)")
              svg: use(xlink:href="#icon_expand")

        .menu-group(v-if="listBookmarksMenu.length")
          .group-title {{t('menu.editor.list_title')}}
          .opt(v-for="(opt, i) in listBookmarksMenu", :title="t(bookmarksOpts[opt])")
            .opt-title {{t(bookmarksOpts[opt])}}
            .opt-btn(@click="downBookmarkOpt(opt, i, -1)"): svg: use(xlink:href="#icon_expand")
            .opt-btn.-up(@click="upBookmarkOpt(opt, i, -1)"): svg: use(xlink:href="#icon_expand")

        .menu-group(v-if="disabledBookmarksMenu.length")
          .group-title {{t('menu.editor.disabled_title')}}
          .opt(v-for="opt in disabledBookmarksMenu", :title="t(bookmarksOpts[opt])")
            .opt-title {{t(bookmarksOpts[opt])}}
            .opt-btn.-up(@click="restoreBookmarkOpt(opt)"): svg: use(xlink:href="#icon_expand")

        .btn(@click="resetBookmarksMenu") {{t('menu.editor.reset')}}

</template>


<script>
import State from '../../store.state'
import Store from '../../store'
import ScrollBox from '../scroll-box'
import { DEFAULT_TABS_MENU, DEFAULT_BOOKMARKS_MENU } from '../../store.state'

const TABS_MENU_OPTS = {
  'undoRmTab': 'menu.tab.undo',
  'pin': 'menu.tab.pin',
  'reload': 'menu.tab.reload',
  'bookmark': 'menu.tab.bookmark',
  'moveToNewWin': 'menu.tab.move_to_new_window',
  'moveToNewPrivWin': 'menu.tab.move_to_new_priv_window',
  'moveToAnotherWin': 'menu.tab.move_to_another_window',
  'moveToWin': 'menu.tab.move_to_window_',
  'moveToCtr': 'menu.tab.reopen_in_ctr_',
  'mute': 'menu.tab.mute',
  'discard': 'menu.tab.discard',
  'group': 'menu.tab.group',
  'flatten': 'menu.tab.flatten',
  'clearCookies': 'menu.tab.clear_cookies',
  'close': 'menu.tab.close',
}

const BOOKMARKS_MENU_OPTS = {
  'openInNewWin': 'menu.bookmark.open_in_new_window',
  'openInNewPrivWin': 'menu.bookmark.open_in_new_priv_window',
  'openInCtr': 'menu.bookmark.open_in_ctr_',
  'createBookmark': 'menu.bookmark.create_bookmark',
  'createFolder': 'menu.bookmark.create_folder',
  'createSeparator': 'menu.bookmark.create_separator',
  'edit': 'menu.bookmark.edit_bookmark',
  'delete': 'menu.bookmark.delete_bookmark',
}

export default {
  components: {
    ScrollBox,
  },

  data() {
    return {
      active: false,
      tabsOpts: TABS_MENU_OPTS,
      bookmarksOpts: BOOKMARKS_MENU_OPTS,
    }
  },

  computed: {
    inlineTabsMenu() {
      return State.tabsMenu.filter(m => m instanceof Array)
    },

    listTabsMenu() {
      return State.tabsMenu.filter(m => !(m instanceof Array))
    },

    disabledTabsMenu() {
      const all = Object.keys(TABS_MENU_OPTS)
      return all.filter(action => {
        const usedInInline = this.inlineTabsMenu.some(m => m.includes(action))
        const usedInList = this.listTabsMenu.includes(action)
        return !usedInInline && !usedInList
      })
    },

    inlineBookmarksMenu() {
      return State.bookmarksMenu.filter(m => m instanceof Array)
    },

    listBookmarksMenu() {
      return State.bookmarksMenu.filter(m => !(m instanceof Array))
    },

    disabledBookmarksMenu() {
      const all = Object.keys(BOOKMARKS_MENU_OPTS)
      return all.filter(action => {
        const usedInInline = this.inlineBookmarksMenu.some(m => m.includes(action))
        const usedInList = this.listBookmarksMenu.includes(action)
        return !usedInInline && !usedInList
      })
    },
  },

  methods: {
    /**
     * Move tab option up
     */
    upTabOpt(opt, i, gi) {
      if (gi > -1) {
        State.tabsMenu[gi].splice(i, 1)
        if (i === 0) {
          // Create new inline group or push to upper group
          if (gi === 0) State.tabsMenu.unshift([opt])
          else State.tabsMenu[gi - 1].push(opt)
        } else {
          State.tabsMenu[gi].splice(i - 1, 0, opt)
        }
      } else {
        const globalIndex = i + this.inlineTabsMenu.length
        State.tabsMenu.splice(globalIndex, 1)
        if (i === 0) {
          if (!this.inlineTabsMenu.length) State.tabsMenu.unshift([opt])
          else State.tabsMenu[this.inlineTabsMenu.length - 1].push(opt)
        } else {
          State.tabsMenu.splice(globalIndex - 1, 0, opt)
        }
      }

      const empty = State.tabsMenu.findIndex(inline => inline.length === 0)
      if (empty !== -1) State.tabsMenu.splice(empty, 1)

      Store.dispatch('saveCtxMenu')
    },

    /**
     * Move tab option down
     */
    downTabOpt(opt, i, gi) {
      if (gi > -1) {
        State.tabsMenu[gi].splice(i, 1)
        if (i === State.tabsMenu[gi].length) {
          if (gi === this.inlineTabsMenu.length - 1) {
            State.tabsMenu.splice(this.inlineTabsMenu.length, 0, opt)
          } else {
            State.tabsMenu[gi + 1].unshift(opt)
          }
        } else {
          State.tabsMenu[gi].splice(i + 1, 0, opt)
        }
      } else {
        const globalIndex = i + this.inlineTabsMenu.length
        State.tabsMenu.splice(globalIndex, 1)
        if (globalIndex !== State.tabsMenu.length) {
          State.tabsMenu.splice(globalIndex + 1, 0, opt)
        }
      }

      const empty = State.tabsMenu.findIndex(inline => inline.length === 0)
      if (empty !== -1) State.tabsMenu.splice(empty, 1)

      Store.dispatch('saveCtxMenu')
    },

    /**
     * Restore tab option
     */
    restoreTabOpt(opt) {
      State.tabsMenu.push(opt)
      Store.dispatch('saveCtxMenu')
    },

    /**
     * Reset tabs menu
     */
    resetTabsMenu() {
      State.tabsMenu = JSON.parse(JSON.stringify(DEFAULT_TABS_MENU))
      Store.dispatch('saveCtxMenu')
    },

    /**
     * Move bookmark option up
     */
    upBookmarkOpt(opt, i, gi) {
      if (gi > -1) {
        State.bookmarksMenu[gi].splice(i, 1)
        if (i === 0) {
          // Create new inline group or push to upper group
          if (gi === 0) State.bookmarksMenu.unshift([opt])
          else State.bookmarksMenu[gi - 1].push(opt)
        } else {
          State.bookmarksMenu[gi].splice(i - 1, 0, opt)
        }
      } else {
        const globalIndex = i + this.inlineBookmarksMenu.length
        State.bookmarksMenu.splice(globalIndex, 1)
        if (i === 0) {
          if (!this.inlineBookmarksMenu.length) State.bookmarksMenu.unshift([opt])
          else State.bookmarksMenu[this.inlineBookmarksMenu.length - 1].push(opt)
        } else {
          State.bookmarksMenu.splice(globalIndex - 1, 0, opt)
        }
      }

      const empty = State.bookmarksMenu.findIndex(inline => inline.length === 0)
      if (empty !== -1) State.bookmarksMenu.splice(empty, 1)

      Store.dispatch('saveCtxMenu')
    },

    /**
     * Move bookmark option down
     */
    downBookmarkOpt(opt, i, gi) {
      if (gi > -1) {
        State.bookmarksMenu[gi].splice(i, 1)
        if (i === State.bookmarksMenu[gi].length) {
          if (gi === this.inlineBookmarksMenu.length - 1) {
            State.bookmarksMenu.splice(this.inlineBookmarksMenu.length, 0, opt)
          } else {
            State.bookmarksMenu[gi + 1].unshift(opt)
          }
        } else {
          State.bookmarksMenu[gi].splice(i + 1, 0, opt)
        }
      } else {
        const globalIndex = i + this.inlineBookmarksMenu.length
        State.bookmarksMenu.splice(globalIndex, 1)
        if (globalIndex !== State.bookmarksMenu.length) State.bookmarksMenu.splice(globalIndex + 1, 0, opt)
      }

      const empty = State.bookmarksMenu.findIndex(inline => inline.length === 0)
      if (empty !== -1) State.bookmarksMenu.splice(empty, 1)

      Store.dispatch('saveCtxMenu')
    },

    /**
     * Restore bookarks option
     */
    restoreBookmarkOpt(opt) {
      State.bookmarksMenu.push(opt)
      Store.dispatch('saveCtxMenu')
    },

    /**
     * Reset bookmarks menu
     */
    resetBookmarksMenu() {
      State.bookmarksMenu = JSON.parse(JSON.stringify(DEFAULT_BOOKMARKS_MENU))
      Store.dispatch('saveCtxMenu')
    },
  },
}
</script>


<style lang="stylus">
@import '../../../styles/mixins'

.CtxMenuBuilder
  box(absolute, flex)
  pos(0, 0)
  size(100%, same)
  flex-direction: column

.CtxMenuBuilder > .title
  box(relative)
  text(s: rem(24))
  color: var(--title-fg)
  padding: 12px 16px

.CtxMenuBuilder .box
  box(relative, grid)
  grid-gap: 32px 0
  padding: 8px 0 16px

.CtxMenuBuilder .section
  box(relative, grid)
  grid-gap: 8px 0

.CtxMenuBuilder .section > .title
  box(relative)
  text(s: rem(24))
  color: var(--sub-title-fg)
  padding: 2px 16px

.CtxMenuBuilder .menu-group
  box(relative)
  size(100%)
  overflow: hidden
  padding: 2px 8px 2px 16px
  
.CtxMenuBuilder .group-title
  box(relative)
  text(s: rem(18))
  color: var(--sub-title-fg)
  padding: 3px 0

.CtxMenuBuilder .opt
  box(relative, flex)
  align-items: center
  padding: 0 0 0 3px
  &:hover
    .opt-title
      opacity: 1
    .opt-btn
      opacity: .8

  .opt-title
    box(relative)
    text(s: rem(14))
    color: var(--label-fg)
    white-space: nowrap
    overflow: hidden
    text-overflow: ellipsis
    padding: 2px 5px 2px 0
    margin: 0 auto 0 0
    opacity: .7

  .opt-btn
    box(relative)
    size(22px, same)
    fill: var(--title-fg)
    flex-shrink: 0
    flex-grow: 0
    cursor: pointer
    opacity: .2
    &.-up svg
      transform: rotateZ(180deg)
    &:hover
      opacity: 1
    &:active
      opacity: .7
    svg
      box(absolute)
      size(16px, same)
      pos(calc(50% - 8px), same)

.CtxMenuBuilder .btn
  size(80%)
  margin-top: 8px
  margin-left: auto
  margin-right: auto
</style>
