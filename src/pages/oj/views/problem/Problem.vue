<template>
  <div class="flex-container spin-container">
    <div id="problem-main">
      <Spin v-if="isLoading" fix />
      <!--problem main-->
      <Panel :padding="40" shadow>
        <div slot="title">{{ problem.title }}</div>
        <div v-katex id="problem-content" class="markdown-body">
          <p class="title">{{ $t('m.Description') }}</p>
          <p class="content" v-html="problem.description" /><p class="title">{{ $t('m.Input') }} <span v-if="problem.ioMode=='File IO'">({{ $t('m.FromFile') }}: {{ problem.ioMode }})</span></p>
          <p class="content" v-html="problem.inputDescription" /><p class="title">{{ $t('m.Output') }} <span v-if="problem.ioMode=='File IO'">({{ $t('m.ToFile') }}: {{ problem.ioMode }})</span></p>
          <p class="content" v-html="problem.outputDescription" />
          <div v-for="(sample, index) of problem.samples" :key="index">
            <div class="flex-container sample">
              <div class="sample-input">
                <p class="title">{{ $t('m.Sample_Input') }} {{ index + 1 }}
                  <a
                    v-clipboard:copy="sample.input.replace(/<br\/>/ig, '\n')"
                    v-clipboard:success="onCopy"
                    v-clipboard:error="onCopyError"
                    class="copy">
                    <Icon type="clipboard" />
                  </a>
                </p>
                <!--todo: pre无法识别\n-->
                <pre v-html="sample.input" />
              </div>
              <div class="sample-output">
                <p class="title">{{ $t('m.Sample_Output') }} {{ index + 1 }}</p>
                <pre v-html="sample.output" />
              </div>
            </div>
          </div>

          <div v-if="problem.hint">
            <p class="title">{{ $t('m.Hint') }}</p>
            <Card dis-hover>
            <div class="content" v-html="problem.hint" /></Card>
          </div>

          <div v-if="problem.sourceName">
            <p class="title">{{ $t('m.Source') }}</p>
            <p class="content">{{ problem.sourceName }}</p>
          </div>

        </div>
      </Panel>
      <!--problem main end-->
      <Card id="submit-code" :padding="20" dis-hover>
        <CodeMirror
          :value.sync="code"
          :languages="problem.language"
          :language="language"
          :code-template="problem.codeTemplate"
          :theme="theme"
          @resetCode="onResetToTemplate"
          @changeTheme="onChangeTheme"
          @changeLang="onChangeLang" />
        <Row type="flex" justify="space-between">
          <Col :span="10">
          <div v-if="statusVisible" class="status">
            <template v-if="!contestID || (contestID && OIContestRealTimePermission)">
              <span>{{ $t('m.Status') }}</span>
              <Tag :color="submissionStatus.color" type="dot" @click.native="handleRoute('/status/'+submissionId)">
                {{ submissionStatus.text }}
              </Tag>
            </template>
            <template v-else-if="contestID && !OIContestRealTimePermission">
              <Alert type="success" show-icon>{{ $t('m.Submitted_successfully') }}</Alert>
            </template>
          </div>
          <div v-else-if="problem.my_status === 0">
            <Alert type="success" show-icon>{{ $t('m.You_have_solved_the_problem') }}</Alert>
          </div>
          <div v-else-if="contestID && !OIContestRealTimePermission && submissionExists">
            <Alert type="success" show-icon>{{ $t('m.You_have_submitted_a_solution') }}</Alert>
          </div>
          <div v-if="contestEnded">
            <Alert type="warning" show-icon>{{ $t('m.Contest_has_ended') }}</Alert>
          </div>
          </Col>
          <Col :span="12">
          <template v-if="captchaRequired">
            <div class="captcha-container">
              <Tooltip v-if="captchaRequired" content="Click to refresh" placement="top">
                <img :src="captchaSrc" @click="getCaptchaSrc">
              </Tooltip>
              <Input v-model="captchaCode" class="captcha-code" />
            </div>
          </template>
          <Button
            :loading="submitting"
            :disabled="!isAuthenticated || ((contestID && problemSubmitDisabled) || submitted)"
            class="fl-right"
            type="warning"
            icon="edit"
            @click="submitCode">
            <span v-if="submitting">{{ $t('m.Submitting') }}</span>
            <span v-else>{{ $t('m.Submit') }}</span>
          </Button>
          </Col>
        </Row>
      </Card>
    </div>

    <div id="right-column">
      <VerticalMenu @on-click="handleRoute">
        <template v-if="contestID">
          <VerticalMenu-item :route="{name: 'contest-problem-list', params: {contestID: contestID}}">
            <Icon type="ios-photos" />
            {{ $t('m.Problems') }}
          </VerticalMenu-item>

          <VerticalMenu-item :route="{name: 'contest-announcement-list', params: {contestID: contestID}}">
            <Icon type="chatbubble-working" />
            {{ $t('m.Announcements') }}
          </VerticalMenu-item>
        </template>

        <VerticalMenu-item
          v-if="!contestID || OIContestRealTimePermission"
          :route="submissionRoute"
          class="cursor-sub">
          <Icon type="navicon-round" />
          {{ $t('m.Submissions') }}
        </VerticalMenu-item>

        <template v-if="contestID">
          <VerticalMenu-item
            v-if="!contestID || OIContestRealTimePermission"
            :route="{name: 'contest-rank', params: {contestID: contestID}}">
            <Icon type="stats-bars" />
            {{ $t('m.Rankings') }}
          </VerticalMenu-item>
          <VerticalMenu-item :route="{name: 'contest-details', params: {contestID: contestID}}">
            <Icon type="home" />
            {{ $t('m.View_Contest') }}
          </VerticalMenu-item>
        </template>
      </VerticalMenu>

      <Card id="info">
        <div slot="title" class="header">
          <Icon type="information-circled" />
          <span class="card-title">{{ $t('m.Information') }}</span>
        </div>
        <ul>
          <li><p>{{ $t('m.ID') }}</p>
          <p>{{ problem.id }}</p></li>
          <li>
            <p>{{ $t('m.Time_Limit') }}</p>
          <p>{{ problem.timeLimit }}MS</p></li>
          <li>
            <p>{{ $t('m.Memory_Limit') }}</p>
          <p>{{ problem.memoryLimit }}MB</p></li>
          <li /><li>
            <p>{{ $t('m.IOMode') }}</p>
            <p>{{ problem.ioMode }}</p>
          </li>
          <li>
            <p>{{ $t('m.Created') }}</p>
          <p>{{ problem.creator }}</p></li>
          <li v-if="problem.level">
            <p>{{ $t('m.Level') }}</p>
          <p>{{ $t('m.' + problem.level.charAt(0).toUpperCase() + problem.level.slice(1)) }}</p></li>
          <li v-if="problem.total_score">
            <p>{{ $t('m.Score') }}</p>
            <p>{{ problem.total_score }}</p>
          </li>
          <li>
            <p>{{ $t('m.Tags') }}</p>
            <p>
              <Poptip trigger="hover" placement="left-end">
                <a>{{ $t('m.Show') }}</a>
                <div slot="content">
                  <Tag v-for="tag in problem.tagList" :key="tag.tagId">{{ tag.name }}</Tag>
                </div>
              </Poptip>
            </p>
          </li>
        </ul>
      </Card>

      <Card v-if="!contestID || OIContestRealTimePermission" id="pieChart" :padding="0">
        <div slot="title">
          <Icon type="ios-analytics" />
          <span class="card-title">{{ $t('m.Statistic') }}</span>
          <Button id="detail" type="ghost" size="small" @click="graphVisible = !graphVisible">Details</Button>
        </div>
        <div class="echarts">
          <ECharts :options="pie" />
        </div>
      </Card>
    </div>

    <Modal v-model="graphVisible">
      <div id="pieChart-detail">
        <ECharts :options="largePie" :init-options="largePieInitOpts" />
      </div>
      <div slot="footer">
        <Button type="ghost" @click="graphVisible=false">{{ $t('m.Close') }}</Button>
      </div>
    </Modal>
  </div>
</template>

<script>
  import { mapGetters, mapActions } from 'vuex'
  import { types } from '../../../../store'
  import CodeMirror from '@oj/components/CodeMirror.vue'
  import storage from '@/utils/storage'
  import { FormMixin } from '@oj/components/mixins'
  import { JUDGE_STATUS, CONTEST_STATUS, buildProblemCodeKey, CONSTANTS_TEMPLATE, JUDGE_SERVER } from '@/utils/constants'
  import { exFormatText } from '@/utils/JudgeServer/poj'
  import api from '@oj/api'
  import { pie, largePie } from './chartData'
  export default {
    name: 'Problem',
    components: {
      CodeMirror
    },
    mixins: [FormMixin],
    data () {
      return {
        isLoading: true,
        submissionResult: '',
        statusVisible: false,
        captchaRequired: false,
        graphVisible: false,
        submissionExists: false,
        captchaCode: '',
        captchaSrc: '',
        contestID: '',
        problemID: '',
        submitting: false,
        code: '',
        language: 'C++',
        theme: 'monokai',
        submissionId: '',
        submitted: false,
        result: {
          submitStatus: 'Judging',
          judge: null
        },
        problem: {
          title: '',
          description: '',
          hint: '',
          my_status: '',
          template: {},
          languages: [],
          created_by: {
            username: ''
          },
          tags: [],
          io_mode: { 'io_mode': 'Standard IO' }
        },
        pie: pie,
        largePie: largePie,
        // echarts 无法获取隐藏dom的大小，需手动指定
        largePieInitOpts: {
          width: '500',
          height: '480'
        }
      }
    },
    beforeRouteEnter (to, from, next) {
      const problemCode = storage.get(buildProblemCodeKey(to.params.problemID, to.params.contestID))
      if (problemCode) {
        next(vm => {
          vm.language = problemCode.language
          vm.code = problemCode.code
          vm.theme = problemCode.theme
        })
      } else {
        next()
      }
    },
    computed: {
      ...mapGetters({
        problemSubmitDisabled: 'contest/problemSubmitDisabled',
        OIContestRealTimePermission: 'contest/OIContestRealTimePermission',
        contestStatus: 'contest/contestStatus',
        profile: 'user/profile',
        isAuthenticated: 'user/isAuthenticated'
      }),
      contest () {
        return this.$store.state.contest.contest
      },
      contestEnded () {
        return this.contestStatus === CONTEST_STATUS.ENDED
      },
      submissionStatus () {
        return {
          text: this.result.submitStatus ? JUDGE_STATUS[this.result.submitStatus]['name'] : '',
          color: this.result.submitStatus ? JUDGE_STATUS[this.result.submitStatus]['color'] : ''
        }
      },
      submissionRoute () {
        if (this.contestID) {
          return { name: 'contest-submission-list', query: { problemID: this.problemID }}
        } else {
          return { name: 'submission-list', query: { problemID: this.problemID }}
        }
      }
    },
    beforeRouteLeave (to, from, next) {
      // 防止切换组件后仍然不断请求
      clearTimeout(this.refreshStatus)
      this.$store.commit(`contest/${types.CHANGE_CONTEST_ITEM_VISIBLE}`, { menu: true })
      storage.set(buildProblemCodeKey(this.problem.id, from.params.contestID), {
        code: this.code,
        language: this.language,
        theme: this.theme
      })
      next()
    },
    watch: {
      '$route' () {
        this.init()
      }
    },
    mounted () {
      this.$store.commit(`contest/${types.CHANGE_CONTEST_ITEM_VISIBLE}`, { menu: false })
      this.init()
      window.addEventListener('beforeunload', () => {
        storage.set(buildProblemCodeKey(this.problem.id, this.contestID), {
          code: this.code,
          language: this.language,
          theme: this.theme
        })
      })
    },
    methods: {
      ...mapActions(['changeDomTitle']),
      init () {
        this.$Loading.start()
        this.isLoading = true
        clearTimeout(this.refreshStatus)
        this.contestID = this.$route.params.contestID
        this.problemID = this.$route.params.problemID
        const problemDetail = this.cache.get(buildProblemCodeKey(this.problemID, this.contestID))
        const func = this.$route.name === 'problem-details' ? 'getProblem' : 'getContestProblem'
        const ID = {
          contestID: this.contestID,
          problemID: this.problemID
        }
        if (problemDetail) {
          this.$Loading.finish()
          this.isLoading = false
          this.problem = problemDetail.problem
          this.submissionExists = problemDetail.submissionExists
          this.changeDomTitle({ title: this.problem.title })
          this.changePie(this.problem)
          if (this.code !== '') {
            return
          }
          // try to load problem template
          // this.language = firstLanguage
          // this.code = firstCode
          return
        }
        api[func](ID).then(res => {
          this.$Loading.finish()
          this.isLoading = false
          const problem = res.data.data
          this.changeDomTitle({ title: problem.title })
          if (this.profile.id) {
            api.getSolvedProblems(this.profile.id).then(res => {
              if (res.data.data && res.data.data.indexOf(ID.problemID) > -1) {
                this.submissionExists = true
              }
            })
          }
          problem.language = problem.language.split(',').sort((a, b) => a - b)
          const Base64 = require('js-base64').Base64
          const template = problem.codeTemplate ? JSON.parse(Base64.decode(problem.codeTemplate)) : null
          const templateLists = {}
          var firstLanguage = ''
          var firstCode = ''
          if (template) {
            Object.keys(template).forEach(res => {
              const nowValue = template[res]
              const nowName = template[res].key
              var mode = ''
              Object.keys(CONSTANTS_TEMPLATE).forEach(res => {
                const now = CONSTANTS_TEMPLATE[res]
                if (now.name === nowName) {
                  mode = now.content_type
                }
              })
              templateLists[nowName] = {
                checked: false,
                code: nowValue.value,
                mode,
                language: nowName
              }
              if (!firstLanguage) {
                firstLanguage = nowName
                firstCode = nowValue.value
              }
            })
          } else {
            for (const lang of problem.language) {
              for (const template of CONSTANTS_TEMPLATE) {
                if (template.name === lang) {
                  templateLists[lang] = {
                    checked: false,
                    code: template.template,
                    mode: template.content_type,
                    language: lang
                  }
                  if (!firstLanguage) {
                    firstLanguage = lang
                    firstCode = template.template
                  }
                  break
                }
              }
            }
          }
          problem.codeTemplate = templateLists
          problem.description = exFormatText(problem.description)
          problem.hint = exFormatText(problem.hint)
          problem.inputDescription = exFormatText(problem.inputDescription)
          problem.outputDescription = exFormatText(problem.outputDescription)
          // 延迟
          const samples = []
          var inputSamples = []
          var outputSamples = []
          // 样例格式统一
          if (problem.judgeTypeName === JUDGE_SERVER.POJ) {
            inputSamples.push(problem.inputSamples)
            outputSamples.push(problem.outputSamples)
          } else if (problem.judgeTypeName === JUDGE_SERVER.QDOJ) {
            inputSamples = JSON.parse(problem.inputSamples)
            outputSamples = JSON.parse(problem.outputSamples)
          }
          const samplesLength = Math.min(inputSamples.length, outputSamples.length)
          for (let i = 0; i < samplesLength; ++i) {
            samples.push({
              input: inputSamples[i].replace(/\\n/ig, '<br/>'),
              output: outputSamples[i].replace(/\\n/ig, '<br/>')
            })
          }
          problem.samples = samples
          this.problem = problem

          this.cache.set(buildProblemCodeKey(this.problem.id, this.contestID), { problem: this.problem, submissionExists: this.submissionExists })

          this.changePie(problem)
          // 在beforeRouteEnter中修改了, 说明本地有code，无需加载template
          if (this.code !== '') {
            return
          }
          // try to load problem template
          this.language = firstLanguage
          this.code = firstCode
        }, (_) => {
          this.$router.go(-1)
          this.$Loading.error()
          this.isLoading = false
        })
      },
      changePie (problemData) {
        const status = problemData.statistic
        const acNum = status.Accepted
        const data = [
          { name: 'WA', value: problemData.submitCount - acNum },
          { name: 'AC', value: acNum }
        ]
        this.pie.series[0].data = data
        // 只把大图的AC selected下，这里需要做一下deepcopy
        const data2 = JSON.parse(JSON.stringify(data))
        data2[1].selected = true
        this.largePie.series[1].data = data2

        // 根据结果设置legend,没有提交过的legend不显示
        const legend = Object.keys(status).map(ele => {
          return JUDGE_STATUS[ele].short
        })
        if (legend.length === 0) {
          legend.push('AC', 'WA')
        }
        this.largePie.legend.data = legend

        // 把ac的数据提取出来放在最后
        const acCount = problemData.acCount

        const largePieData = []
        Object.keys(status).forEach(ele => {
          largePieData.push({ name: JUDGE_STATUS[ele].short, value: status[ele] })
        })
        largePieData.push({ name: 'AC', value: acCount })
        this.largePie.series[0].data = largePieData
      },
      handleRoute (route) {
        this.$router.push(route)
      },
      onChangeLang (newLang) {
        if (this.problem.codeTemplate[newLang]) {
          this.code = this.problem.codeTemplate[newLang].code
        }
        this.language = newLang
      },
      onChangeTheme (newTheme) {
        this.theme = newTheme
      },
      onResetToTemplate () {
        this.$Modal.confirm({
          content: this.$i18n.t('m.Are_you_sure_you_want_to_reset_your_code'),
          onOk: () => {
            const template = this.problem.template
            if (template && template[this.language]) {
              this.code = template[this.language]
            } else {
              this.code = ''
            }
          }
        })
      },
      buildStatusQuery () {
        return {
          creatorKey: this.profile.username,
          pageModel: {
            limit: 1,
            offset: 1,
            paramData: null
          },
          problemId: this.problem.id,
          submitStatus: null
        }
      },
      checkSubmissionStatus () {
        // 使用setTimeout避免一些问题
        if (this.refreshStatus) {
          clearTimeout(this.refreshStatus)
        }
        const checkStatus = () => {
          api.getSubmissionList(this.buildStatusQuery()).then(res => {
            let nowArr = res.data.data.records
            nowArr = nowArr && nowArr.length ? nowArr[0] : null
            // 首次提交 + 往后提交
            if (nowArr && (!this.result.judge || this.result.judge.id !== nowArr.id) && JUDGE_STATUS[nowArr.submitStatus].isResult) {
              this.result.judge = nowArr
              this.result.submitStatus = nowArr.submitStatus
              this.submitting = false
              this.submitted = false
              clearTimeout(this.refreshStatus)
              this.init()
            } else {
              if (nowArr) {
                this.result.submitStatus = nowArr.submitStatus
              }
              this.refreshStatus = setTimeout(checkStatus, 2000)
            }
          }, res => {
            this.submitting = false
            clearTimeout(this.refreshStatus)
          })
        }
        this.refreshStatus = setTimeout(checkStatus, 2000)
      },
      submitCode () {
        if (this.code.trim() === '') {
          this.$error(this.$i18n.t('m.Code_can_not_be_empty'))
          return
        }
        const Base64 = require('js-base64').Base64
        this.submitting = true
        const data = {
          problemId: this.problem.id,
          language: this.language,
          code: Base64.encode(this.code)
        }
        const submitFunc = (data) => {
          this.statusVisible = true
          const funcName = this.$route.name === 'contest-problem-details' ? 'submitContestCode' : 'submitCode'
          api[funcName](data).then(res => {
            // 定时检查状态
            this.submitting = false
            this.submissionExists = true
            this.submitted = true
            this.checkSubmissionStatus()
          }, res => {
            this.submitting = false
            this.statusVisible = false
          })
        }
        // 由于后端提交的时候没有返回标识位, 因此要想判断当前代码执行的状态, 需要在提交之前获取该用户对于该题的最新提交状态, 通过定时器监听即可
        const funcName = this.$route.name === 'contest-problem-details' ? 'getContestSubmissionList' : 'getSubmissionList'
        if (this.$route.name === 'contest-problem-details') {
          data.contestId = this.contestID
          api.submitContestCode(data).then(res => {
            this.$router.push({ name: 'contest-submission-list' })
          })
          return
        }
        api[funcName](this.buildStatusQuery()).then(res => {
          const nowArr = res.data.data.records
          this.result.judge = nowArr && nowArr.length && nowArr[0] || null
          if (this.submissionExists) {
            this.$Modal.confirm({
              title: '',
              content: '<h3>' + this.$i18n.t('m.You_have_submission_in_this_problem_sure_to_cover_it') + '<h3>',
              onOk: () => {
                // 暂时解决对话框与后面提示对话框冲突的问题(否则一闪而过）
                setTimeout(() => {
                  submitFunc(data)
                }, 1000)
              },
              onCancel: () => {
                this.submitting = false
              }
            })
          } else {
            submitFunc(data)
          }
        })
      },
      onCopy (event) {
        this.$success('Code copied')
      },
      onCopyError (e) {
        this.$error('Failed to copy code')
      }
    }
  }
</script>

<style lang="less" scoped>
  /deep/.ivu-card {
    background: var(--table-card-top);
    color: var(--font-color-white);
  }
  /deep/.markdown-body pre {
    background: var(--problem-mark-bg-color)
  }
  /deep/ .CodeMirror-gutters {
    background: var(--problem-codenav-bg-color);
  }
  /deep/.CodeMirror {
    background: var(--problem-codebody-bg-color);
  }
  /deep/.ivu-modal-content {
    background: var(--problem-canvas-bg-color);
  }
  .card-title {
    margin-left: 8px;
  }

  .spin-container {
    position: relative;
  }
  .flex-container {
    #problem-main {
      flex: auto;
      margin-right: 18px;
    }
    #right-column {
      flex: none;
      width: 220px;
      li,
      .ivu-btn-ghost {
        color: var(--font-color-white);
      }
      .cursor-sub {
        cursor: pointer;
      }
      li:hover {
        background: var(--rightnav-bg-color);
      }
    }
  }
  #problem-content {
    margin-top: -50px;
    .title {
      font-size: 20px;
      font-weight: 400;
      margin: 25px 0 8px 0;
      color: #3091f2;
      .copy {
        padding-left: 8px;
      }
    }
    p.content {
      margin-left: 25px;
      margin-right: 20px;
      font-size: 15px
    }
    .sample {
      align-items: stretch;
      &-input, &-output {
        width: 50%;
        flex: 1 1 auto;
        display: flex;
        flex-direction: column;
        margin-right: 5%;
      }
      pre {
        flex: 1 1 auto;
        align-self: stretch;
        border-style: solid;
        background: transparent;
        white-space: pre-line;
      }
    }
  }

  #submit-code {
    margin-top: 20px;
    margin-bottom: 20px;
    .status {
      float: left;
      span {
        margin-right: 10px;
        margin-left: 10px;
      }
    }
    .captcha-container {
      display: inline-block;
      .captcha-code {
        width: auto;
        margin-top: -20px;
        margin-left: 20px;
      }
    }
  }

  #info {
    margin-bottom: 20px;
    margin-top: 20px;
    ul {
      list-style-type: none;
      li {
        border-bottom: 1px dotted #e9eaec;
        margin-bottom: 10px;
        p {
          display: inline-block;
        }
        p:first-child {
          width: 90px;
        }
        p:last-child {
          float: right;
        }
      }
    }
  }

  .fl-right {
    float: right;
  }

  #pieChart {
    .echarts {
      height: 250px;
      width: 210px;
    }
    #detail {
      position: absolute;
      right: 10px;
      top: 10px;
    }
  }

  #pieChart-detail {
    margin-top: 20px;
    width: 500px;
    height: 480px;
  }
</style>

