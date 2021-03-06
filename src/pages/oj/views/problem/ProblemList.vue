<template>
  <Row :gutter="18" type="flex">
    <Col :span="19">
    <Panel shadow>
      <div slot="title">{{ $t('m.Problem_List') }}</div>
      <div slot="extra">
        <ul class="filter">
          <li>
            <Dropdown @on-click="filterByDifficulty">
              <span>{{ !query.difficulty ? this.$i18n.t('m.Difficulty') : this.$i18n.t('m.' + query.difficulty.charAt(0).toUpperCase() + query.difficulty.slice(1)) }}
                <Icon type="arrow-down-b" />
              </span>
              <Dropdown-menu slot="list">
                <Dropdown-item name="">{{ $t('m.All') }}</Dropdown-item>
                <Dropdown-item name="easy">{{ $t('m.Easy') }}</Dropdown-item>
                <Dropdown-item name="medium">{{ $t('m.Medium') }}</Dropdown-item>
                <Dropdown-item name="hard">{{ $t('m.Hard') }}</Dropdown-item>
              </Dropdown-menu>
            </Dropdown>
          </li>
          <!-- 后端未提供合适API
          <li>
            <i-switch size="large" @on-change="handleTagsVisible">
              <span slot="open">{{$t('m.Tags')}}</span>
              <span slot="close">{{$t('m.Tags')}}</span>
            </i-switch>
          </li> -->
          <li>
            <Input
              v-model="query.keyword"
              placeholder="keyword"
              icon="ios-search-strong"
              @on-enter="filterByKeyword"
              @on-click="filterByKeyword" />
          </li>
          <li>
            <Button type="info" @click="onReset">
              <Icon type="refresh" />
              {{ $t('m.Reset') }}
            </Button>
          </li>
        </ul>
      </div>
      <Table :columns="problemTableColumns"
             :data="problemList"
             :loading="loadings.table"
             style="width: 100%; font-size: 16px;"
             disabled-hover></Table>
    </Panel>
    <Pagination :total="total" :page-size="limit" :current.sync="query.page" @on-change="pushRouter" />
    </Col>

    <Col :span="5">
    <Panel :padding="10">
      <div slot="title" class="taglist-title">{{ $t('m.Tags') }}</div>
      <Button v-for="tag in tagList"
              :key="tag.name"
              :disabled="query.tag === tag.id"
              type="ghost"
              shape="circle"
              class="tag-btn"
              @click="filterByTag(tag.id)">{{ tag.name }}
      </Button>

      <Button id="pick-one" long @click="pickone">
        <Icon type="shuffle" />
        {{ $t('m.Pick_One') }}
      </Button>
    </Panel>
    <Spin v-if="loadings.tag" fix size="large" />
    </Col>
  </Row>
</template>

<script>
  import { mapGetters, mapActions } from 'vuex'
  import api from '@oj/api'
  import { ProblemMixin } from '@oj/components/mixins'
  import Pagination from '@oj/components/Pagination'
  export default {
    name: 'ProblemList',
    components: {
      Pagination
    },
    mixins: [ProblemMixin],
    data () {
      return {
        tagList: [],
        difficulty: '',
        problemMap: null,
        problemTableColumns: [
          {
            title: '#',
            key: 'id',
            width: 80,
            render: (h, params) => {
              return h('Button', {
                props: {
                  type: 'text',
                  size: 'large'
                },
                on: {
                  click: () => {
                    this.$router.push({ name: 'problem-details', params: { problemID: params.row.id }})
                  }
                },
                style: {
                  padding: '2px 0'
                }
              }, params.row.id)
            }
          },
          {
            title: this.$i18n.t('m.Title'),
            width: 400,
            render: (h, params) => {
              return h('Button', {
                props: {
                  type: 'text',
                  size: 'large'
                },
                on: {
                  click: () => {
                    this.$router.push({ name: 'problem-details', params: { problemID: params.row.id }})
                  }
                },
                style: {
                  padding: '2px 0',
                  overflowX: 'auto',
                  textAlign: 'left',
                  width: '100%'
                }
              }, params.row.title)
            }
          },
          {
            title: this.$i18n.t('m.Level'),
            render: (h, params) => {
              const t = params.row.level
              let color = 'blue'
              if (t === 'easy') color = 'green'
              else if (t === 'hard') color = 'yellow'
              return h('Tag', {
                props: {
                  color: color
                }
              }, this.$i18n.t('m.' + params.row.level.charAt(0).toUpperCase() + params.row.level.slice(1)))
            }
          },
          {
            title: this.$i18n.t('m.Total'),
            key: 'submitCount'
          },
          {
            title: this.$i18n.t('m.AC_Rate'),
            key: 'acRate'
          }
        ],
        problemList: [],
        limit: 20,
        total: 0,
        loadings: {
          table: true,
          tag: true
        },
        routeName: '',
        query: {
          keyword: '',
          difficulty: '',
          tag: '',
          page: 1
        }
      }
    },
    computed: {
      ...mapGetters({
        isAuthenticated: 'user/isAuthenticated',
        profile: 'user/profile'
      })
    },
    watch: {
      '$route' (newVal, oldVal) {
        if (newVal !== oldVal) {
          this.init(true)
        }
      },
      'isAuthenticated' (newVal) {
        if (newVal === true) {
          this.init()
        }
      }
    },
    mounted () {
      this.init()
    },
    methods: {
      ...mapActions({
        getProblemMaps: 'user/getSolvedProblems'
      }),
      init (simulate = false) {
        this.routeName = this.$route.name
        const query = this.$route.query
        this.query.difficulty = query.difficulty || null
        this.query.keyword = query.keyword || null
        this.query.tag = query.tag
        this.query.page = parseInt(query.page) || 1
        if (!simulate) {
          this.getTagList()
        }
        if (this.profile.id) {
          this.getProblemMaps({ userId: this.profile.id }).then(res => {
            this.problemMap = res
            this.getProblemList()
          })
          return
        }
        this.getProblemList()
      },
      pushRouter () {
        this.$router.push({
          name: 'problem-list',
          query: this.query
        })
      },
      getProblemList () {
        const offset = this.query.page
        this.loadings.table = true
        const query = {
          level: this.query.difficulty,
          pageModel: {
            limit: this.limit,
            offset: offset,
            paramData: null
          },
          tagId: this.query.tag,
          titleKey: this.query.keyword
        }
        api.getProblemList(query).then(res => {
          this.loadings.table = false
          this.total = res.data.data.total
          this.problemList = res.data.data.records
          if (this.isAuthenticated) {
            this.addStatusColumn(this.problemTableColumns, res.data.data.records)
          }
        }, res => {
          this.loadings.table = false
        })
      },
      getTagList () {
        api.getProblemTagList().then(res => {
          this.tagList = res.data.data
          this.loadings.tag = false
        }, res => {
          this.loadings.tag = false
        })
      },
      filterByTag (tagName) {
        this.query.tag = tagName
        this.query.page = 1
        this.pushRouter()
      },
      filterByDifficulty (difficulty) {
        this.query.difficulty = difficulty
        this.query.page = 1
        this.pushRouter()
      },
      filterByKeyword () {
        this.query.page = 1
        this.pushRouter()
      },
      handleTagsVisible (value) {
        if (value) {
          this.problemTableColumns.push(
            {
              title: this.$i18n.t('m.Tags'),
              align: 'center',
              render: (h, params) => {
                const tags = []
                params.row.tags.forEach(tag => {
                  tags.push(h('Tag', {}, tag))
                })
                return h('div', {
                  style: {
                    margin: '8px 0'
                  }
                }, tags)
              }
            })
        } else {
          this.problemTableColumns.splice(this.problemTableColumns.length - 1, 1)
        }
      },
      onReset () {
        this.$router.push({ name: 'problem-list' })
      },
      pickone () {
        const total = this.total
        const randomNum = Math.floor(Math.random() * total) + 1
        const consult = Math.floor(randomNum / this.limit)
        const remainder = randomNum % this.limit
        let page = consult
        let limit = remainder
        if (!consult) {
          page = 1
        }
        if (!remainder) {
          page -= 1
          limit = this.limit
        }
        const query = {
          level: null,
          pageModel: {
            limit: this.limit,
            offset: page,
            paramData: null
          },
          tagId: null,
          titleKey: null
        }
        api.getProblemList(query).then(res => {
          this.$success('Good Luck')
          const current = res.data.data.records[limit - 1]
          this.$router.push({ name: 'problem-details', params: { problemID: current.id }})
        })
      }
    }
  }
</script>

<style scoped lang="less">
  .taglist-title {
    margin-left: -10px;
    margin-bottom: -10px;
  }

  .tag-btn {
    margin-right: 5px;
    margin-bottom: 10px;
  }

  #pick-one {
    margin-top: 10px;
  }
  /deep/.ivu-card {
    color: var(--font-color-white);
    background: var(--table-card-top);
  }
  /deep/.ivu-card-body .ivu-table th {
    color: var(--font-color-origin);
    background: var(--table-card-head);
  }
  /deep/.ivu-table td {
    color: var(--font-color-origin);
    background: var(--table-card-body);
  }
  /deep/.ivu-btn-text {
    color: var(--font-color-origin);
  }
  /deep/.ivu-input {
    background: var(--search-bg-color);
    color: var(--header-input-placeholder-color);
  }
  /deep/.ivu-btn-ghost {
    color: var(--font-color-white);
  }
  /deep/.ivu-dropdown-item:hover {
    background: var(--dropdown-difficulty);
  }
  /deep/.ivu-dropdown-item {
    color: var(--font-color-white);
    border-bottom: #fff;
  }
  /deep/.ivu-select-dropdown {
    background-color: var(--dropdown-diff-bg-color);
  }
</style>
