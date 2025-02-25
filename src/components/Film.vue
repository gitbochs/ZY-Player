<template>
  <div class="listpage" id="film">
    <div class="listpage-header" id="film-header">
      <el-select v-model="selectedSiteName" size="small" placeholder="源站" :popper-append-to-body="false" popper-class="popper" @change="siteClick">
        <el-option
          v-for="item in sites"
          :key="item.key"
          :label="item.name"
          :value="item.name">
        </el-option>
      </el-select>
      <el-select v-model="selectedClassName" size="small" placeholder="类型" :popper-append-to-body="false" popper-class="popper" @change="classClick" v-show="show.class">
        <el-option
          v-for="item in classList"
          :key="item.tid"
          :label="item.name"
          :value="item.name">
        </el-option>
      </el-select>
      <el-autocomplete
        clearable
        size="small"
        v-model.trim="searchTxt"
        value-key="keywords"
        :fetch-suggestions="querySearch"
        :popper-append-to-body="false"
        popper-class="popper"
        placeholder="搜索"
        @keyup.enter.native="searchAndRecord"
        @select="searchEvent"
        @change="searchChangeEvent">
        <el-select v-model="searchGroup" size="small" slot="prepend"
          :popper-append-to-body="false"
          popper-class="popper"
          default-first-option placeholder="请选择"
          @change="searchEvent">
          <el-option
            v-for="item in searchGroups"
            :key="item"
            :label="item"
            :value="item">
          </el-option>
        </el-select>
        <!--方便触屏-->
        <el-button icon="el-icon-search" @click.stop="searchEvent" slot="append" />
      </el-autocomplete>
    </div>
    <div class="listpage-body" id="film-body" infinite-wrapper>
      <div class="show-picture" v-if="setting.view === 'picture' && !show.find">
          <Waterfall ref="filmWaterfall" :list="list" :gutter="20" :width="240"
          :breakpoints="{
            1200: { //当屏幕宽度小于等于1200
              rowPerView: 4,
            },
            800: { //当屏幕宽度小于等于800
              rowPerView: 3,
            },
            500: { //当屏幕宽度小于等于500
              rowPerView: 2,
            }
          }"
          animationEffect="fadeIn"
          backgroundColor="rgba(0, 0, 0, 0)">
            <template slot="item" slot-scope="props">
              <div class="card" v-show="!setting.excludeR18Films || !containsR18Keywords(props.data.type)">
                <div class="img">
                  <img style="width: 100%" :src="props.data.pic" alt="" @load="$refs.filmWaterfall.refresh()" @click="detailEvent(site, props.data)">
                  <div class="operate">
                    <div class="operate-wrap">
                      <span class="o-play" @click="playEvent(site, props.data)">播放</span>
                      <span class="o-star" @click="starEvent(site, props.data)">收藏</span>
                      <span class="o-share" @click="shareEvent(site, props.data)">分享</span>
                    </div>
                  </div>
                </div>
                <div class="name" @click="detailEvent(site, props.data)">{{props.data.name}}</div>
                <div class="info">
                  <span>{{props.data.area}}</span>
                  <span>{{props.data.year}}</span>
                  <span>{{props.data.note}}</span>
                  <span>{{props.data.type}}</span>
                </div>
              </div>
            </template>
          </Waterfall>
          <infinite-loading force-use-infinite-wrapper :identifier="infiniteId" @infinite="infiniteHandler"></infinite-loading>
      </div>
      <div class="show-table" v-if="setting.view === 'table' && !show.find">
        <el-table
          size="mini"
          :data="list.filter(res => !setting.excludeR18Films || !containsR18Keywords(res.type))"
          height="100%"
          :empty-text="statusText"
          @row-click="(row) => detailEvent(site, row)"
          style="width: 100%">
          <el-table-column
            prop="name"
            label="片名">
          </el-table-column>
          <el-table-column
            prop="type"
            label="类型"
            width="100">
          </el-table-column>
          <el-table-column
            prop="year"
            label="上映"
            align="center"
            width="100">
          </el-table-column>
          <el-table-column
            prop="area"
            label="地区"
            width="100">
          </el-table-column>
          <el-table-column
            prop="lang"
            label="语言"
            width="100">
          </el-table-column>
          <el-table-column
            prop="note"
            label="备注"
            width="120">
          </el-table-column>
          <el-table-column
            prop="last"
            label="最近更新"
            :formatter="dateFormat"
            align="left">
          </el-table-column>
          <el-table-column
            label="操作"
            header-align="center"
            align="right"
            width="200">
            <template slot-scope="scope">
              <el-button @click.stop="playEvent(site, scope.row)" type="text">播放</el-button>
              <el-button @click.stop="starEvent(site, scope.row)" type="text">收藏</el-button>
              <el-button @click.stop="shareEvent(site, scope.row)" type="text">分享</el-button>
              <el-button @click.stop="downloadEvent(site, scope.row)" type="text">下载</el-button>
            </template>
          </el-table-column>
          <infinite-loading
            slot="append"
            :identifier="infiniteId"
            @infinite="infiniteHandler"
            force-use-infinite-wrapper=".el-table__body-wrapper">
            <div slot="no-more">数据量过少时请重复操作一次，以防网站抽风</div>
          </infinite-loading>
        </el-table>
      </div>
      <div class="show-table" v-show="show.find">
        <el-table size="mini"
          ref="searchResultTable"
          :data="searchContents.filter(res => !setting.excludeR18Films || (res.type !== undefined && !containsR18Keywords(res.type)))"
          height="100%"
          :empty-text="statusText"
          @filter-change="filterChange"
          @row-click="(row) => detailEvent(row.site, row)"
          style="width: 100%">
          <el-table-column
            sortable
            :sort-method="(a , b) => sortByLocaleCompare(a.name, b.name)"
            prop="name"
            label="片名">
          </el-table-column>
          <el-table-column v-if="searchGroup !== '站内'"
            sortable
            :sort-method="(a , b) => sortByLocaleCompare(a.site.name, b.site.name)"
            :filters="getFilters('siteName')"
            :filter-method="(value, row, column) => { this.currentColumn = column; return value === row.site.name }"
            prop="site"
            label="源站"
            width="120">
            <template slot-scope="scope">
              <span>{{ scope.row.site.name }}</span>
            </template>
          </el-table-column>
          <el-table-column
            prop="type"
            :filters="getFilters('type')"
            :filter-method="(value, row, column) => { this.currentColumn = column; return value === row.type }"
            label="类型"
            width="90">
          </el-table-column>
          <el-table-column
              sortable
              prop="year"
              label="上映"
              width="90">
          </el-table-column>
          <el-table-column
            prop="area"
            :filters="getFilters('area')"
            :filter-method="(value, row, column) => { this.currentColumn = column; return value === row.area }"
            label="地区"
            width="90">
          </el-table-column>
          <el-table-column
            :filters="getFilters('lang')"
            :filter-method="(value, row, column) => { this.currentColumn = column; return value === row.lang }"
            prop="lang"
            label="语言"
            width="70">
          </el-table-column>
          <el-table-column
            sortable
            prop="note"
            label="备注"
            width="120">
          </el-table-column>
          <el-table-column
            label="操作"
            header-align="center"
            align="right"
            width="200">
            <template slot-scope="scope">
              <el-button @click.stop="playEvent(scope.row.site, scope.row)" type="text">播放</el-button>
              <el-button @click.stop="starEvent(scope.row.site, scope.row)" type="text">收藏</el-button>
              <el-button @click.stop="shareEvent(scope.row.site, scope.row)" type="text">分享</el-button>
              <el-button @click.stop="downloadEvent(scope.row.site, scope.row)" type="text">下载</el-button>
            </template>
          </el-table-column>
        </el-table>
      </div>
    </div>
  </div>
</template>
<script>
import { mapMutations } from 'vuex'
import { star, history, search, sites, setting } from '../lib/dexie'
import zy from '../lib/site/tools'
import Waterfall from 'vue-waterfall-plugin'
import InfiniteLoading from 'vue-infinite-loading'
const { clipboard } = require('electron')
export default {
  name: 'film',
  data () {
    return {
      show: {
        body: false,
        site: false,
        class: false,
        classList: false,
        find: false
      },
      sites: [],
      site: {},
      classList: [],
      type: {},
      selectedClassName: '最新',
      selectedSiteName: '',
      pagecount: 0,
      list: [],
      statusText: ' ',
      infiniteId: +new Date(),
      searchID: 0,
      searchList: [],
      searchTxt: '',
      searchContents: [],
      currentColumn: '',
      searchGroup: '',
      searchGroups: [],
      // 福利片关键词
      r18KeyWords: ['伦理', '论理', '倫理', '福利', '激情', '理论', '写真', '情色', '美女', '街拍', '赤足', '性感', '里番']
    }
  },
  components: {
    Waterfall,
    InfiniteLoading
  },
  computed: {
    view: {
      get () {
        return this.$store.getters.getView
      },
      set (val) {
        this.SET_VIEW(val)
      }
    },
    video: {
      get () {
        return this.$store.getters.getVideo
      },
      set (val) {
        this.SET_VIDEO(val)
      }
    },
    detail: {
      get () {
        return this.$store.getters.getDetail
      },
      set (val) {
        this.SET_DETAIL(val)
      }
    },
    share: {
      get () {
        return this.$store.getters.getShare
      },
      set (val) {
        this.SET_SHARE(val)
      }
    },
    setting: {
      get () {
        return this.$store.getters.getSetting
      },
      set (val) {
        this.SET_SETTING(val)
      }
    },
    filterSettings () {
      return this.$store.getters.getSetting.excludeR18Films // 需要监听的数据
    }
  },
  filters: {
    classNameFilter: (name) => {
      const clsName = name.toString()
      return clsName.replace(/[^\u4e00-\u9fa5]/gi, '')
    }
  },
  watch: {
    view () {
      this.changeView()
    },
    searchTxt () {
      if (this.searchTxt === '清除历史记录...') {
        this.clearSearchHistory()
        this.searchTxt = ''
        this.searchChangeEvent()
      }
    },
    filterSettings () {
      this.siteClick(this.site.name)
    }
  },
  methods: {
    ...mapMutations(['SET_VIEW', 'SET_DETAIL', 'SET_VIDEO', 'SET_SHARE', 'SET_SETTING']),
    sortByLocaleCompare (a, b) {
      return a.localeCompare(b, 'zh')
    },
    dateFormat (row, column) {
      var date = row[column.property]
      if (date === undefined) {
        return ''
      }
      return date.split(/\s/)[0]
    },
    getFilters (column) {
      const searchContents = this.searchContents.filter(res => !this.setting.excludeR18Films || (res.type !== undefined && !this.containsR18Keywords(res.type)))
      if (column === 'siteName') return [...new Set(searchContents.map(row => row.site.name))].map(e => { return { text: e, value: e } }) // 有方法合并这两行吗？
      return [...new Set(searchContents.map(row => row[column]))].map(e => { return { text: e, value: e } })
    },
    filterChange (filters) {
      // 一次只能一列
      if (Object.values(filters)[0].length) {
        const otherColumns = this.$refs.searchResultTable.columns.filter(col => col.id !== this.currentColumn.id)
        otherColumns.forEach(col => { col.filterable = false })
      } else {
        const filterLabels = ['源站', '类型', '地区', '语言']
        const columns = this.$refs.searchResultTable.columns.filter(col => filterLabels.includes(col.label))
        columns.forEach(col => { col.filterable = true })
      }
    },
    siteClick (siteName) {
      this.list = []
      this.site = this.sites.find(x => x.name === siteName)
      if (this.searchTxt.length > 0 && this.searchGroup === '站内') {
        this.searchEvent()
      } else {
        this.searchTxt = ''
        this.show.find = false
        this.classList = []
        this.type = {}
        this.getClass().then(res => {
          this.infiniteId += 1
          this.classClick(this.classList[0].name)
        })
      }
    },
    classClick (className) {
      this.show.classList = false
      this.list = []
      this.type = this.classList.find(x => x.name === className)
      this.getPage().then(res => {
        if (res) {
          this.infiniteId += 1
        }
      })
    },
    getClass () {
      return new Promise((resolve, reject) => {
        const key = this.site.key
        // 屏蔽主分类
        const classToHide = ['电影', '电影片', '电视剧', '连续剧', '综艺', '动漫']
        zy.class(key).then(res => {
          var allClass = [{ name: '最新', tid: 0 }]
          res.class.forEach(element => {
            if (!this.setting.excludeRootClasses || !classToHide.includes(element.name)) {
              if (this.setting.excludeR18Films) {
                const containKeyWord = this.containsR18Keywords(element.name)
                if (!containKeyWord) {
                  allClass.push(element)
                }
              } else {
                allClass.push(element)
              }
            }
          })
          this.classList = allClass
          this.show.class = true
          this.pagecount = res.pagecount
          this.type = this.classList[0]
          resolve(true)
        }).catch(err => {
          reject(err)
        })
      })
    },
    containsR18Keywords (name) {
      var containKeyWord = false
      if (!name) {
        return containKeyWord
      }
      return this.r18KeyWords.some(v => name.includes(v))
    },
    getPage () {
      return new Promise((resolve, reject) => {
        const key = this.site.key
        const type = this.type.tid
        zy.page(key, type).then(res => {
          this.pagecount = res.pagecount
          this.show.body = true
          resolve(true)
        }).catch(err => {
          reject(err)
        })
      })
    },
    infiniteHandler ($state) {
      const key = this.site.key
      const type = this.type.tid
      const page = this.pagecount
      this.statusText = ' '
      if (key && page < 1) { // OK资源前几类硬是去不掉
        $state.complete()
        this.statusText = '暂无数据'
        return false
      }
      zy.list(key, page, type).then(res => {
        if (res) {
          this.pagecount -= 1
          const type = Object.prototype.toString.call(res)
          if (type === '[object Undefined]') {
            $state.complete()
          }
          if (type === '[object Array]') {
            // zy.list 返回的是按时间从旧到新排列, 我门需要翻转为从新到旧
            this.list.push(...res.reverse())
          }
          if (type === '[object Object]') {
            this.list.push(res)
          }
          $state.loaded()
        } else {
          $state.complete()
          this.statusText = '暂无数据'
        }
      })
    },
    detailEvent (site, e) {
      this.detail = {
        show: true,
        key: site.key,
        site: site,
        info: e
      }
    },
    async playEvent (site, e) {
      const db = await history.find({ site: site.key, ids: e.id })
      if (db) {
        this.video = { key: db.site, info: { id: db.ids, name: db.name, index: db.index, site: site } }
      } else {
        this.video = { key: site.key, info: { id: e.id, name: e.name, index: 0, site: site } }
      }
      zy.detail(site.key, e.id).then(detailRes => {
        this.video.detail = detailRes
      })
      this.view = 'Play'
    },
    async starEvent (site, e) {
      const db = await star.find({ key: site.key, ids: e.id })
      if (db) {
        this.$message.info('已存在')
      } else {
        zy.detail(site.key, e.id).then(detailRes => {
          const docs = {
            key: site.key,
            ids: e.id,
            site: site,
            name: e.name,
            detail: detailRes
          }
          star.add(docs).then(res => {
            this.$message.success('收藏成功')
          })
        })
      }
    },
    shareEvent (site, e) {
      this.share = {
        show: true,
        key: site.key,
        info: e
      }
    },
    downloadEvent (site, row) {
      zy.download(site.key, row.id).then(res => {
        if (res && res.length > 0) {
          const text = res.dl.dd._t
          if (text) {
            const list = text.split('#')
            let downloadUrl = res.name + '\n'
            for (const i of list) {
              const url = encodeURI(i.split('$')[1])
              downloadUrl += (url + '\n')
            }
            clipboard.writeText(downloadUrl)
            this.$message.success('『MP4』格式的链接已复制, 快去下载吧!')
          } else {
            this.$message.warning('没有查询到下载链接.')
          }
        } else {
          let m3u8List = []
          const dd = row.dl.dd
          const type = Object.prototype.toString.call(dd)
          if (type === '[object Array]') {
            for (const i of dd) {
              if (i._flag.indexOf('m3u8') >= 0) {
                m3u8List = i._t.split('#')
              }
            }
          } else {
            m3u8List = dd._t.split('#')
          }
          let downloadUrl = row.name + '\n'
          for (const i of m3u8List) {
            const url = encodeURI(i.split('$')[1])
            downloadUrl += (url + '\n')
          }
          clipboard.writeText(downloadUrl)
          this.$message.success('『M3U8』格式的链接已复制, 快去下载吧!')
        }
      })
    },
    changeView () {
      if (this.view === 'Film') {
        this.getAllSites()
        if (this.setting.view === 'picture') {
          if (this.$refs.filmWaterfall) {
            this.$refs.filmWaterfall.refresh()
          }
          this.getPage().then(() => {
            this.infiniteId += 1
          })
        }
      }
    },
    querySearch (queryString, cb) {
      if (this.searchList.length === 0) return
      var searchList = this.searchList.slice(0, -1)
      var results = queryString ? searchList.filter(this.createFilter(queryString)) : this.searchList
      // 调用 callback 返回建议列表的数据
      cb(results)
    },
    createFilter (queryString) {
      return (item) => {
        return (item.keywords.toLowerCase().indexOf(queryString.toLowerCase()) === 0)
      }
    },
    addSearchRecord () {
      const wd = this.searchTxt
      if (wd) {
        search.find({ keywords: wd }).then(res => {
          if (!res) {
            search.add({ keywords: wd })
          }
          this.getSearchHistory()
        })
      }
    },
    clearSearchHistory () {
      search.clear().then(res => {
        this.getSearchHistory()
      })
    },
    getSearchHistory () {
      search.all().then(res => {
        this.searchList = res.reverse()
        this.searchList.push({ id: this.searchList.length + 1, keywords: '清除历史记录...' })
      })
    },
    searchEvent () {
      const wd = this.searchTxt
      if (this.setting.searchGroup !== this.searchGroup) {
        this.setting.searchGroup = this.searchGroup
        setting.update(this.setting)
      }
      if (!wd) return
      this.searchID += 1
      var searchSites = []
      if (this.searchGroup === '站内') searchSites.push(this.site)
      if (this.searchGroup === '全站') searchSites = this.sites
      if (!searchSites.length) {
        searchSites = this.sites.filter(site => site.group === this.searchGroup)
      }
      this.searchContents = []
      this.pagecount = 0
      this.show.find = true
      this.show.class = false
      this.statusText = ' '
      if (wd) {
        searchSites.forEach(site => {
          const id = this.searchID
          zy.search(site.key, wd).then(res => {
            if (id !== this.searchID) return
            const type = Object.prototype.toString.call(res)
            if (type === '[object Array]') {
              res.forEach(element => {
                zy.detail(site.key, element.id).then(detailRes => {
                  detailRes.site = site
                  if (id !== this.searchID) return
                  this.searchContents.push(detailRes)
                  this.searchContents.sort(function (a, b) {
                    return a.site.id - b.site.id
                  })
                  this.statusText = '暂无数据'
                })
              })
            }
            if (type === '[object Object]') {
              zy.detail(site.key, res.id).then(detailRes => {
                detailRes.site = site
                if (id !== this.searchID) return
                this.searchContents.push(detailRes)
                this.searchContents.sort(function (a, b) {
                  return a.site.id - b.site.id
                })
                this.statusText = '暂无数据'
              })
            }
          })
        })
      }
    },
    searchAndRecord () {
      this.addSearchRecord()
      this.searchEvent()
    },
    searchChangeEvent () {
      if (this.searchTxt.length >= 1) {
        this.show.class = false
      } else {
        this.show.class = true
        this.searchContents = []
        this.show.find = false
        if (this.setting.view === 'picture' && this.$refs.filmWaterfall) {
          this.$refs.filmWaterfall.refresh()
        } else {
          this.getClass().then(res => {
            if (res) {
              this.infiniteId += 1
            }
          })
        }
      }
    },
    getAllSites () {
      sites.all().then(res => {
        if (res.length <= 0) {
          this.site = {}
          this.type = {}
          this.list = []
        } else {
          this.sites = res.filter(item => item.isActive)
          if (this.site === undefined || !this.sites.some(x => x.key === this.site.key)) {
            this.site = this.sites[0]
            this.selectedSiteName = this.sites[0].name
          }
        }
        this.searchGroups = [...new Set(this.sites.map(site => site.group))]
        if (this.searchGroups.length === 1) this.searchGroups = []
        this.searchGroups.unshift('站内')
        this.searchGroups.push('全站')
        this.searchGroup = this.setting.searchGroup
        if (this.searchGroup === undefined) setting.find().then(res => { this.searchGroup = res.searchGroup })
      })
    }
  },
  created () {
    this.getAllSites()
    this.getSearchHistory()
  },
  mounted () {
    window.addEventListener('resize', () => {
      if (this.$refs.filmWaterfall && this.view === 'Film') {
        this.$refs.filmWaterfall.resize()
        this.$refs.filmWaterfall.refresh()
        setTimeout(() => {
          this.$refs.filmWaterfall.refresh()
        }, 500)
      }
    }, false)
  }
}
</script>
