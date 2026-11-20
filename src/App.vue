<script setup>
import { ref, reactive, computed, watch, onMounted, onBeforeUnmount } from 'vue'
import Rater from './components/Rater.vue'

const TITLE = 'Video Study'
const email = ref('')
const round = ref(null)
const plan = ref([])
const current = computed(() => (round.value ? plan.value[round.value - 1] : null))
const nPractice = computed(() => plan.value.filter((p) => p.practice && p.exp === 1).length)
const nPractice2 = computed(() => plan.value.filter((p) => p.practice && p.exp === 2).length)
const nRated = computed(() => plan.value.filter((p) => p.ridx).length)
const nPart2 = computed(() => plan.value.filter((p) => p.ridx && p.exp === 2).length)
const ratedIndex = computed(() => (current.value && current.value.ridx) || 0)
const ratedDone = computed(() => {
  let n = 0
  for (let i = 0; i < round.value - 1; i++) if (plan.value[i].ridx) n++
  return n
})
function next() {
  round.value += 1
  window.scrollTo(0, 0)
}
const session = Math.random().toString(36).slice(2, 10) + Date.now().toString(36)
const N1 = Number(import.meta.env.VITE_EXP1_PAIRS || 16)
const N2 = Number(import.meta.env.VITE_EXP2_PAIRS || 6)

function shuffle(a) {
  for (let i = a.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [a[i], a[j]] = [a[j], a[i]]
  }
  return a
}

// One session: N1 comparison pairs on distinct scenes, balanced over the four conditions, two attention checks
// inserted at random positions, then N2 problem pairs on distinct scenes. Left/right order is random per pair.
function makePlan(pairs) {
  const e1 = pairs.filter((p) => p.exp === 1)
  const e2 = pairs.filter((p) => p.exp === 2)
  const ck = pairs.filter((p) => p.exp === 0)
  const groups = shuffle([...new Set(e1.map((p) => p.g))]).slice(0, N1)
  const conds = shuffle([...new Set(e1.map((p) => p.c))])
  const part1 = groups.map((g, i) => e1.find((p) => p.g === g && p.c === conds[i % conds.length]))
  shuffle(part1)
  for (const c of shuffle([...ck])) part1.splice(1 + Math.floor(Math.random() * (part1.length - 1)), 0, c)
  const seen = new Set()
  const part2 = []
  for (const p of shuffle([...e2])) {
    if (part2.length >= N2) break
    if (seen.has(p.g)) continue
    seen.add(p.g)
    part2.push(p)
  }
  return [...part1, ...part2].map((p) => {
    const swap = Math.random() < 0.5
    return { ...p, left: swap ? p.v[1] : p.v[0], right: swap ? p.v[0] : p.v[1] }
  })
}

const checked = ref(false)
const showWarning = ref(false)
const loadError = ref(false)
let manifest = null
onMounted(async () => {
  try {
    const r = await fetch(`${import.meta.env.BASE_URL}manifest.json`)
    manifest = await r.json()
  } catch (e) {
    loadError.value = true
  }
})
function start() {
  if (checked.value === false) {
    showWarning.value = true
    return
  }
  if (!manifest) return
  showWarning.value = false
  // Practice items: exp 1 before Part 1, exp 2 after the Part 2 intro. Numbered within their part.
  const prac = (e) => {
    const items = (manifest.practice || []).filter((p) => (p.exp || 1) === e)
    return items.map((p, i) => ({ ...p, exp: e, practice: true, pidx: i + 1, pn: items.length }))
  }
  const practice = prac(1)
  const practice2 = prac(2)
  const rated = makePlan(manifest.pairs).map((p, i) => ({ ...p, ridx: i + 1 }))
  const firstPart2 = rated.findIndex((p) => p.exp === 2)
  const steps = []
  if (practice.length) steps.push({ intro: 'practice' }, ...practice)
  steps.push({ intro: 'main' })
  rated.forEach((p, i) => {
    if (i === firstPart2) steps.push({ intro: 'part2' }, ...practice2, ...(practice2.length ? [{ intro: 'part2go' }] : []))
    steps.push(p)
  })
  // Preview builds only (VITE_PREVIEW_START=part2): start at the Part 2 intro to check that part quickly.
  if (import.meta.env.VITE_PREVIEW_START === 'part2') steps.splice(0, steps.findIndex((x) => x.intro === 'part2'))
  plan.value = steps
  round.value = 1
  document.getElementById('app').classList.add('trial')
}

const thankYou = reactive({
  show: false,
  countdown: 10
})

function submit(answer) {
  const item = current.value
  if (item.practice) {
    round.value += 1
    window.scrollTo(0, 0)
    return
  }
  const keys = JSON.parse(import.meta.env.VITE_GOOGLE_FORM_ENTRIES)
  const values = [email.value, session, item.ridx, item.id, item.left, item.right,
    answer.a1, answer.a2, answer.seconds, lang.value]
  // Same submission path as the lab template (prefilled Google Form response, fire and forget), with encoded values.
  const query = keys.map((k, i) => `${k}=${encodeURIComponent(values[i])}`).join('&')
  fetch(`${import.meta.env.VITE_GOOGLE_FORM_URL}?${query}`, { mode: 'no-cors' })

  round.value += 1
  if (round.value > plan.value.length) {
    thankYou.show = true
    setInterval(() => thankYou.countdown--, 1000)
  }
  window.scrollTo(0, 0)
}
watch(thankYou, (newThankYou, _) => {
  if (newThankYou.countdown === 0) {
    location.reload()
  }
})

const lang = ref('en-US')
onMounted(() => {
  lang.value = navigator.language || navigator.userLanguage
})

// sync width of the header and main content on mobile device
function onResize() {
  const appDOM = document.getElementById('app')
  const headerDiv = document.getElementById('header-div')
  if (headerDiv !== null) {
    if (window.matchMedia('(max-width: 1023px)').matches) {
      headerDiv.style.width = appDOM.offsetWidth + 'px'
    } else {
      headerDiv.style.width = null
    }
  }
}

const resizeObserver = new ResizeObserver(onResize)
onMounted(() => {
  resizeObserver.observe(document.getElementById('app'))
})
onBeforeUnmount(() => {
  resizeObserver.unobserve(document.getElementById('app'))
})

</script>

<template>
  <div v-if="thankYou.show">
    <template v-if="lang === 'zh-TW'">
      感謝您的填寫，我們已收到您的意見。 本頁面將在 {{ thankYou.countdown }} 秒後重新載入，
      您可以再次使用同一個 email 填寫。
    </template>
    <template v-else>
      We have received your submission. Thank you! This page will reload in
      {{ thankYou.countdown }} seconds. You can enter the same email for a new submission.
    </template>
  </div>
  <template v-else>
    <div class="main" :class="{ home: round === null }">
      <aside v-if="round === null" class="info">
      <div class="scrollable">
        <template v-if="lang === 'zh-TW'">
          <h3>研究參與資訊</h3>
          <ul>
            <li>資料收集：我們計劃收集100-200名受試者的生成影像或影片評分資料。若有人中途退出或數據無法使用，將遞補至收集完所需受試者為止。</li>
            <li>遞補機制：若有人中途退出或數據無法使用，將遞補至收集完所需受試者為止。</li>
            <li>目標對象：已滿 18 歲且未受監護宣告之成年人。</li>
            <li>經費來源：中研院資創中心智慧優網。</li>
          </ul>
          <br>
          <h3>資料管理與隱私保障</h3>
          <ul>
            <li>資料用途：本問卷內容僅供學術研究之用。</li>
            <li>聯絡資訊：您的 email 等聯絡資訊僅供 7-11 禮券抽獎使用，不會留下任何身份等敏感資訊。</li>
            <li>資料刪除：所有聯絡資料將於問卷調查與抽獎完畢後立即刪除。</li>
            <li>退出與刪除權利：您可在填寫問卷時隨時退出，填寫完成後也可來信 (<a href="nickchiu@citi.sinica.edu.tw">nickchiu@citi.sinica.edu.tw</a> 或 <a href="pullpull@citi.sinica.edu.tw">pullpull@citi.sinica.edu.tw</a>) 要求刪除相關資料。</li>
          </ul>
          <br>
          <h3>資料管理細節</h3>
          <ul>
            <li>不收集敏感資料：本計畫實驗不收集任何關於您的個人敏感資料，僅收集生成影像或影片的評分。</li>
            <li>資料儲存與加密：實驗收集的資料將儲存於實驗地點（中央研究院資訊創新研究中心）的資料處理電腦，並會進一步加密處理。除了評分結果與抽獎用 email，研究人員分析資料時不會取得您的身分資料。</li>
            <li>資料使用限制：所收集的資料僅供本實驗室人員學術研究使用。</li>
            <li>抽獎後刪除：email 聯絡資訊將於抽獎完成後（問卷結束後一至兩週內進行）立即刪除。</li>
            <li>資料庫管理：此計畫所收集的資料庫（生成影像或影片評分結果）為不記名資料庫，放置在主持人實驗室伺服器，資料保存和使用時間至計畫結束（2026/12/31）。若被挪作其他用途，本計畫團隊將追溯非法使用來源，以確保您的權益。計畫結束後，實驗室將銷毀相關問卷評量結果。</li>
          </ul>
          <br>
          <h3>參與權益聲明</h3>
          <ul>
            <li>研究計畫名稱：AI 生成影像與影片之使用者偏好評量</li>
            <li>編號：ASIRB-HS-24031</li>
            <li>自願參與：參加此項計畫是完全自願性質，無論參加與否均不影響您的權益。</li>
            <li>權益諮詢：若對參與研究的相關權益有疑問，可來電查詢中央研究院人文社會科學研究倫理委員會 IRB on Humanities & Social Science Research / IRB-HS，查詢電話：02-27898722。</li>
          </ul>
          <br>
          <h3>實驗進行方式</h3>
          您將對預先使用不同生成模型的生成影像或影片進行個人主觀的品質評分與排序，並評估其是否與給定的條件相符（如文字描述等條件）。您將進行 18 組的實驗（影片兩兩為一組），大約花費 10 到 15 分鐘，但您可隨時依自己的狀態停止實驗。
        </template>
        <template v-else>
          
          <h3>Research Participation Information</h3>
          <ul>
            <li>Data Collection: We will collect image or video evaluation data from 100-200 participants. If any participant withdraws or their data becomes unusable, we will recruit additional participants until the required number is reached.</li>
            <li>Target Subject: Adults aged 18 and above who are not under guardianship.</li>
            <li>Funding Source: Intelligent Networks Program at the Research Center for Information Technology Innovation, Academia Sinica</li>
          </ul>
          <br>
          <h3>Data Management and Privacy Protection</h3>
          <ul>
            <li>Data Usage: The content of this questionnaire is solely for academic research purposes.</li>
            <li>Contact Information: Your email and other contact information will only be used for a 7-11 gift voucher lottery and will not be used to record any personal identification or sensitive information.</li>
            <li>Data Deletion: All contact information will be deleted immediately after the questionnaire survey and lottery are completed.</li>
            <li>Withdrawal and Deletion Rights: You can withdraw from the questionnaire at any time during the process. After completion, you can also request the deletion of relevant data by sending an email to <a href="nickchiu@citi.sinica.edu.tw">nickchiu@citi.sinica.edu.tw</a> or <a href="pullpull@citi.sinica.edu.tw">pullpull@citi.sinica.edu.tw</a>.</li>
          </ul>
          <br>
          <h3>Details of Data Management</h3>
          <ul>
            <li>No Collection of Sensitive Data: This project does not collect any of your personal sensitive data, only evaluations of the generated images or videos.</li>
            <li>Data Storage and Encryption: Collected data will be stored on the data processing computers at the research site (Academia Sinica Research Center for Information Technology Innovation) and will be further encrypted. Researchers analyzing the data will not have access to your identity information, except for the evaluation results and the email used for the lottery.</li>
            <li>Data Usage Restrictions: The collected data will only be used for academic research by the personnel of this laboratory.</li>
            <li>Deletion After Lottery: Email contact information will be deleted immediately after the lottery is completed (within one to two weeks after the questionnaire ends).</li>
            <li>Database Management: The database collected in this project (evaluation results of generated images or videos) is anonymous and will be stored on the principal investigator's laboratory server. Data storage and usage will continue until the end of the project (2026/12/31). If the data is used for any other purpose, our team will track the source of unauthorized use to protect your rights. After the project concludes, the laboratory will destroy the relevant questionnaire evaluation results.</li>
          </ul>
          <br>
          <h3>Statement of Participation Rights</h3>
          <ul>
            <li>Voluntary Participation: Participation in this project is entirely voluntary, and your decision to participate or not will not affect your rights in any way.</li>
            <li>ID: ASIRB-HS-24031</li>
            <li>Voluntary Participation: Participation in this project is completely voluntary, and your rights will not be affected whether you choose to participate or not.</li>
            <li>Inquiry of Rights: If you have any questions about your rights related to research participation, please call the Academia Sinica Institutional Review Board on Humanities & Social Science Research (IRB-HS) at 02-27898722.</li>
          </ul>
          <br>
          <h3>Experimental Procedure</h3>
          You will be asked to provide your subjective evaluation and ranking of the quality of pre-generated images or videos using different generation models, and to assess whether they meet the given criteria (e.g., text descriptions). You will participate in 18 sets of experiments (the videos are grouped in pairs), which will take approximately 10 to 15 minutes. You can stop the experiment at any time based on your own condition.
        </template>
      </div>
      </aside>
      <div v-if="round === null" class="home-main">
        <div class="lang-selector">
          <span :class="{ green: lang === 'zh-TW' }" @click="lang = 'zh-TW'">中文</span> /
          <span :class="{ green: lang !== 'zh-TW' }" @click="lang = 'en-US'">Eng</span>
        </div>
        <h1 class="green">{{ TITLE }}</h1>
        <p>
          <template v-if="lang === 'zh-TW'">
            <!-- 感謝您參加本次測驗，本測驗共30題，我們將抽出五位完成作答的幸運兒，贈與超商禮券100元。
            您可以提供 email 讓我們在抽到您時通知您。如中途離開，本測驗不保留您未完成的作答。 -->

            大家好，我們是中央研究院資訊創新研究中心陳駿丞副研究員的研究團隊。
            <br><br>
            我們正在進行 AI 生成影像與影片之使用者偏好評量相關研究。本研究旨在評估現有影像與影片生成模型的生成品質，通過主觀和客觀的評估方法，分析模型生成影像的品質，進一步了解並分析生成模型的特性。
            <br><br>
            本研究將透過此網站進行問卷調查，提供您由模型預先生成的影像或影片，並請您針對其品質及是否符合給定的生成條件進行評分。
            <br><br>
            有效填寫問卷者可選擇留下email參加抽獎，我們將抽出有效問卷的 1/20 贏取7-11百元禮券。
          </template>
          <template v-else>
            <!-- Thanks for your interest in participating this test. The test contains 30 questions.
            We have $100 NTD coupon giveaway to 5 random selected participates who complete the test.
            You're welcome to fill in your email address so we can reach you if you won the lottery.
            If you leave the test half-way, this site will not keep your record. -->

            Hello everyone, we are the research team of Associate Research Fellow Jun-Cheng Chen from the Research Center for Information Technology Innovation at Academia Sinica.
            <br><br>
            We are currently conducting research on user preference evaluation of AI-generated images and videos. The aim of this study is to assess the quality of existing image and video generation models. 
            <br><br>
            Through subjective and objective evaluation methods, we will analyze the quality of the images produced by these models to further understand and analyze their characteristics.
            <br><br>
            Participants who complete the survey accurately can choose to provide their email address for a chance to enter a raffle. We will randomly select 1 out of every 20 valid survey submissions to win a 100 NT$ 7-11 gift voucher.
          </template>
        </p>

        <div class="consent">
          <template v-if="lang === 'zh-TW'">
            <a href="https://drive.google.com/file/d/1B3iZ2qKKpcfQexR-aIo1ZNPC1SNIT4Mm/view" target="_blank">中央研究院人文社會科學研究對象說明同意書</a>
          </template>
          <template v-else>
            <a href="https://drive.google.com/file/d/1B3iZ2qKKpcfQexR-aIo1ZNPC1SNIT4Mm/view" target="_blank">Academia Sinica Humanities and Social Sciences Research Subject Description and Consent Form</a>
          </template>
            <br>
          <input type="checkbox" id="checkbox" v-model="checked" />
          <template v-if="lang === 'zh-TW'"> 我已閱讀並同意 </template>
          <template v-else> I have read and agree </template>
        </div>

        <label for="email" class="green">
          <br>
          <template v-if="lang === 'zh-TW'">
            <!-- 請輸入您的 email 來開始作答 -->
            抽獎用信箱（選填，僅為抽獎通知用）
          </template>
          <template v-else>
            <!-- Please enter your email to get started. -->
            E-mail (optional, only for the lottery notification)
          </template>
        </label>

        <div>
          <input type="email" id="email" v-model="email" v-on:keyup.enter="start" />
          <button class="primary" @click="start">
            <template v-if="lang === 'zh-TW'"> 下一步 </template>
            <template v-else> Next </template>
          </button>
        </div>
        <p class="green" for="" v-if="showWarning">
          <template v-if="lang === 'en-US'"> Please confirm that you've read the consent form and agree </template>
          <template v-else> 請確認已閱讀以上同意書並同意 </template>
        </p>
      </div>
      <div v-else>
        <header>
          <div id="header-div">
            <div class="lang-selector">
              <span :class="{ green: lang === 'zh-TW' }" @click="lang = 'zh-TW'">中文</span> /
              <span :class="{ green: lang !== 'zh-TW' }" @click="lang = 'en-US'">Eng</span>
            </div>
            <h1 class="green">{{ TITLE }}</h1>
            <h3 v-if="current && current.intro"></h3>
            <h3 v-else-if="current && current.practice">
              <template v-if="lang === 'zh-TW'"> 例題 </template>
              <template v-else> Practice </template>
              {{ current.pidx }}/{{ current.pn }}
              <span class="practice-note">{{ lang === 'zh-TW' ? '不計分，答完會顯示正確答案' : 'not recorded, the answer is shown afterwards' }}</span>
            </h3>
            <h3 v-else>
              <template v-if="lang === 'zh-TW'"> 問題 </template>
              <template v-else> Question </template>
              {{ ratedIndex }}/{{ nRated }}
            </h3>
            <div class="progress"><div :style="{ width: (100 * ratedDone) / nRated + '%' }"></div></div>
          </div>
        </header>

        <p class="instruction" v-if="current && !current.intro && current.exp !== 2">
          <template v-if="lang === 'zh-TW'">畫面中途會被遮住，之後重新出現。請比較兩支影片<b class="green">揭開後</b>的樣子。</template>
          <template v-else>The view is blocked for a while and then revealed. Compare the two videos <b class="green">after the reveal</b>.</template>
        </p>
        <p class="instruction" v-else-if="current && !current.intro">
          <template v-if="lang === 'zh-TW'">請比較兩支影片中，<b class="prob">下方這個問題</b>的嚴重程度。</template>
          <template v-else>Compare how strongly each video shows <b class="prob">the problem below</b>.</template>
        </p>
      </div>
    </div>
    <div v-if="round !== null">
      <div v-if="current.intro" class="intro-page">
        <template v-if="current.intro === 'practice'">
          <h2>{{ lang === 'zh-TW' ? '作答方式' : 'How to answer' }}</h2>
          <template v-if="lang === 'zh-TW'">
            <p>每一題會有兩支影片（A 和 B）。影片中途畫面會被遮住一段時間，之後重新出現。請看完兩支影片，回答兩個問題：</p>
            <ol class="parts">
              <li>揭開後，哪支影片的「物體和場景」<b>保持得比較好</b>？</li>
              <li>比較「遮住前」和「揭開後」，哪支影片的<b>狀態變化比較合理</b>？</li>
            </ol>
            <p>兩支差不多時可以選「差不多」。下面先做 {{ nPractice }} 題例題，<b>不計分</b>，答完會顯示正確答案和說明。</p>
          </template>
          <template v-else>
            <p>Each question shows two videos (A and B). Partway through, the view is blocked for a while and then revealed again. Watch both videos and answer two questions:</p>
            <ol class="parts">
              <li>After the reveal, in which video are “the objects and the scene” <b>better preserved</b>?</li>
              <li>Comparing “before the occlusion” and “after the reveal”, which video shows a more <b>plausible change of state</b>?</li>
            </ol>
            <p>If the two look the same, choose "About the same". First, a practice question. It is <b>not recorded</b>, and the correct answer and explanation are shown afterwards.</p>
          </template>
        </template>
        <template v-else-if="current.intro === 'main'">
          <h2 v-if="nPractice">{{ lang === 'zh-TW' ? '例題結束，接下來是正式作答' : 'Practice done. The study starts now' }}</h2>
          <h2 v-else>{{ lang === 'zh-TW' ? '作答說明' : 'How the study works' }}</h2>
          <p v-if="lang === 'zh-TW'">正式作答共 <b>{{ nRated }} 題</b>，約 10 到 15 分鐘，分成兩種題型：</p>
          <p v-else>There are <b>{{ nRated }} questions</b> in two parts, about 10 to 15 minutes:</p>
          <ol class="parts">
            <li :class="{ now: true }">
              <template v-if="lang === 'zh-TW'"><b>第一部分（{{ nRated - nPart2 }} 題）</b>：{{ nPractice ? '和例題一樣的兩個問題。' : '比較兩支影片揭開後的物體和場景，以及遮住前後的狀態變化是否合理。' }}</template>
              <template v-else><b>Part 1 ({{ nRated - nPart2 }} questions)</b>: {{ nPractice ? 'the same two questions as in the practice.' : 'compare the objects and the scene after the reveal, and whether the change of state from before the occlusion is plausible.' }}</template>
            </li>
            <li>
              <template v-if="lang === 'zh-TW'"><b>第二部分（{{ nPart2 }} 題）</b>：比較影片中某個問題的嚴重程度，開始前會再說明。</template>
              <template v-else><b>Part 2 ({{ nPart2 }} questions)</b>: compare how strongly each video shows a given problem. It is explained before it starts.</template>
            </li>
          </ol>
          <p v-if="lang === 'zh-TW'">{{ nPractice ? '正式作答' : '作答時' }}<b>不會顯示答案</b>，請依您自己的判斷作答。兩支看起來差不多時，可以選「差不多」。現在開始第一部分。</p>
          <p v-else>The answers are <b>not shown</b>, so please answer with your own judgment. If the two look the same, choose "About the same". Part 1 starts now.</p>
        </template>
        <template v-else-if="current.intro === 'part2go'">
          <h2>{{ lang === 'zh-TW' ? '例題結束，開始第二部分正式作答' : 'Practice done. Part 2 starts now' }}</h2>
          <p v-if="lang === 'zh-TW'">接下來 <b>{{ nPart2 }} 題</b>不會顯示答案，請依您自己的判斷作答。</p>
          <p v-else>The next <b>{{ nPart2 }} questions</b> do not show the answers, so please answer with your own judgment.</p>
        </template>
        <template v-else>
          <h2>{{ lang === 'zh-TW' ? '第二部分（' + nPart2 + ' 題）：新的題型' : 'Part 2 (' + nPart2 + ' questions): a new type' }}</h2>
          <template v-if="lang === 'zh-TW'">
            <p>第一部分結束。接下來每一題的影片上方會列出<b class="prob">一個問題</b>，下面一行寫出<b class="ok">正確應該是</b>什麼樣子。請看完兩支影片，回答兩個問題：</p>
            <ol class="parts">
              <li>哪支影片的<b class="prob">這個問題</b>比較輕微？</li>
              <li>比較「遮住前」和「揭開後」，哪支影片的<b>狀態變化比較合理</b>？</li>
            </ol>
            <p>兩支差不多時可以選「差不多」。{{ nPractice2 ? '下面先做 ' + nPractice2 + ' 題例題，不計分，答完會顯示正確答案和說明。' : '' }}</p>
          </template>
          <template v-else>
            <p>Part 1 is done. From now on, <b class="prob">a problem</b> is described above the videos, and the line below it says what <b class="ok">should</b> happen. Watch both videos and answer two questions:</p>
            <ol class="parts">
              <li>In which video is <b class="prob">this problem</b> less severe?</li>
              <li>Comparing “before the occlusion” and “after the reveal”, which video shows a more <b>plausible change of state</b>?</li>
            </ol>
            <p>If the two look the same, choose "About the same".{{ nPractice2 ? ' First, a practice question. It is not recorded, and the correct answer and explanation are shown afterwards.' : '' }}</p>
          </template>
        </template>
        <button class="primary" @click="next">
          {{ current.intro === 'practice' ? (lang === 'zh-TW' ? '開始例題' : 'Start practice')
             : current.intro === 'main' ? (lang === 'zh-TW' ? '開始第一部分' : 'Start part 1')
             : current.intro === 'part2' && nPractice2 ? (lang === 'zh-TW' ? '開始例題' : 'Start practice') : (lang === 'zh-TW' ? '開始第二部分' : 'Start part 2') }}
        </button>
      </div>
      <Rater v-else :key="round" :item="current" :lang="lang" :last-practice="!!current.practice && current.pidx === current.pn" @submit="submit" />
    </div>
  </template>

</template>

<style scoped>
.lang-selector {
  float: right;
  font-size: 0.9rem;
  border: 1px solid var(--rule);
  border-radius: 999px;
  padding: 2px 12px;
  color: var(--muted);
  background: var(--card);
}

.lang-selector .green {
  color: var(--clay);
  font-weight: 600;
}

.lang-selector span:hover {
  cursor: pointer;
}

h1 {
  font-size: 2.2rem;
  font-weight: 600;
  margin-bottom: 0.4rem;
  letter-spacing: 0.01em;
}

h1.green {
  color: var(--ink);
}

h3 {
  font-size: 1.05rem;
  color: var(--muted);
  font-weight: 400;
}

input[type='email'] {
  border: 1px solid var(--rule);
  background: var(--card);
  color: var(--ink);
  height: 38px;
  border-radius: 6px;
  margin: 6px 8px 0 0;
  padding: 0 10px;
  font-size: 1rem;
  width: min(360px, 100%);
}

input[type='checkbox'] {
  accent-color: var(--clay-strong);
  width: 18px;
  height: 18px;
  vertical-align: -3px;
  margin-right: 6px;
}

button {
  height: 38px;
  padding: 0 20px;
  font-size: 1rem;
  border-radius: 6px;
  cursor: pointer;
}

p {
  margin: 10px 0 10px 0;
}

.scrollable {
  margin: 1.4rem 0;
  max-height: 380px;
  overflow-y: auto;
  background: var(--card);
  border: 1px solid var(--rule);
  border-radius: 10px;
  padding: 1rem 1.2rem;
  font-size: 0.92rem;
  line-height: 1.65;
}

.scrollable h3 {
  color: var(--ink);
  font-weight: 600;
  margin-top: 0.4rem;
}

.scrollable ul {
  padding-left: 1.2rem;
}

.consent {
  margin-top: 1.2rem;
}

/* Home page: the main content on the left, the research information (required but secondary) on the right. */
.main.home {
  display: flex;
  flex-direction: column;
}

.main.home .home-main {
  order: 0;
}

.main.home .info {
  order: 1;
}

@media (min-width: 1024px) {
  .main.home {
    display: grid;
    grid-template-columns: minmax(0, 1fr) minmax(0, 1fr);
    column-gap: 5rem;
    align-items: stretch;
    /* center the two columns in the window */
    min-height: calc(100vh - 4rem);
    align-content: center;
    max-width: 1100px;
    margin: 0 auto;
  }

  .main.home .home-main {
    grid-column: 1;
    grid-row: 1;
  }

  .main.home .info {
    grid-column: 2;
    grid-row: 1;
  }

  /* the box takes exactly the height of the left column and scrolls inside */
  .main.home .info {
    position: relative;
    min-height: 360px;
  }

  .main.home .info .scrollable {
    position: absolute;
    inset: 0;
    margin: 0;
    max-height: none;
  }
}

.intro-page {
  max-width: 640px;
  margin: 3rem auto;
  padding: 2rem 2.2rem;
  background: var(--card);
  border: 1px solid var(--rule);
  border-radius: 12px;
  line-height: 1.8;
}

.intro-page h2 {
  font-size: 1.5rem;
  margin-bottom: 0.6rem;
}

.intro-page b {
  color: var(--clay);
}

.parts {
  padding-left: 1.3rem;
  margin: 0.4rem 0 0.8rem;
}

.parts li {
  margin-bottom: 0.3rem;
}

.intro-page button {
  margin-top: 1rem;
}

.ok {
  color: var(--ok);
}

.practice-note {
  font-family: var(--sans);
  font-size: 0.8rem;
  color: var(--muted);
  margin-left: 8px;
}

.progress {
  height: 6px;
  background: var(--rule);
  border-radius: 3px;
  overflow: hidden;
  margin: 4px 0 10px;
}

.progress > div {
  height: 100%;
  background: var(--clay-strong);
  transition: width 0.3s;
}

.prob {
  background: var(--clay-soft);
  color: var(--clay);
  border-radius: 4px;
  padding: 0 4px;
}

p.instruction {
  font-size: 1.05rem;
}

@media (max-width: 1023px) {
  header {
    position: sticky;
    top: 0;
    background: var(--color-background);
    z-index: 1;
  }

  p.instruction {
    margin-top: 0;
  }
}
</style>
