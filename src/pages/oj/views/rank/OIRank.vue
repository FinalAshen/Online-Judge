<template>
  <Row type="flex" justify="space-around">
    <Col :span="22">
    <Panel :padding="10">
      <div slot="title">{{ $t('m.OI_Ranklist') }}</div>
      <div class="echarts">
        <ECharts ref="chart" :options="options" auto-resize />
      </div>
    </Panel>
    <Table :data="dataRank" :columns="columns" size="large" />
    <Pagination
      :total="total"
      :page-size.sync="limit"
      :current.sync="page"
      show-sizer
      @on-change="getRankData"
      @on-page-size-change="getRankData(1)"></Pagination>
    </Col>
  </Row>
</template>

<script>
  import api from '@oj/api'
  import Pagination from '@oj/components/Pagination'
  import utils from '@/utils/utils'
  import { RULE_TYPE } from '@/utils/constants'

  export default {
    name: 'AcmRank',
    components: {
      Pagination
    },
    data () {
      return {
        page: 1,
        limit: 30,
        total: 0,
        dataRank: [],
        columns: [
          {
            align: 'center',
            width: 60,
            render: (h, params) => {
              return h('span', {}, params.index + (this.page - 1) * this.limit + 1)
            }
          },
          {
            title: this.$i18n.t('m.User_User'),
            align: 'center',
            render: (h, params) => {
              return h('a', {
                style: {
                  'display': 'inline-block',
                  'max-width': '200px'
                },
                on: {
                  click: () => {
                    this.$router.push(
                      {
                        name: 'user-home',
                        query: { id: params.row.id }
                      })
                  }
                }
              }, params.row.username)
            }
          },
          {
            title: this.$i18n.t('m.mood'),
            align: 'center',
            render: (h, params) => {
              return h('span', params.row.mood && params.row.mood.trim() ? params.row.mood : '-')
            }
          },
          {
            title: this.$i18n.t('m.Score'),
            align: 'center',
            key: 'ratingScore'
          },
          {
            title: this.$i18n.t('m.AC'),
            align: 'center',
            key: 'acNum'
          },
          {
            title: this.$i18n.t('m.Total'),
            align: 'center',
            key: 'submitCount'
          },
          {
            title: this.$i18n.t('m.Rating'),
            align: 'center',
            render: (h, params) => {
              return h('span', utils.getACRate(params.row.acNum, params.row.submitCount))
            }
          }
        ],
        options: {
          tooltip: {
            trigger: 'axis'
          },
          legend: {
            data: [this.$i18n.t('m.Score')]
          },
          grid: {
            x: '3%',
            x2: '3%'
          },
          toolbox: {
            show: true,
            feature: {
              dataView: { show: true, readOnly: true },
              magicType: { show: true, type: ['line', 'bar'] },
              saveAsImage: { show: true }
            },
            right: '10%'
          },
          calculable: true,
          xAxis: [
            {
              type: 'category',
              data: ['root'],
              boundaryGap: true,
              axisLabel: {
                interval: 0,
                showMinLabel: true,
                showMaxLabel: true,
                align: 'center',
                formatter: (value, index) => {
                  return utils.breakLongWords(value, 14)
                }
              },
              axisTick: {
                alignWithLabel: true
              }
            }
          ],
          yAxis: [
            {
              type: 'value'
            }
          ],
          series: [
            {
              name: this.$i18n.t('m.Score'),
              type: 'bar',
              data: [0],
              barMaxWidth: '80',
              markPoint: {
                data: [
                  { type: 'max', name: 'max' }
                ]
              }
            }
          ]
        }
      }
    },
    mounted () {
      this.getRankData(1)
    },
    methods: {
      getRankData (page) {
        const offset = page
        const bar = this.$refs.chart
        bar.showLoading({ maskColor: 'rgba(250, 250, 250, 0.8)' })
        api.getUserRank(offset, this.limit, RULE_TYPE.OI).then(res => {
          if (page === 1) {
            this.changeCharts(res.data.data.records.slice(0, 10))
          }
          this.total = res.data.data.total
          this.dataRank = res.data.data.records
          bar.hideLoading()
        })
      },
      changeCharts (rankData) {
        const [usernames, scores] = [[], []]
        rankData.forEach(ele => {
          usernames.push(ele.username)
          scores.push(ele.ratingScore)
        })
        this.options.xAxis[0].data = usernames
        this.options.series[0].data = scores
      }
    }
  }
</script>

<style scoped lang="less">
  /deep/.ivu-card {
    background: var(--table-card-canvas-top);
    color: var(--font-color-white);
  }
  /deep/.ivu-table th {
    color: var(--font-color-origin);
    background: var(--table-card-head);
  }
  /deep/.ivu-table td {
    color: var(--font-color-origin);
    background: var(--table-card-body) !important;
  }
  .echarts {
    margin: 0 auto;
    width: 95%;
    height: 400px;
  }
</style>
