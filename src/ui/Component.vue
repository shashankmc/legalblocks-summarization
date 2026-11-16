<template>
  <div class="summ-shell">
    <div class="summ-header">
      <span class="summ-eyebrow">nlp · summarization</span>
      <h3 class="summ-title">Summarization</h3>
      <span class="summ-stats" v-if="summary">
        {{ wordCount }} words · {{ modelLabel }}
      </span>
    </div>

    <div class="summ-controls">
      <div class="summ-field">
        <label class="summ-label">Model</label>
        <select v-model="selectedModel" class="summ-select" :disabled="loading">
          <optgroup label="Extractive (sumy)">
            <option value="lex_rank">LexRank (default)</option>
            <option value="lsa">LSA</option>
            <option value="text_rank">TextRank</option>
            <option value="luhn">Luhn</option>
            <option value="edmundson">Edmundson</option>
          </optgroup>
          <optgroup label="Abstractive (HuggingFace)">
            <option value="hf:sshleifer/distilbart-cnn-12-6">distilBART (fast)</option>
            <option value="hf:facebook/bart-large-cnn">BART large</option>
            <option value="hf:yhavinga/ul2-large-dutch">UL2 Dutch</option>
          </optgroup>
        </select>
      </div>

      <div class="summ-field">
        <label class="summ-label">Length</label>
        <div class="summ-length-group">
          <button
            v-for="opt in ['short', 'medium', 'long']"
            :key="opt"
            :class="['summ-len-btn', { 'summ-len-btn--active': selectedLength === opt }]"
            :disabled="loading"
            @click="selectedLength = opt"
          >
            {{ opt }}
          </button>
        </div>
      </div>
    </div>

    <button class="summ-run-btn" :disabled="!inputText || loading" @click="run">
      <span v-if="loading" class="summ-spinner" />
      {{ loading ? 'Summarising…' : 'Summarise' }}
    </button>

    <div v-if="error" class="summ-error" role="alert">{{ error }}</div>

    <div v-if="summary" class="summ-result">
      <p class="summ-text">{{ summary }}</p>
      <div class="summ-meta">
        <span>{{ modelLabel }}</span>
        <span>·</span>
        <span>{{ selectedLength }}</span>
        <span>·</span>
        <span>{{ wordCount }} words</span>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, inject, watch } from 'vue'

// Props injected by the LegalBlocks canvas runtime.
// Falls back to defaults so the component is also usable standalone.
const props = defineProps({
  serviceUrl:    { type: String, default: 'http://legal-summarizer-service:8000' },
  model:         { type: String, default: 'lex_rank' },
  summaryLength: { type: String, default: 'medium' },
  apiKey:        { type: String, default: '' },
  // text_layer artifact from the upstream port, provided by the canvas runtime
  textLayer:     { type: Object, default: null },
})

const emit = defineEmits(['output'])

const selectedModel  = ref(props.model)
const selectedLength = ref(props.summaryLength)
const loading        = ref(false)
const summary        = ref('')
const error          = ref('')

// Pull the document text out of the upstream text_layer artifact.
// text_layer schema: { text: string } (see @lawnotation/artifact-schemas)
const inputText = computed(() => props.textLayer?.payload?.text ?? '')

const wordCount  = computed(() => summary.value.split(/\s+/).filter(Boolean).length)
const modelLabel = computed(() => {
  const m = selectedModel.value
  if (m.startsWith('hf:')) return m.replace('hf:', '')
  return { lex_rank: 'LexRank', lsa: 'LSA', text_rank: 'TextRank', luhn: 'Luhn', edmundson: 'Edmundson' }[m] ?? m
})

function buildHeaders() {
  const h = { 'Content-Type': 'application/json' }
  if (props.apiKey) h['Authorization'] = `Bearer ${props.apiKey}`
  return h
}

async function pollUntilDone(base, requestId) {
  for (let i = 0; i < 120; i++) {
    await new Promise(r => setTimeout(r, 1500))
    const res = await fetch(`${base}/status/${requestId}`, { headers: buildHeaders() })
    if (!res.ok) throw new Error(`Status poll failed: ${res.status}`)
    const update = await res.json()
    if (update.status === 'complete') return update.summarized_text
    if (update.status === 'error')    throw new Error(update.error ?? 'Summarization failed')
  }
  throw new Error('Timed out waiting for summary')
}

async function run() {
  if (!inputText.value || loading.value) return
  loading.value = true
  error.value   = ''
  summary.value = ''

  const base      = props.serviceUrl.replace(/\/$/, '')
  const requestId = `lb-${Date.now()}-${Math.random().toString(36).slice(2, 7)}`

  try {
    const res = await fetch(`${base}/summarize`, {
      method:  'POST',
      headers: buildHeaders(),
      body: JSON.stringify({
        request_id:     requestId,
        input_text:     inputText.value,
        summary_length: selectedLength.value,
        priority:       { relevance: 0.6, readability: 0.5 },
        model:          selectedModel.value,
      }),
    })
    if (!res.ok) throw new Error(`Request failed: ${res.status} ${res.statusText}`)

    const data = await res.json()

    let text
    if (data.summarized_text !== undefined) {
      // Sync mode: POST returned the result directly
      text = data.summarized_text
    } else {
      // Async mode: poll until complete
      text = await pollUntilDone(base, requestId)
    }

    summary.value = text

    // Emit the summary_layer artifact for downstream canvas connections
    emit('output', {
      port_id:       'summary_layer',
      artifact_type: 'summary-layer',
      schema_version: '1.0.0',
      payload: {
        summary:    text,
        model:      selectedModel.value,
        request_id: requestId,
        length:     selectedLength.value,
      },
      lineage: {
        produced_by_package: 'summarization',
        produced_by_module:  'summarization',
      },
    })
  } catch (e) {
    error.value = e.message ?? 'An unexpected error occurred'
  } finally {
    loading.value = false
  }
}
</script>

<style scoped>
.summ-shell {
  font-family: system-ui, sans-serif;
  padding: 20px;
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.summ-eyebrow {
  font-family: monospace;
  font-size: 10px;
  font-weight: 500;
  letter-spacing: 0.14em;
  text-transform: uppercase;
  color: #8C8579;
  display: block;
}
.summ-title {
  font-size: 20px;
  font-weight: 600;
  margin: 4px 0 0;
  color: #1A1714;
}
.summ-stats {
  font-size: 11px;
  color: #8C8579;
  font-family: monospace;
}

.summ-controls { display: flex; flex-direction: column; gap: 12px; }
.summ-field    { display: flex; flex-direction: column; gap: 4px; }
.summ-label    { font-size: 11px; font-weight: 500; color: #5C564E; text-transform: uppercase; letter-spacing: 0.08em; }

.summ-select {
  padding: 6px 10px;
  border: 1px solid #D9D3C9;
  border-radius: 6px;
  font-size: 13px;
  background: #fff;
  color: #1A1714;
  cursor: pointer;
}
.summ-select:disabled { opacity: 0.5; cursor: not-allowed; }

.summ-length-group { display: flex; gap: 6px; }
.summ-len-btn {
  flex: 1;
  padding: 5px 0;
  border: 1px solid #D9D3C9;
  border-radius: 6px;
  font-size: 12px;
  background: #F7F4EF;
  color: #5C564E;
  cursor: pointer;
  transition: background 0.15s, color 0.15s;
}
.summ-len-btn:hover:not(:disabled)    { background: #EDE8E0; }
.summ-len-btn--active                 { background: #7B4F9E; border-color: #7B4F9E; color: #fff; }
.summ-len-btn:disabled                { opacity: 0.5; cursor: not-allowed; }

.summ-run-btn {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  padding: 9px 18px;
  background: #7B4F9E;
  color: #fff;
  border: none;
  border-radius: 8px;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  transition: background 0.15s;
}
.summ-run-btn:hover:not(:disabled) { background: #6A3F8C; }
.summ-run-btn:disabled             { opacity: 0.4; cursor: not-allowed; }

.summ-spinner {
  display: inline-block;
  width: 12px;
  height: 12px;
  border: 2px solid rgba(255,255,255,0.4);
  border-top-color: #fff;
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
}
@keyframes spin { to { transform: rotate(360deg); } }

.summ-error {
  padding: 10px 14px;
  background: #FEF2F2;
  border: 1px solid #FECACA;
  border-radius: 8px;
  font-size: 13px;
  color: #B91C1C;
}

.summ-result {
  display: flex;
  flex-direction: column;
  gap: 8px;
  padding: 14px;
  background: #F7F4EF;
  border: 1px solid #E5DFD3;
  border-radius: 8px;
}
.summ-text {
  font-size: 14px;
  line-height: 1.65;
  color: #1A1714;
  margin: 0;
}
.summ-meta {
  display: flex;
  gap: 6px;
  font-size: 11px;
  color: #8C8579;
  font-family: monospace;
}
</style>
