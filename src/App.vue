<script setup>
import { ref, onMounted, computed } from 'vue';
import axios from 'axios';

const lotteries = ref([]);
const selectedRound = ref('');

const numberInputRefs = ref([]);

const lotteryGroup = ref('');

const lotteryNumbers = ref(['', '', '', '', '', '']);

const resultMessage = ref('');
const loading = ref(false);

// 全角→半角に変換
const toHalfWidth = (str) => {
  return str.replace(/[０-９]/g, (s) => {
    return String.fromCharCode(s.charCodeAt(0) - 0xfee0);
  });
};

// 回と当選番号の情報取得
const fetchLotteries = async () => {
  loading.value = true;
  try {
    const res = await axios.get(`${import.meta.env.VITE_API_URL}/api/lotteries`);

    lotteries.value = res.data;
  } catch (error) {
    console.error(error);
  }
  loading.value = false;
};

// 数字のみ入力受付
const onlyNumberInput = (event, index) => {
  const value = toHalfWidth(event.target.value);

  const numberOnly = value.replace(/[^0-9]/g, '');

  lotteryNumbers.value[index] = numberOnly;
};

// 番号のペースト対応
const pasteNumbers = (event) => {
  const pastedText = event.clipboardData.getData('text');

  const numbersOnly = pastedText.replace(/[^0-9]/g, '');

  if (!numbersOnly) {
    return;
  }

  event.preventDefault();

  numbersOnly
    .slice(0, 6)
    .split('')
    .forEach((num, index) => {
      lotteryNumbers.value[index] = num;
    });

  const lastIndex = Math.min(numbersOnly.length, 6) - 1;

  numberInputRefs.value[lastIndex]?.focus();
};

// 入力時、次の桁のインプットボックスに移動
const moveNextInput = (index) => {
  if (lotteryNumbers.value[index] && index < 5) {
    numberInputRefs.value[index + 1]?.focus();
  }
};

// 数字削除時、前の桁のインプットボックスに移動
const movePrevInput = (event, index) => {
  if (event.key === 'Backspace' && !lotteryNumbers.value[index] && index > 0) {
    numberInputRefs.value[index - 1]?.focus();
  }
};

const isSelectedRound = computed(() => {
  return selectedRound.value;
});

onMounted(() => {
  fetchLotteries();
});

const selectedLottery = computed(() => {
  return lotteries.value.find((lottery) => lottery.round === selectedRound.value);
});

// 当選チェック
const checkLottery = () => {
  resultMessage.value = '';

  if (!selectedLottery.value) {
    return;
  }

  const inputNumber = lotteryNumbers.value.join('');
  const inputGroup = lotteryGroup.value;

  const prizes = selectedLottery.value.prizes;

  const firstPrize = prizes.find((prize) => prize.rank.includes('1等'));

  for (const prize of prizes) {
    // ====================
    // 1等の前後賞
    // ====================

    if (prize.rank.includes('前後賞') && firstPrize) {
      const firstNumber = Number(firstPrize.number);

      const prevNumber = String(firstNumber - 1).padStart(6, '0');

      const nextNumber = String(firstNumber + 1).padStart(6, '0');

      if (inputNumber === prevNumber || inputNumber === nextNumber) {
        resultMessage.value = `${prize.rank} 当選 (${prize.amount})`;

        return;
      }
    }

    // ====================
    // 1等の組違い賞
    // ====================

    if (prize.rank.includes('組違い賞') && firstPrize) {
      if (
        inputNumber === firstPrize.number &&
        inputGroup !== firstPrize.rule.match(/(\d+)組/)?.[1]
      ) {
        resultMessage.value = `${prize.rank} 当選 (${prize.amount})`;

        return;
      }
    }

    const rule = prize.rule;

    // ====================
    // 下◯ケタ判定
    // ====================

    const tailMatch = rule.match(/下(\d)ケタ/);

    if (tailMatch && !rule.includes('組下')) {
      const digit = Number(tailMatch[1]);

      const inputTail = inputNumber.slice(-digit);

      const prizeTail = prize.number.slice(-digit);

      if (inputTail === prizeTail) {
        resultMessage.value = `${prize.rank} 当選 (${prize.amount})`;

        return;
      }
    }

    // ====================
    // 各組共通
    // ====================

    if (rule.includes('各組共通')) {
      if (inputNumber === prize.number) {
        resultMessage.value = `${prize.rank} 当選 (${prize.amount})`;

        return;
      }
    }

    // ====================
    // xx組
    // ====================

    const exactGroupMatch = rule.match(/(\d+)組/);

    if (exactGroupMatch && !rule.includes('組下')) {
      const prizeGroup = exactGroupMatch[1];

      if (inputGroup === prizeGroup && inputNumber === prize.number) {
        resultMessage.value = `${prize.rank} 当選 (${prize.amount})`;

        return;
      }
    }

    // ====================
    // 組下◯ケタ◯組
    // ====================

    const groupTailMatch = rule.match(/組下(\d)ケタ(\d+)組/);

    if (groupTailMatch) {
      const digit = Number(groupTailMatch[1]);

      const targetGroupTail = groupTailMatch[2];

      const inputGroupTail = inputGroup.slice(-digit);

      if (inputGroupTail === targetGroupTail && inputNumber === prize.number) {
        resultMessage.value = `${prize.rank} 当選 (${prize.amount})`;

        return;
      }
    }
  }

  resultMessage.value = 'ハズレ';
};
</script>

<template>
  <div class="page-wrapper">
    <div class="container">
      <h1>ジャンボ宝くじ当選チェッカー</h1>

      <div v-if="loading" class="loading-overlay">
        <div class="loading-content">データ取得中です...</div>
      </div>

      <div v-else class="form-area">
        <label> 回を選択 </label>

        <select v-model="selectedRound">
          <option value="">回を選択</option>

          <option v-for="lottery in lotteries" :key="lottery.round" :value="lottery.round">
            第{{ lottery.round }}回
            {{ lottery.name }}
          </option>
        </select>
        <div v-if="isSelectedRound" class="number-input-area">
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

          <p class="input-number-info">宝くじ番号を入力してください</p>
          <div class="number-inputs">
            <input
              v-for="(number, index) in lotteryNumbers"
              :key="index"
              :ref="(el) => (numberInputRefs[index] = el)"
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

          <button class="lottery-check" @click="checkLottery">照合開始</button>

          <p v-if="resultMessage" class="result-message">照合結果:{{ resultMessage }}</p>
        </div>
      </div>
    </div>

    <footer class="footer">
      <p class="footer-text">© {{ new Date().getFullYear() }} ジャンボ宝くじ当選チェッカー | Akifumi Doi</p>
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

.input-box {
  width: 100%;
  padding: 1rem;
  margin-top: 1rem;
  box-sizing: border-box;
}

.check-button {
  margin-top: 1rem;
  padding: 1rem;
  cursor: pointer;
}

.result-box {
  margin-top: 2rem;
  border: 1px solid #ccc;
  padding: 1rem;
}

.number-input-area {
  margin-top: 1.5rem;
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

.lottery-check {
  display: block;
  margin: 1rem auto 0;
  padding: 0.75rem 1.5rem;
  cursor: pointer;
}

.result-message {
  margin-top: 1rem;
  font-size: xx-large;
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
</style>
