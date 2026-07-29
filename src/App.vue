<script setup lang="ts">
import { ref, onMounted, computed } from 'vue';
import axios from 'axios';

interface Prize {
  rank: string;
  amount: string;
  rule: string;
  number: string;
}

interface Lottery {
  round: string;
  name: string;
  draw_date: string;
  prizes: Prize[];
}

interface Stats {
  lotteries: number;
  'lotteries/zenkoku': number;
  'lotteries/tokyo': number;
  'lotteries/kct': number;
  'lotteries/kinki': number;
  'lotteries/nishinihon': number;
  'lotteries/chiiki': number;
  total: number;
}

interface CheckResult {
  number: string;
  result: string;
  isWin: boolean;
}

const lotteries = ref<Lottery[]>([]);
const selectedRound = ref('');

const numberInputRefs = ref<(HTMLInputElement | null)[]>([]);

const lotteryGroup = ref('');

const lotteryNumbers = ref(['', '', '', '', '', '']);

const results = ref<CheckResult[]>([]);
const loading = ref(false);

const inputMode = ref<'single' | 'renban'>('single');
const renbanCount = ref(10);

const lotteryType = ref('jumbo');

const LOTTERY_TYPES: {
  key: string;
  label: string;
  endpoint: string;
  statsKey: keyof Omit<Stats, 'total'>;
}[] = [
  { key: 'jumbo', label: 'ジャンボ宝くじ', endpoint: '/api/lotteries', statsKey: 'lotteries' },
  {
    key: 'zenkoku',
    label: '全国通常宝くじ',
    endpoint: '/api/lotteries/zenkoku',
    statsKey: 'lotteries/zenkoku',
  },
  {
    key: 'tokyo',
    label: '東京都宝くじ',
    endpoint: '/api/lotteries/tokyo',
    statsKey: 'lotteries/tokyo',
  },
  {
    key: 'kct',
    label: '関東・中部・東北自治宝くじ',
    endpoint: '/api/lotteries/kct',
    statsKey: 'lotteries/kct',
  },
  {
    key: 'kinki',
    label: '近畿宝くじ',
    endpoint: '/api/lotteries/kinki',
    statsKey: 'lotteries/kinki',
  },
  {
    key: 'nishinihon',
    label: '西日本宝くじ',
    endpoint: '/api/lotteries/nishinihon',
    statsKey: 'lotteries/nishinihon',
  },
  {
    key: 'chiiki',
    label: '地域医療等振興自治宝くじ',
    endpoint: '/api/lotteries/chiiki',
    statsKey: 'lotteries/chiiki',
  },
];

const stats = ref<Stats | null>(null);

const fetchStats = async () => {
  try {
    const res = await axios.get<Stats>(`${import.meta.env.VITE_API_URL}/api/stats`);
    stats.value = res.data;
  } catch (error) {
    console.error(error);
  }
};

// 全角→半角に変換
const toHalfWidth = (str: string): string => {
  return str.replace(/[０-９]/g, (s: string) => {
    return String.fromCharCode(s.charCodeAt(0) - 0xfee0);
  });
};

// 回と当選番号の情報取得
const fetchLotteries = async () => {
  loading.value = true;
  try {
    const type = LOTTERY_TYPES.find((t) => t.key === lotteryType.value);
    if (!type) return;
    const res = await axios.get<Lottery[]>(`${import.meta.env.VITE_API_URL}${type.endpoint}`);

    lotteries.value = res.data;
  } catch (error) {
    console.error(error);
  }
  loading.value = false;
};

const switchType = (key: string) => {
  lotteryType.value = key;
  lotteries.value = [];
  selectedRound.value = '';
  lotteryGroup.value = '';
  lotteryNumbers.value = ['', '', '', '', '', ''];
  results.value = [];
  fetchLotteries();
};

const switchMode = (mode: 'single' | 'renban') => {
  inputMode.value = mode;
  results.value = [];
};

// 数字のみ入力受付
const onlyNumberInput = (event: Event, index: number) => {
  const value = toHalfWidth((event.target as HTMLInputElement).value);

  const numberOnly = value.replace(/[^0-9]/g, '');

  lotteryNumbers.value[index] = numberOnly;
};

// 番号のペースト対応
const pasteNumbers = (event: ClipboardEvent) => {
  const pastedText = event.clipboardData?.getData('text') ?? '';

  const numbersOnly = pastedText.replace(/[^0-9]/g, '');

  if (!numbersOnly) {
    return;
  }

  event.preventDefault();

  numbersOnly
    .slice(0, 6)
    .split('')
    .forEach((num: string, index: number) => {
      lotteryNumbers.value[index] = num;
    });

  const lastIndex = Math.min(numbersOnly.length, 6) - 1;

  numberInputRefs.value[lastIndex]?.focus();
};

// 入力時、次の桁のインプットボックスに移動
const moveNextInput = (index: number) => {
  if (lotteryNumbers.value[index] && index < 5) {
    numberInputRefs.value[index + 1]?.focus();
  }
};

// 数字削除時、前の桁のインプットボックスに移動
const movePrevInput = (event: KeyboardEvent, index: number) => {
  if (event.key === 'Backspace' && !lotteryNumbers.value[index] && index > 0) {
    numberInputRefs.value[index - 1]?.focus();
  }
};

const isSelectedRound = computed(() => {
  return selectedRound.value;
});

onMounted(() => {
  fetchLotteries();
  fetchStats();
});

const selectedLottery = computed(() => {
  return lotteries.value.find((lottery) => lottery.round === selectedRound.value);
});

// 1番号に対する当選チェック（結果文字列を返す）
const checkSingleNumber = (
  inputGroup: string,
  inputNumber: string,
): { result: string; isWin: boolean } => {
  if (!selectedLottery.value) return { result: 'ハズレ', isWin: false };

  const prizes = selectedLottery.value.prizes;
  const firstPrize = prizes.find((prize) => prize.rank.includes('1等'));

  for (const prize of prizes) {
    if (prize.rank.includes('前後賞') && firstPrize) {
      const firstNumber = Number(firstPrize.number);
      const prevNumber = String(firstNumber - 1).padStart(6, '0');
      const nextNumber = String(firstNumber + 1).padStart(6, '0');

      if (inputNumber === prevNumber || inputNumber === nextNumber) {
        return { result: `${prize.rank} 当選 (${prize.amount})`, isWin: true };
      }
    }

    if (prize.rank.includes('組違い賞') && firstPrize) {
      if (
        inputNumber === firstPrize.number &&
        inputGroup !== firstPrize.rule.match(/(\d+)組/)?.[1]
      ) {
        return { result: `${prize.rank} 当選 (${prize.amount})`, isWin: true };
      }
    }

    const rule = prize.rule;

    const tailMatch = rule.match(/下(\d)ケタ/);

    if (tailMatch && !rule.includes('組下')) {
      const digit = Number(tailMatch[1]);
      const inputTail = inputNumber.slice(-digit);
      const prizeTail = prize.number.slice(-digit);

      if (inputTail === prizeTail) {
        return { result: `${prize.rank} 当選 (${prize.amount})`, isWin: true };
      }
    }

    if (rule.includes('各組共通')) {
      if (inputNumber === prize.number) {
        return { result: `${prize.rank} 当選 (${prize.amount})`, isWin: true };
      }
    }

    const exactGroupMatch = rule.match(/(\d+)組/);

    if (exactGroupMatch && !rule.includes('組下')) {
      const prizeGroup = exactGroupMatch[1];

      if (inputGroup === prizeGroup && inputNumber === prize.number) {
        return { result: `${prize.rank} 当選 (${prize.amount})`, isWin: true };
      }
    }

    const groupTailMatch = rule.match(/組下(\d)ケタ(\d+)組/);

    if (groupTailMatch) {
      const digit = Number(groupTailMatch[1]);
      const targetGroupTail = groupTailMatch[2];
      const inputGroupTail = inputGroup.slice(-digit);

      if (inputGroupTail === targetGroupTail && inputNumber === prize.number) {
        return { result: `${prize.rank} 当選 (${prize.amount})`, isWin: true };
      }
    }
  }

  return { result: 'ハズレ', isWin: false };
};

// 当選チェック
const checkLottery = () => {
  results.value = [];

  if (!selectedLottery.value) {
    return;
  }

  const startNumber = lotteryNumbers.value.join('');
  const inputGroup = lotteryGroup.value;

  if (inputMode.value === 'single') {
    const { result, isWin } = checkSingleNumber(inputGroup, startNumber);
    results.value = [{ number: startNumber, result, isWin }];
  } else {
    const startNum = parseInt(startNumber, 10);
    const count = Math.max(2, Math.min(renbanCount.value, 100));
    for (let i = 0; i < count; i++) {
      const num = String(startNum + i).padStart(6, '0');
      const { result, isWin } = checkSingleNumber(inputGroup, num);
      results.value.push({ number: num, result, isWin });
    }
  }
};

const hasResults = computed(() => results.value.length > 0);
const winCount = computed(() => results.value.filter((r) => r.isWin).length);
</script>

<template>
  <div class="page-wrapper">
    <div class="container">
      <h1 class="title">宝くじ当選チェッカー</h1>

      <div class="lottery-type-area">
        <div class="lottery-type-label-row">
          <label>宝くじの種類</label>
        </div>
        <select
          class="select-type"
          :value="lotteryType"
          @change="switchType(($event.target as HTMLSelectElement).value)"
        >
          <option v-for="type in LOTTERY_TYPES" :key="type.key" :value="type.key">
            {{ type.label }}
          </option>
        </select>
      </div>

      <div v-if="loading" class="loading-overlay">
        <div class="loading-content">データ取得中です...</div>
      </div>

      <div v-else class="form-area">
        <label> 回を選択 </label>

        <select class="select-round" v-model="selectedRound">
          <option value="">回を選択</option>

          <option v-for="lottery in lotteries" :key="lottery.round" :value="lottery.round">
            第{{ lottery.round }}回
            {{ lottery.name }}
          </option>
        </select>
        <div v-if="isSelectedRound" class="number-input-area">
          <div class="mode-toggle">
            <button
              class="mode-btn"
              :class="{ active: inputMode === 'single' }"
              @click="switchMode('single')"
            >
              単番
            </button>
            <button
              class="mode-btn"
              :class="{ active: inputMode === 'renban' }"
              @click="switchMode('renban')"
            >
              連番
            </button>
          </div>

          <div class="group-input-area">
            <p>組を入力してください</p>
            <input
              v-model="lotteryGroup"
              @input="lotteryGroup = toHalfWidth(lotteryGroup).replace(/[^0-9]/g, '')"
              type="text"
              class="group-input"
              placeholder="組数"
            />
          </div>

          <p class="input-number-info">
            {{
              inputMode === 'renban' ? '開始番号を入力してください' : '宝くじ番号を入力してください'
            }}
          </p>
          <div class="number-inputs">
            <input
              v-for="(_number, index) in lotteryNumbers"
              :key="index"
              :ref="(el) => (numberInputRefs[index] = el as HTMLInputElement | null)"
              v-model="lotteryNumbers[index]"
              type="text"
              maxlength="1"
              class="number-input"
              @input="
                onlyNumberInput($event, index);
                moveNextInput(index);
              "
              @keydown="movePrevInput($event, index)"
              @paste="pasteNumbers"
            />
          </div>

          <div v-if="inputMode === 'renban'" class="renban-count-area">
            <label class="renban-count-label">枚数</label>
            <input
              v-model.number="renbanCount"
              type="number"
              min="2"
              max="100"
              class="renban-count-input"
            />
            <span class="renban-count-unit">枚</span>
          </div>

          <button class="lottery-check" @click="checkLottery">照合開始</button>

          <div v-if="hasResults" class="results-area">
            <template v-if="inputMode === 'single'">
              <p
                class="result-message"
                :class="{ win: results[0]?.isWin, lose: !results[0]?.isWin }"
              >
                照合結果：{{ results[0]?.result }}
              </p>
            </template>
            <template v-else>
              <p class="renban-summary">
                照合結果：{{ winCount > 0 ? `${winCount}枚当選` : '全てハズレ' }}
              </p>
              <ul class="renban-results">
                <li
                  v-for="item in results"
                  :key="item.number"
                  class="renban-result-item"
                  :class="{ win: item.isWin, lose: !item.isWin }"
                >
                  <span class="renban-number">{{ item.number }}</span>
                  <span class="renban-result-text">{{ item.result }}</span>
                </li>
              </ul>
            </template>
          </div>
        </div>
      </div>
    </div>

    <footer class="footer">
      <p class="footer-text">Front: Vercel, Back: Render</p>
      <p class="footer-text">© {{ new Date().getFullYear() }} 宝くじ当選チェッカー | Akifumi Doi</p>
    </footer>
  </div>
</template>

<style>
body {
  margin: 0;
  font-family: sans-serif;
  min-height: 100vh;
}

#app {
  min-height: 100vh;
}

.page-wrapper {
  display: grid;
  grid-template-rows: 1fr auto;
  min-height: 100vh;
}

.container {
  max-width: 40rem;
  margin: 0 auto;
  padding: 2rem;
  width: 100%;
  box-sizing: border-box;
}

.title {
  margin-top: 10rem;
  margin-bottom: 3rem;
}

.lottery-type-area {
  margin-bottom: 1.5rem;
}

.lottery-type-label-row {
  display: flex;
  align-items: baseline;
  gap: 1rem;
  margin-bottom: 0.4rem;
}

.select-type {
  display: block;
  width: 70%;
}

.form-area {
  font-size: large;
}

.select-round {
  width: 50%;
}

.number-input-area {
  margin-top: 1.5rem;
}

.mode-toggle {
  display: flex;
  gap: 0;
  margin-bottom: 1rem;
  width: fit-content;
}

.mode-btn {
  padding: 0.4rem 1.2rem;
  font-size: 0.95rem;
  cursor: pointer;
  border: 1px solid #aaa;
  background: #f5f5f5;
  color: #555;
}

.mode-btn:first-child {
  border-radius: 4px 0 0 4px;
}

.mode-btn:last-child {
  border-radius: 0 4px 4px 0;
  border-left: none;
}

.mode-btn.active {
  background: #333;
  color: #fff;
  border-color: #333;
}

.input-number-info {
  margin-top: 1rem;
}

.number-inputs {
  display: flex;
  gap: 0.5rem;
}

.number-input {
  width: 3rem;
  height: 3rem;
  text-align: center;
  font-size: 1.5rem;
}

.group-input-area {
  margin-top: 1rem;
}

.group-input {
  width: 6rem;
  padding: 0.5rem;
  font-size: 1rem;
}

.renban-count-area {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  margin-top: 0.75rem;
}

.renban-count-label {
  font-size: 1rem;
}

.renban-count-input {
  width: 4rem;
  padding: 0.4rem;
  font-size: 1rem;
}

.renban-count-unit {
  font-size: 1rem;
}

.lottery-check {
  display: block;
  margin: 0.75rem 0 0 calc(2 * (3rem + 0.5rem));
  padding: 0.75rem 1.5rem;
  cursor: pointer;
}

.results-area {
  margin-top: 1rem;
}

.result-message {
  font-size: xx-large;
}

.result-message.win {
  color: #c00;
  font-weight: bold;
}

.result-message.lose {
  color: inherit;
}

.renban-summary {
  font-size: x-large;
  font-weight: bold;
  margin-bottom: 0.75rem;
}

.renban-results {
  list-style: none;
  padding: 0;
  margin: 0;
  border: 1px solid #ddd;
  border-radius: 6px;
  overflow: hidden;
}

.renban-result-item {
  display: flex;
  align-items: center;
  gap: 1rem;
  padding: 0.5rem 0.75rem;
  border-bottom: 1px solid #eee;
  font-size: 1rem;
}

.renban-result-item:last-child {
  border-bottom: none;
}

.renban-result-item.win {
  background: #fff5f5;
  color: #c00;
  font-weight: bold;
}

.renban-result-item.lose {
  color: #888;
}

.renban-number {
  font-family: monospace;
  font-size: 1.1rem;
  min-width: 5rem;
}

.renban-result-text {
  flex: 1;
}

.loading-overlay {
  position: fixed;
  top: 0;
  left: 0;

  width: 100%;
  height: 100%;

  background: rgba(0, 0, 0, 0.5);

  display: flex;
  justify-content: center;
  align-items: center;

  z-index: 9999;
}

.loading-content {
  background: white;
  padding: 2rem 3rem;
  border-radius: 1rem;
  font-size: 1.25rem;
  color: black;
  box-shadow: 0 0 20px rgba(0, 0, 0, 0.3);
}

.footer {
  width: 100%;
  box-sizing: border-box;
  padding: 1.5rem;
  text-align: center;
  border-top: 1px solid #ddd;
  background-color: #f8f8f8;
}

.footer-text {
  margin: 0 0 0.25rem;
  font-size: 0.875rem;
  color: #555;
}

.footer-note {
  margin: 0;
  font-size: 0.75rem;
  color: #999;
}

@media (max-width: 600px) {
  .container {
    padding: 1.25rem;
  }

  .title {
    margin-top: 3rem;
    font-size: 1.5rem;
  }

  .select-type,
  .select-round {
    width: 100%;
  }

  .result-message {
    font-size: x-large;
  }
}

@media (max-width: 400px) {
  .container {
    padding: 1rem;
  }

  .number-input {
    width: 2.5rem;
    height: 2.5rem;
    font-size: 1.2rem;
  }

  .lottery-check {
    margin-left: calc(2 * (2.5rem + 0.5rem));
  }
}
</style>
