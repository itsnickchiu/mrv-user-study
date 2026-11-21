<script setup>
import { ref, computed, onMounted } from 'vue'
const props = defineProps({
  item: { required: true },
  lang: { required: true },
  lastPractice: { type: Boolean, default: false }
})
const emit = defineEmits(['submit'])

const base = import.meta.env.BASE_URL
const src = (id) => `${base}videos/${id}.mp4`
const zh = computed(() => props.lang === 'zh-TW')
const problemPair = computed(() => props.item.exp === 2)

// Answers are recorded as L (left video), R (right video) or S (about the same).
const a1 = ref(null)
const a2 = ref(null)
const a3 = ref(null)
const ended = ref({ left: false, right: false })
const watched = computed(() => ended.value.left && ended.value.right)
const showWarning = ref(false)
const started = Date.now()
const leftEl = ref(null)
const rightEl = ref(null)

// Both videos autoplay together (muted, inline) and loop in sync: when both have reached the end, both restart from the
// first frame, so a shorter video waits on its last frame. `ended` records that each video was watched once in full.
const roundDone = { left: false, right: false }
const autoplayBlocked = ref(false)
function playBoth() {
  roundDone.left = roundDone.right = false
  for (const el of [leftEl.value, rightEl.value]) {
    if (!el) continue
    el.currentTime = 0
    const p = el.play()
    if (p && p.catch) p.catch(() => { autoplayBlocked.value = true })
  }
}
function onEnded(side) {
  ended.value[side] = true
  roundDone[side] = true
  if (roundDone.left && roundDone.right) playBoth()
}
onMounted(() => playBoth())

// Key words are highlighted (<em>) so raters see what each question asks; static strings only.
const Q_SCENE = { zh: '揭開後，哪支影片的「物體和場景」<em>保持得比較好</em>？', zhHint: '沒有被換掉、消失、變形或突然跳位',
  en: 'After the reveal, in which video are “the objects and the scene” <em>better preserved</em>?', enHint: 'nothing replaced, missing, deformed or jumping' }
const Q_PROCESS = { zh: '比較「遮住前」和「揭開後」，哪支影片的<em>狀態有正確往前演進</em>？', zhHint: '例如該融化的繼續融化、該掉下的已經落下，沒有停住或倒退',
  en: 'Comparing “before the occlusion” and “after the reveal”, in which video has <em>the state progressed correctly</em>?', enHint: 'e.g. what should melt keeps melting, what should fall has fallen, nothing freezes or goes backwards' }
const Q_PHYS = { zh: '哪支影片的動作和變化<em>比較符合物理常識</em>？', zhHint: '例如東西不會憑空出現或消失、不會穿過其他物體、液體往下流',
  en: 'Which video’s motion and changes are <em>more physically plausible</em>?', enHint: 'e.g. nothing appears or vanishes from nowhere, nothing passes through other objects, liquids flow down' }
const Q_PROBLEM = { zh: '哪支影片的<em class="prob">這個問題</em>比較輕微？', en: 'In which video is <em class="prob">this problem</em> less severe?' }
const questions = computed(() => (problemPair.value
  ? [{ key: 'a1', ...Q_PROBLEM, same: true }, { key: 'a2', ...Q_PROCESS, same: true }, { key: 'a3', ...Q_PHYS, same: true }]
  : [{ key: 'a1', ...Q_SCENE, same: true }, { key: 'a2', ...Q_PROCESS, same: true }, { key: 'a3', ...Q_PHYS, same: true }]))
const answers = { a1, a2, a3 }

// Practice items: the first click reveals the correct answers and the explanation, the second moves on.
const feedback = ref(false)
const correct = computed(() => props.item.practice && a1.value === props.item.answers.a1 && a2.value === props.item.answers.a2 && a3.value === props.item.answers.a3)

function submit() {
  if (!watched.value || a1.value === null || a2.value === null || a3.value === null) {
    showWarning.value = true
    return
  }
  if (props.item.practice && !feedback.value) {
    showWarning.value = false
    feedback.value = true
    return
  }
  emit('submit', { a1: a1.value, a2: a2.value, a3: a3.value, seconds: Math.round((Date.now() - started) / 1000) })
}
</script>

<template>
  <p class="task">
    <span class="eyebrow">{{ zh ? '場景' : 'Scene' }}</span> {{ zh ? item.task.zh : item.task.en }}
  </p>
  <p v-if="problemPair" class="problem">
    <span class="eyebrow">{{ zh ? '問題' : 'Problem' }}</span>
    {{ zh ? item.problem.zh : item.problem.en }}
    <span v-if="item.expected" class="expected">
      <span class="eyebrow">{{ zh ? '正確應該是' : 'Should be' }}</span>
      {{ zh ? item.expected.zh : item.expected.en }}
    </span>
  </p>

  <div class="video-container">
    <figure>
      <video ref="leftEl" :src="src(item.left)" muted playsinline preload="auto" controls
             @ended="onEnded('left')"></video>
      <figcaption class="badge">A</figcaption>
    </figure>
    <figure>
      <video ref="rightEl" :src="src(item.right)" muted playsinline preload="auto" controls
             @ended="onEnded('right')"></video>
      <figcaption class="badge">B</figcaption>
    </figure>
  </div>
  <div class="play-row">
    <button class="play" @click="playBoth">{{ autoplayBlocked ? (zh ? '同時播放兩支影片' : 'Play both videos') : (zh ? '從頭同步播放' : 'Restart both') }}</button>
    <span class="hint" v-if="!watched">
      {{ zh ? '兩支影片會同步重複播放，各看完一次後就可以作答。' : 'Both videos play in sync and repeat. You can answer once each has played to the end.' }}
    </span>
  </div>

  <div class="question" v-for="(q, i) in questions" :key="q.key">
    <p><span class="qnum">Q{{ i + 1 }}</span> <span v-html="zh ? q.zh : q.en"></span>
      <span class="qhint" v-if="q.zhHint">{{ zh ? '（' + q.zhHint + '）' : '(' + q.enHint + ')' }}</span></p>
    <div class="choices">
      <button v-for="opt in (q.same ? ['L', 'S', 'R'] : ['L', 'R'])" :key="opt" type="button" class="choice"
              :class="{ selected: answers[q.key].value === opt, correct: feedback && item.answers[q.key] === opt }"
              :disabled="!watched || feedback"
              @click="answers[q.key].value = opt">
        {{ opt === 'L' ? 'A' : opt === 'R' ? 'B' : (zh ? '差不多' : 'About the same') }}
      </button>
    </div>
  </div>

  <p class="green" v-if="showWarning">
    {{ zh ? '請看完兩支影片並回答所有問題後再送出。' : 'Please watch both videos and answer every question before submitting.' }}
  </p>
  <div v-if="feedback" class="feedback" :class="{ right: correct }">
    <b>{{ correct ? (zh ? '答對了！' : 'Correct!') : (zh ? '正確答案已用框線標出。' : 'The correct answers are outlined.') }}</b>
    {{ zh ? item.explain.zh : item.explain.en }}
  </div>
  <button class="primary submit" @click="submit">
    {{ !item.practice ? (zh ? '送出' : 'Submit')
       : !feedback ? (zh ? '看答案' : 'Check answer')
       : lastPractice ? (zh ? '繼續' : 'Continue')
       : (zh ? '下一題例題' : 'Next practice') }}
  </button>
</template>

<style scoped>
p {
  margin: 10px 0;
}

.eyebrow {
  display: inline-block;
  font-family: var(--serif);
  font-weight: 600;
  font-size: 0.8em;
  letter-spacing: 0.06em;
  text-transform: uppercase;
  color: var(--clay);
  margin-right: 8px;
}

.task {
  font-size: 1.05em;
}

/* same treatment as the repair strips in the paper's Fig. 5 */
.problem {
  background: var(--clay-soft);
  border-radius: 6px;
  padding: 10px 14px;
  font-size: 1.08em;
}

.problem::before {
  content: '';
  display: inline-block;
  border-left: 7px solid transparent;
  border-right: 7px solid transparent;
  border-top: 11px solid var(--clay-strong);
  margin-right: 8px;
}

.expected {
  display: block;
  margin-top: 6px;
  padding-top: 6px;
  border-top: 1px dashed var(--clay-strong);
  color: var(--ok);
}

.expected .eyebrow {
  color: var(--ok);
}

.video-container {
  display: flex;
  justify-content: center;
  gap: 14px;
  margin: 14px 0 10px;
}

figure {
  position: relative;
  width: 50%;
  margin: 0;
}

video {
  width: 100%;
  display: block;
  border-radius: 6px;
  background: #000;
}

.badge {
  position: absolute;
  top: 8px;
  left: 8px;
  font-family: var(--serif);
  font-weight: 600;
  font-size: 1.1em;
  color: #fff;
  background: rgba(29, 28, 26, 0.72);
  border-radius: 6px;
  padding: 0 10px;
  pointer-events: none;
}

.play-row {
  display: flex;
  flex-wrap: wrap;
  gap: 12px;
  align-items: center;
  margin-bottom: 14px;
}

.play {
  border: 1px solid var(--rule);
  background: var(--card);
  color: var(--ink);
}

.hint {
  font-size: 0.9em;
  color: var(--muted);
}

.question {
  padding: 12px 0;
  border-top: 1px solid var(--rule);
}

.question p {
  font-size: 1.08em;
  margin: 0 0 8px;
}

.qnum {
  font-family: var(--serif);
  font-weight: 600;
  color: var(--clay);
  margin-right: 6px;
}

.question :deep(em) {
  font-style: normal;
  font-weight: 700;
  color: var(--clay);
}

.question :deep(em.prob) {
  background: var(--clay-soft);
  border-radius: 4px;
  padding: 0 4px;
}

.qhint {
  font-size: 0.9em;
  color: var(--muted);
}

.choices {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
}

.choice {
  min-width: 104px;
  height: 44px;
  font-family: var(--serif);
  font-size: 1.1em;
  border: 1.5px solid var(--rule);
  border-radius: 8px;
  background: var(--card);
  color: var(--ink);
  cursor: pointer;
  transition: 0.15s;
}

.choice:hover:not(:disabled) {
  border-color: var(--clay-strong);
}

.choice.selected {
  border-color: var(--clay-strong);
  background: var(--clay-strong);
  color: #fff;
  font-weight: 600;
}

.choice:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

.feedback .choice:disabled,
.choice.correct:disabled,
.choice.selected:disabled {
  opacity: 1;
}

.choice.correct {
  outline: 3px solid #4a8a3c;
  outline-offset: 2px;
}

.feedback {
  margin: 12px 0 6px;
  padding: 12px 14px;
  border-radius: 8px;
  background: var(--card);
  border: 1px solid var(--rule);
  line-height: 1.6;
}

.feedback.right b {
  color: #4a8a3c;
}

button {
  font-size: 1em;
  padding: 8px 20px;
  height: auto;
  border-radius: 8px;
  cursor: pointer;
}

.submit {
  margin-top: 6px;
}

.green {
  color: var(--clay);
}

@media (max-width: 700px) {
  .video-container {
    flex-direction: column;
  }

  figure {
    width: 100%;
  }
}
</style>
