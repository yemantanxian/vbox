<template>
  <div>

    <div class="tag-hd">
      <div class="tag-sort flex">
        <span class="flex-flex"></span>
        <span :class='["sort-type", sortType == 1 ? "tag-select":""]' @click.stop='changeSortType(1)'>最新</span>
        <span :class='["sort-type", sortType == 2 ? "tag-select":""]' @click.stop='changeSortType(2)'>最热</span>
        <span class="more_filter" @click.stop='toggleTag'>
           <setting-icon :color='"#666"'></setting-icon>
        </span>
      </div>
    </div>
    <transition name="'slide-left'">
      <album-tag class="album-tag" v-show="showTag" :tag='tagOpt' @confirmed='tagsChanged'></album-tag>
    </transition>

    <album-list :albums='albums' @addAlbumToPlaying='addAlbumToPlaying' v-show="!showTag"></album-list>

    <infinite-loading :on-infinite="onInfinite" ref="infiniteLoading" spinner='bubbles'>
      <span slot="no-more" class="no-more">
        没有更多了
     </span>
    </infinite-loading>
  </div>
</template>

<script>
  import InfiniteLoading from 'vue-infinite-loading'
  import AlbumList from '../components/Album/AlbumList'
  import AlbumTag from '../components/Album/AlbumTag'
  import Search from '../api/search'
  import SettingIcon from '../components/public/Icon/SettingIcon'
  export default {
    name: 'album-search-view',
    components: {
      InfiniteLoading,
      AlbumList,
      AlbumTag,
      SettingIcon
    },
    data() {
      return {
        // 从0开始
        tagOpt: undefined,
        page: 0,
        pagesize: 30,
        sortType: 1,
        tag: {
          // 语种
          language: -1,
          // 流派
          genre: 0,
          // 年代
          year: 1,
          // 1 免费  2 免费
          pay: 0,
          // 专辑/演唱会等
          type: -1,
          // 唱片公司 华纳唱片/环球唱片等
          company: -1
        },
        sum: 100,
        albums: [],
        showTag: false
      }
    },
    methods: {
      async onInfinite() {
        try {
          let res = await Search.albumLib(!this.tagOpt ? 'firstpage' : 'get_album_info', this.page, this.pagesize, this.sortType, this.tag.language,
            this.tag.genre, this.tag.year, 0, this.tag.type, this.tag.company).then(res => res.json());
          if (!this.tagOpt) {
            this.tagOpt = res.data.tag
          }
          this.sum = res.data.sum
          if (!res.data || !res.data.albumlist || res.data.albumlist.length === 0 || this.albums.length >= this.sum) {
            this.$refs.infiniteLoading.$emit('$InfiniteLoading:complete')
          } else {
            this.albums = this.albums.concat(res.data.albumlist.map(album => {
              album.albumMID = album.album_mid
              album.albumName = album.album_name
              return album
            }))
            this.page++
            this.$refs.infiniteLoading.$emit('$InfiniteLoading:loaded')
          }
        } catch (err) {
          console.log(err)
          if (this.$refs.infiniteLoading) {
            this.$refs.infiniteLoading.$emit('$InfiniteLoading:complete')
          }
        }
      },
      changeSortType(sortType) {
        this.sortType = sortType
      },
      resetInfiniteLoading() {
        this.page = 0
        this.albums = []
        this.$refs.infiniteLoading.isLoading = true
        this.onInfinite();
      },
      async addAlbumToPlaying(albumMID) {
        let list = await Search.albumInfo(albumMID, true).then(res => res.json())
        this.$store.commit('playing/addSongs', list)
        this.$store.commit('playing/next', [0])
      },
      toggleTag() {
        this.showTag = !this.showTag
      },
      tagsChanged(tag) {
        if (tag.confirmed) {
          this.tag = tag.tag
        }
        this.showTag = false
      }
    },
    watch: {
      tag: {
        handler: async function (to) {
          this.resetInfiniteLoading();
        },
        deep: true
      },
      sortType() {
        this.resetInfiniteLoading()
      }
    }
  }

</script>


<style scoped>
  .more_filter {
    padding: 0.3rem 0.4rem;
    height: 1.2rem;
    display: inline-block
  }

  .album-tag {
    position: absolute;
    top: 0;
    background: #FFF;
    z-index: 999;
    width: 100%;
  }
</style>