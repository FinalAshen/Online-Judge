<template>
  <Row type="flex">
    <Col :span="24">
    <Panel id="contest-card" shadow>
      <div slot="title">{{ $t('m.Contests') }}</div>
      <div slot="extra">
        <ul class="filter">
          <li>
            <Dropdown @on-click="onRuleChange">
              <span>{{ ruleTypeName(query.rule_type) }}
                <Icon type="arrow-down-b" />
              </span>
              <Dropdown-menu slot="list">
                <Dropdown-item :name="null">{{ $t('m.All') }}</Dropdown-item>
                <Dropdown-item :name="CONTEST_QUERY_VALUE.WARM_UP">{{ $t('m.Warm_Up') }}</Dropdown-item>
                <Dropdown-item :name="CONTEST_QUERY_VALUE.INTEGRAL">{{ $t('m.Integral') }}</Dropdown-item>
                <Dropdown-item :name="CONTEST_QUERY_VALUE.ACM_ICPC">{{ $t('m.ACM') }}</Dropdown-item>
              </Dropdown-menu>
            </Dropdown>
          </li>
          <li>
            <Dropdown @on-click="onSignRuleChange">
              <span>{{ signUpRuleName(query.sign_rule) }}
                <Icon type="arrow-down-b" />
              </span>
              <Dropdown-menu slot="list">
                <Dropdown-item :name="null">{{ $t('m.All') }}</Dropdown-item>
                <Dropdown-item :name="CONTEST_QUERY_VALUE.PUBLIC">{{ $t('m.Public') }}</Dropdown-item>
                <Dropdown-item :name="CONTEST_QUERY_VALUE.PROOF">{{ $t('m.Proof') }}</Dropdown-item>
                <Dropdown-item :name="CONTEST_QUERY_VALUE.KEY">{{ $t('m.Key') }}</Dropdown-item>
              </Dropdown-menu>
            </Dropdown>
          </li>
          <li>
            <Dropdown @on-click="onStatusChange">
              <span>{{ statusName(query.status) }}
                <Icon type="arrow-down-b" />
              </span>
              <Dropdown-menu slot="list">
                <Dropdown-item :name="null">{{ $t('m.All') }}</Dropdown-item>
                <Dropdown-item :name="CONTEST_QUERY_VALUE.EX_UNDERWAY">{{ $t('m.Underway') }}</Dropdown-item>
                <Dropdown-item :name="CONTEST_QUERY_VALUE.EX_NOT_START">{{ $t('m.Not_Started') }}</Dropdown-item>
                <Dropdown-item :name="CONTEST_QUERY_VALUE.EX_ENDED">{{ $t('m.Ended') }}</Dropdown-item>
              </Dropdown-menu>
            </Dropdown>
          </li>
          <li>
            <Input
              id="keyword"
              v-model="query.keyword"
              icon="ios-search-strong"
              placeholder="Keyword"
              @on-enter="changeRoute"
              @on-click="changeRoute" />
          </li>
        </ul>
      </div>
      <p v-if="contests.length == 0" id="no-contest">{{ $t('m.No_contest') }}</p>
      <ol id="contest-list">
        <li v-for="contest in contests" :key="contest.title">
          <Row type="flex" justify="space-between" align="middle">
            <img class="trophy" src="../../../../assets/Cup.png">
            <Col :span="18" class="contest-main">
            <p class="title">
              <a class="entry" @click.stop="goContest(contest)">
                {{ contest.title }}
              </a>
              <template v-if="contest.signUpRule === CONTEST_QUERY_VALUE.KEY">
                <Icon type="ios-locked-outline" size="20" />
              </template>
              <template v-else-if="contest.signUpRule === CONTEST_QUERY_VALUE.PUBLIC">
                <Icon type="social-snapchat-outline" size="20" />
              </template>
              <template v-else>
                <Icon type="card" size="20" />
              </template>
            </p>
            <ul class="detail">
              <li>
                <Icon type="calendar" color="#3091f2" />
                {{ contest.startTime }}
              </li>
              <li>
                <Icon type="android-time" color="#3091f2" />
                {{ getDuration(contest.startTime, contest.endTime) }}
              </li>
              <li>
                <Button size="small" shape="circle" @click="onRuleChange(contest.rankModel)">
                  {{ contest.rankModel }}
                </Button>
              </li>
            </ul>
            </Col>
            <Col :span="4" style="text-align: center">
            <Tag :color="showStatus(contest.startTime, contest.endTime).color" type="dot">{{ $t('m.' + showStatus(contest.startTime, contest.endTime).name.replace(/\s+/g, '_')) }}</Tag>
            </Col>
          </Row>
        </li>
      </ol>
    </Panel>
    <Pagination :total="total" :page-size="limit" :current.sync="page" @on-change="getContestList" />
    </Col>
  </Row>

</template>

<script>
  import api from '@oj/api'
  import { mapGetters } from 'vuex'
  import utils from '@/utils/utils'
  import Pagination from '@/pages/oj/components/Pagination'
  import time from '@/utils/time'
  import { CONTEST_STATUS_REVERSE, CONTEST_QUERY_VALUE } from '@/utils/constants'
  import { NormalMixin } from '@oj/components/mixins'

  const limit = 8

  export default {
    name: 'ContestList',
    components: {
      Pagination
    },
    mixins: [NormalMixin],
    data () {
      return {
        page: 1,
        query: {
          status: '',
          keyword: '',
          rule_type: '',
          sign_rule: ''
        },
        limit: limit,
        total: 0,
        rows: '',
        contests: [],
        currentTime: null,
        cur_contest_id: '',
        CONTEST_STATUS_REVERSE,
        CONTEST_QUERY_VALUE
      }
    },
    computed: {
      ...mapGetters({
        isAuthenticated: 'user/isAuthenticated'
      })
    },
    watch: {
      '$route' (newVal, oldVal) {
        if (newVal !== oldVal) {
          this.init()
        }
      }
    },
    mounted () {
      this.init()
    },

    methods: {
      init () {
        const route = this.$route.query
        this.query.status = route.status
        this.query.rule_type = route.rule_type
        this.query.keyword = route.keyword
        this.query.rule = route.rule
        this.query.sign_rule = route.sign_rule
        this.page = parseInt(route.page) || 1
        this.getContestList()
      },
      getContestList () {
        const data = {
          contestRunningStatus: this.query.status || null,
          pageModel: {
            limit: this.limit,
            offset: this.page,
            paramData: null
          },
          rankModel: this.query.rule_type || null,
          signUpRule: this.query.sign_rule || null,
          titleKey: this.query.keyword || null
        }
        api.getContestList(data).then(res => {
          this.contests = res.data.data.records
          this.currentTime = utils.formatDate(new Date(res.headers.date))
          this.total = res.data.data.total
        })
      },
      changeRoute () {
        const query = Object.assign({}, this.query)
        query.page = this.page
        this.$router.push({
          name: 'contest-list',
          query: utils.filterEmptyValue(query)
        })
      },
      onSignRuleChange (signRule) {
        this.query.sign_rule = signRule
        this.page = 1
        this.changeRoute()
      },
      onRuleChange (rule) {
        this.query.rule_type = rule
        this.page = 1
        this.changeRoute()
      },
      onStatusChange (status) {
        this.query.status = status
        this.page = 1
        this.changeRoute()
      },
      showStatus (startTime, endTime) {
        const startCTime = time.difference(startTime, this.currentTime)._milliseconds
        const endCTime = time.difference(endTime, this.currentTime)._milliseconds

        if (startCTime > 0) {
          return CONTEST_STATUS_REVERSE[CONTEST_QUERY_VALUE.EX_NOT_START]
        } else if (endCTime > 0) {
          return CONTEST_STATUS_REVERSE[CONTEST_QUERY_VALUE.EX_UNDERWAY]
        } else {
          return CONTEST_STATUS_REVERSE[CONTEST_QUERY_VALUE.EX_ENDED]
        }
      },
      goContest (contest) {
        this.cur_contest_id = contest.id
        if (!this.isAuthenticated) {
          this.$error(this.$i18n.t('m.Please_login_first'))
          this.$store.dispatch('changeModalStatus', { visible: true })
        } else {
          this.$router.push({ name: 'contest-details', params: { contestID: contest.id }})
        }
      },

      getDuration (startTime, endTime) {
        return time.duration(startTime, endTime)
      }
    }
  }
</script>
<style lang="less" scoped>
  /deep/.ivu-card {
    background: var(--table-card-top);
  }
  /deep/.ivu-select-dropdown {
    background: var(--dropdown-contest-bg-color);
  }
  /deep/.ivu-dropdown-item {
    color: var(--font-color-white);
  }
  /deep/.ivu-dropdown-item:hover {
    background-color: var(--item-contest-dropdown);
  }
  #contest-card {
    color: var(--font-color-white);
    #keyword {
      width: 80%;
      margin-right: 30px;
    }
    #no-contest {
      text-align: center;
      font-size: 16px;
      padding: 20px;
    }
    #contest-list {
      > li {
        padding: 20px;
        border-bottom: 1px solid rgba(187, 187, 187, 0.5);
        list-style: none;

        .trophy {
          height: 40px;
          margin-left: 10px;
          margin-right: -20px;
        }
        .contest-main {
          .title {
            font-size: 18px;
            a.entry {
              color: var(--font-color-white);
              &:hover {
                color: #2d8cf0;
                border-bottom: 1px solid #2d8cf0;
              }
            }
          }
          li {
            display: inline-block;
            padding: 10px 0 0 10px;
            &:first-child {
              padding: 10px 0 0 0;
            }
          }
        }
      }
    }
  }
</style>
