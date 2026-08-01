<script setup lang="ts">
import axios from 'axios'
import { computed, onBeforeUnmount, onMounted, ref } from 'vue'

type Page = 'home' | 'quiz' | 'result'
type DataSource = 'live' | 'demo'

interface ChapterStat {
  chapter: string
  count: number
}

interface Question {
  id: number
  chapter: string
  stem: string
  options: string[]
  answer?: string
  explanation?: string
}

interface AnswerResult {
  correct: boolean
  answer: string
  explanation: string
}

interface AnswerRecord extends AnswerResult {
  selected: string
}

const api = axios.create({
  baseURL: import.meta.env.VITE_API_BASE || 'http://localhost:8001',
  timeout: 4000,
})

const demoQuestions: Question[] = [
  {
    id: 10001,
    chapter: '股票种类与标识',
    stem: '下列关于普通股股东权利的表述，正确的是哪一项？',
    options: ['A. 可以优先于债权人获得清偿', 'B. 通常享有公司经营决策表决权', 'C. 无权参与公司利润分配', 'D. 收益率由公司事先固定'],
    answer: 'B',
    explanation: '普通股股东通常享有表决权、利润分配权等基本权利，但其清偿顺序在债权人之后，收益也不是固定的。',
  },
  {
    id: 10002,
    chapter: '股票种类与标识',
    stem: '上市公司股票名称前标注“*ST”，通常表示什么？',
    options: ['A. 公司刚刚上市', 'B. 股票暂停交易', 'C. 公司存在退市风险警示', 'D. 公司正在进行分红'],
    answer: 'C',
    explanation: '“*ST”用于提示上市公司存在退市风险，投资者需要特别关注相关公告和交易风险。',
  },
  {
    id: 10003,
    chapter: '股票交易规则',
    stem: '我国A股市场通常实行的股票交易交收制度是？',
    options: ['A. T+0', 'B. T+1', 'C. T+2', 'D. T+3'],
    answer: 'B',
    explanation: 'A股股票交易通常实行T+1制度，即当日买入的股票一般要到下一个交易日才能卖出。',
  },
  {
    id: 10004,
    chapter: '股票交易规则',
    stem: '集合竞价阶段的主要作用是什么？',
    options: ['A. 只处理大宗交易', 'B. 确定开盘价或收盘价', 'C. 取消全部未成交委托', 'D. 限制投资者报单'],
    answer: 'B',
    explanation: '集合竞价通过集中撮合申报来确定开盘价或收盘价，是证券交易价格形成的重要环节。',
  },
  {
    id: 10005,
    chapter: '基金类型',
    stem: 'ETF最典型的特点是哪一项？',
    options: ['A. 不能在交易所买卖', 'B. 只能投资债券', 'C. 可以像股票一样在交易所交易', 'D. 每年只能申购一次'],
    answer: 'C',
    explanation: 'ETF是交易型开放式指数基金，可以在交易所挂牌交易，也通常支持申购和赎回。',
  },
  {
    id: 10006,
    chapter: '基金类型',
    stem: '基金资产净值减去基金负债后得到的是？',
    options: ['A. 基金利润', 'B. 基金净资产', 'C. 基金申购费', 'D. 基金管理费'],
    answer: 'B',
    explanation: '基金资产总值减去基金负债即为基金净资产，再除以基金份额可得到基金份额净值。',
  },
  {
    id: 10007,
    chapter: '证券从业法规',
    stem: '证券从业人员知悉内幕信息后，正确的做法是？',
    options: ['A. 建议亲友提前买入', 'B. 在公开平台暗示走势', 'C. 严格保密并禁止利用信息交易', 'D. 先交易再上报'],
    answer: 'C',
    explanation: '内幕信息知情人应当严格履行保密义务，不得利用内幕信息交易，也不得泄露或建议他人交易。',
  },
  {
    id: 10008,
    chapter: '证券从业法规',
    stem: '投资者适当性管理的核心目的是什么？',
    options: ['A. 保证投资者一定获利', 'B. 将适当的产品销售给适当的投资者', 'C. 限制全部高风险产品交易', 'D. 替投资者决定买卖时点'],
    answer: 'B',
    explanation: '适当性管理强调了解客户、了解产品，并将风险等级适当的产品或服务提供给匹配的投资者。',
  },
]

const page = ref<Page>('home')
const dataSource = ref<DataSource>('demo')
const loading = ref(false)
const notice = ref('')
const chapters = ref<ChapterStat[]>([])
const quizQuestions = ref<Question[]>([])
const currentIndex = ref(0)
const selectedAnswer = ref('')
const answers = ref<Record<number, AnswerRecord>>({})
const practiceChapter = ref<string | null>(null)
const isRandomPractice = ref(false)

const currentQuestion = computed(() => quizQuestions.value[currentIndex.value])
const currentRecord = computed(() => currentQuestion.value ? answers.value[currentQuestion.value.id] : undefined)
const answeredCount = computed(() => Object.keys(answers.value).length)
const correctCount = computed(() => Object.values(answers.value).filter((item) => item.correct).length)
const wrongCount = computed(() => answeredCount.value - correctCount.value)
const accuracy = computed(() => answeredCount.value ? Math.round((correctCount.value / answeredCount.value) * 100) : 0)
const progress = computed(() => quizQuestions.value.length ? ((currentIndex.value + 1) / quizQuestions.value.length) * 100 : 0)
const modeTitle = computed(() => practiceChapter.value || (isRandomPractice.value ? '随机练习' : '综合练习'))

function demoChapters(): ChapterStat[] {
  const counts = new Map<string, number>()
  demoQuestions.forEach((question) => counts.set(question.chapter, (counts.get(question.chapter) || 0) + 1))
  return Array.from(counts, ([chapter, count]) => ({ chapter, count }))
}

function shuffle<T>(items: T[]): T[] {
  const result = [...items]
  for (let i = result.length - 1; i > 0; i -= 1) {
    const j = Math.floor(Math.random() * (i + 1))
    const current = result[i] as T
    result[i] = result[j] as T
    result[j] = current
  }
  return result
}

async function loadHomeData() {
  loading.value = true
  try {
    const { data } = await api.get<ChapterStat[]>('/chapters')
    if (!data.length) throw new Error('题库为空')
    chapters.value = data
    dataSource.value = 'live'
    notice.value = ''
  } catch {
    chapters.value = demoChapters()
    dataSource.value = 'demo'
    notice.value = '当前未连接题库服务，已自动启用演示题。启动后端后刷新页面即可使用正式题库。'
  } finally {
    loading.value = false
  }
}

async function startPractice(chapter: string | null = null, random = false) {
  loading.value = true
  practiceChapter.value = chapter
  isRandomPractice.value = random

  try {
    let questions: Question[] = []
    if (dataSource.value === 'live') {
      const { data } = await api.get<Question[]>('/questions', {
        params: { chapter: chapter || undefined, limit: 500 },
      })
      questions = data
    }

    if (!questions.length) {
      questions = chapter ? demoQuestions.filter((item) => item.chapter === chapter) : [...demoQuestions]
      if (!questions.length) questions = [...demoQuestions]
      dataSource.value = 'demo'
      notice.value = '正式题库暂时不可用，本次练习使用演示题。'
    }

    quizQuestions.value = random ? shuffle(questions) : questions
    currentIndex.value = 0
    selectedAnswer.value = ''
    answers.value = {}
    page.value = 'quiz'
    window.scrollTo({ top: 0, behavior: 'smooth' })
  } catch {
    quizQuestions.value = random ? shuffle(demoQuestions) : [...demoQuestions]
    dataSource.value = 'demo'
    notice.value = '加载题库失败，本次练习已切换为演示题。'
    currentIndex.value = 0
    selectedAnswer.value = ''
    answers.value = {}
    page.value = 'quiz'
  } finally {
    loading.value = false
  }
}

function optionKey(option: string, index: number): string {
  return option.match(/^([A-D])(?:[.、．:：]|\s)/i)?.[1]?.toUpperCase() || String.fromCharCode(65 + index)
}

function optionText(option: string): string {
  return option.replace(/^[A-D](?:[.、．:：]|\s)\s*/i, '')
}

function selectOption(answer: string) {
  if (currentRecord.value) return
  selectedAnswer.value = answer
}

async function submitAnswer() {
  const question = currentQuestion.value
  if (!question || !selectedAnswer.value || currentRecord.value) return

  let result: AnswerResult = {
    correct: selectedAnswer.value === question.answer,
    answer: question.answer || '',
    explanation: question.explanation || '暂无解析。',
  }

  if (dataSource.value === 'live') {
    try {
      const { data } = await api.post<AnswerResult>('/check', {
        question_id: question.id,
        user_answer: selectedAnswer.value,
      })
      result = data
    } catch {
      notice.value = '判分服务暂时不可用，已使用题目数据进行本地判分。'
    }
  }

  answers.value = {
    ...answers.value,
    [question.id]: { ...result, selected: selectedAnswer.value },
  }
}

function goToQuestion(index: number) {
  if (index < 0 || index >= quizQuestions.value.length) return
  const question = quizQuestions.value[index]
  if (!question) return
  currentIndex.value = index
  selectedAnswer.value = answers.value[question.id]?.selected || ''
  window.scrollTo({ top: 0, behavior: 'smooth' })
}

function nextQuestion() {
  if (currentIndex.value < quizQuestions.value.length - 1) {
    goToQuestion(currentIndex.value + 1)
  } else if (answeredCount.value > 0) {
    finishPractice()
  }
}

function finishPractice() {
  if (!answeredCount.value) return
  page.value = 'result'
  localStorage.setItem('securities-last-result', JSON.stringify({
    total: answeredCount.value,
    correct: correctCount.value,
    accuracy: accuracy.value,
    date: new Date().toISOString(),
  }))
  window.scrollTo({ top: 0, behavior: 'smooth' })
}

function backHome() {
  page.value = 'home'
  window.scrollTo({ top: 0, behavior: 'smooth' })
}

function optionState(answer: string): string {
  const record = currentRecord.value
  if (!record) return selectedAnswer.value === answer ? 'selected' : ''
  if (record.answer === answer) return 'correct'
  if (record.selected === answer && !record.correct) return 'wrong'
  return 'muted'
}

function handleKeyboard(event: KeyboardEvent) {
  if (page.value !== 'quiz' || !currentQuestion.value) return
  const keys = ['1', '2', '3', '4']
  const index = keys.indexOf(event.key)
  if (index >= 0 && index < currentQuestion.value.options.length) {
    const option = currentQuestion.value.options[index]
    if (option) selectOption(optionKey(option, index))
  }
  if (event.key === 'Enter') {
    if (!currentRecord.value) submitAnswer()
    else nextQuestion()
  }
}

onMounted(() => {
  loadHomeData()
  window.addEventListener('keydown', handleKeyboard)
})

onBeforeUnmount(() => window.removeEventListener('keydown', handleKeyboard))
</script>

<template>
  <div class="app-shell">
    <header class="site-header">
      <div class="header-inner">
        <button class="brand" type="button" aria-label="返回首页" @click="backHome">
          <span class="brand-mark">证</span>
          <span>
            <strong>证券从业资格考试题库</strong>
            <small>专业知识练习平台</small>
          </span>
        </button>
        <div class="header-status">
          <span class="status-dot" :class="dataSource"></span>
          {{ dataSource === 'live' ? '正式题库' : '演示题库' }}
        </div>
      </div>
    </header>

    <main v-if="page === 'home'" class="page home-page">
      <section class="hero">
        <div class="hero-copy">
          <span class="eyebrow">证券从业资格考试</span>
          <h1>系统练习，稳步掌握<br><em>每一个核心考点</em></h1>
          <p>按章节巩固知识，结合综合练习检验成果。每道题即时判分并提供清晰解析。</p>
          <div class="hero-actions">
            <button class="primary-button" type="button" :disabled="loading" @click="startPractice(null, false)">
              开始综合练习
            </button>
            <button class="secondary-button" type="button" :disabled="loading" @click="startPractice(null, true)">
              随机抽题练习
            </button>
          </div>
        </div>
        <div class="hero-panel" aria-label="题库概况">
          <span class="panel-label">题库概况</span>
          <strong>{{ chapters.reduce((sum, item) => sum + item.count, 0) }}</strong>
          <span>道可练习题目</span>
          <div class="panel-divider"></div>
          <div class="panel-meta">
            <div><b>{{ chapters.length }}</b><span>考试章节</span></div>
            <div><b>即时</b><span>答案解析</span></div>
          </div>
        </div>
      </section>

      <div v-if="notice" class="notice" role="status">
        <span class="notice-icon">i</span>
        <span>{{ notice }}</span>
      </div>

      <section class="section-block">
        <div class="section-heading">
          <div>
            <span class="section-kicker">章节专项练习</span>
            <h2>选择需要巩固的知识模块</h2>
          </div>
          <span class="section-hint">建议按顺序完成各章节练习</span>
        </div>

        <div v-if="loading" class="loading-card">正在读取题库，请稍候……</div>
        <div v-else class="chapter-grid">
          <button
            v-for="(chapter, index) in chapters"
            :key="chapter.chapter"
            class="chapter-card"
            type="button"
            @click="startPractice(chapter.chapter, false)"
          >
            <span class="chapter-number">{{ String(index + 1).padStart(2, '0') }}</span>
            <span class="chapter-content">
              <strong>{{ chapter.chapter }}</strong>
              <small>{{ chapter.count }}道练习题</small>
            </span>
            <span class="chapter-arrow">→</span>
          </button>
        </div>
      </section>

      <section class="study-guide">
        <div>
          <span class="section-kicker">备考建议</span>
          <h2>练习不是刷数量，而是及时弄懂每一道错题</h2>
        </div>
        <ol>
          <li><b>01</b><span><strong>先做章节练习</strong>集中掌握单个知识模块</span></li>
          <li><b>02</b><span><strong>认真阅读解析</strong>理解正确答案背后的依据</span></li>
          <li><b>03</b><span><strong>再做综合练习</strong>检验不同知识点的掌握程度</span></li>
        </ol>
      </section>
    </main>

    <main v-else-if="page === 'quiz' && currentQuestion" class="page quiz-page">
      <div class="quiz-topbar">
        <div>
          <button class="text-button" type="button" @click="backHome">← 返回题库</button>
          <span class="mode-badge">{{ modeTitle }}</span>
        </div>
        <button class="finish-button" type="button" :disabled="!answeredCount" @click="finishPractice">结束练习</button>
      </div>

      <section class="progress-card">
        <div class="progress-copy">
          <span>练习进度</span>
          <strong>{{ currentIndex + 1 }}<small>/{{ quizQuestions.length }}</small></strong>
        </div>
        <div class="progress-track" aria-label="练习进度">
          <span :style="{ width: `${progress}%` }"></span>
        </div>
        <div class="score-summary"><span>已答{{ answeredCount }}题</span><span>答对{{ correctCount }}题</span></div>
      </section>

      <div v-if="notice" class="notice compact" role="status">
        <span class="notice-icon">i</span><span>{{ notice }}</span>
      </div>

      <div class="quiz-layout">
        <section class="question-card">
          <div class="question-meta">
            <span class="question-type">单项选择题</span>
            <span>{{ currentQuestion.chapter }}</span>
          </div>
          <h2><span>{{ currentIndex + 1 }}.</span>{{ currentQuestion.stem }}</h2>

          <div class="options-list" role="radiogroup" aria-label="请选择答案">
            <button
              v-for="(option, index) in currentQuestion.options"
              :key="option"
              class="option-button"
              :class="optionState(optionKey(option, index))"
              type="button"
              :aria-pressed="selectedAnswer === optionKey(option, index)"
              @click="selectOption(optionKey(option, index))"
            >
              <span class="option-key">{{ optionKey(option, index) }}</span>
              <span class="option-text">{{ optionText(option) }}</span>
              <span v-if="currentRecord?.answer === optionKey(option, index)" class="option-result">✓</span>
              <span v-else-if="currentRecord?.selected === optionKey(option, index) && !currentRecord.correct" class="option-result">×</span>
            </button>
          </div>

          <div v-if="currentRecord" class="analysis-panel" :class="currentRecord.correct ? 'success' : 'error'">
            <div class="analysis-title">
              <strong>{{ currentRecord.correct ? '回答正确' : '回答错误' }}</strong>
              <span>正确答案：{{ currentRecord.answer }}</span>
            </div>
            <div class="analysis-body"><b>答案解析</b><p>{{ currentRecord.explanation }}</p></div>
          </div>

          <div class="question-actions">
            <button class="secondary-button small" type="button" :disabled="currentIndex === 0" @click="goToQuestion(currentIndex - 1)">上一题</button>
            <button v-if="!currentRecord" class="primary-button small" type="button" :disabled="!selectedAnswer" @click="submitAnswer">提交答案</button>
            <button v-else class="primary-button small" type="button" @click="nextQuestion">
              {{ currentIndex === quizQuestions.length - 1 ? '查看结果' : '下一题' }}
            </button>
          </div>
          <p v-if="!selectedAnswer && !currentRecord" class="action-tip">请选择一个答案后提交，可使用数字键1—4快速选择。</p>
        </section>

        <aside class="answer-sheet">
          <div class="sheet-heading"><strong>答题卡</strong><span>{{ answeredCount }}/{{ quizQuestions.length }}</span></div>
          <div class="sheet-grid">
            <button
              v-for="(question, index) in quizQuestions"
              :key="question.id"
              type="button"
              :class="{
                active: index === currentIndex,
                correct: answers[question.id]?.correct,
                wrong: answers[question.id]?.correct === false,
              }"
              @click="goToQuestion(index)"
            >{{ index + 1 }}</button>
          </div>
          <div class="sheet-legend">
            <span><i class="done"></i>正确</span><span><i class="error"></i>错误</span><span><i></i>未答</span>
          </div>
        </aside>
      </div>
    </main>

    <main v-else class="page result-page">
      <section class="result-card">
        <span class="result-kicker">本次练习已完成</span>
        <h1>{{ accuracy >= 80 ? '掌握得很好，请继续保持' : accuracy >= 60 ? '基础不错，还可以继续提升' : '建议回顾解析后再次练习' }}</h1>
        <div class="score-ring" :style="{ '--score': `${accuracy * 3.6}deg` }">
          <div><strong>{{ accuracy }}</strong><span>正确率</span></div>
        </div>
        <div class="result-stats">
          <div><span>完成题目</span><strong>{{ answeredCount }}</strong></div>
          <div><span>回答正确</span><strong class="green">{{ correctCount }}</strong></div>
          <div><span>回答错误</span><strong class="red">{{ wrongCount }}</strong></div>
        </div>
        <div class="result-actions">
          <button class="primary-button" type="button" @click="startPractice(practiceChapter, isRandomPractice)">重新练习</button>
          <button class="secondary-button" type="button" @click="backHome">返回题库首页</button>
        </div>
      </section>

      <section v-if="wrongCount" class="wrong-review">
        <div class="section-heading compact-heading">
          <div><span class="section-kicker">错题回顾</span><h2>本次需要巩固的题目</h2></div>
        </div>
        <button
          v-for="(question, index) in quizQuestions.filter((item) => answers[item.id]?.correct === false)"
          :key="question.id"
          class="wrong-item"
          type="button"
          @click="page = 'quiz'; goToQuestion(quizQuestions.indexOf(question))"
        >
          <span>{{ index + 1 }}</span><strong>{{ question.stem }}</strong><i>查看解析 →</i>
        </button>
      </section>
    </main>

    <footer class="site-footer">
      <span>证券从业资格考试题库</span>
      <span>科学练习 · 及时反馈 · 稳步提升</span>
    </footer>

    <div v-if="loading && page !== 'home'" class="loading-overlay" role="status">
      <span class="loader"></span><p>正在准备练习……</p>
    </div>
  </div>
</template>
