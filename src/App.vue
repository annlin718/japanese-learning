<script setup>
  import { ref, computed, watch } from "vue";
  import verbData from "./dataList/N5-126Verb.json";
  import audio_O from "./assets/O.mp3";
  import audio_X from "./assets/X.mp3";

  let clickOpen = true;
  const QContentElement = ref(null);
  const QElement = ref(null);

  const availableDates = Object.keys(verbData); // 取得 JSON 中的所有日期
  const selectedDate = ref(availableDates[0]); // 預設選擇第一個日期
  const isTest = ref(false);
  const isTestEnd = ref(false);
  const testQno = ref(1);
  // 用來保存打亂後的列表
  const shuffledList = ref([]);
  // 紀錄測驗答題成果
  const testResultList = ref([]);

  // 根據選擇的日期顯示對應的動詞列表
  const verbList = computed(() => {
    return verbData[selectedDate.value] || []; // 如果沒有找到對應日期的資料，返回空陣列
  });

  // 根據選擇的日期顯示對應的動詞列表
  const shuffledOptionList = computed(() => {
    const firstElement = shuffledList.value[0];

    // 從 shuffledList 中排除第一個元素
    const remainingElements = shuffledList.value.slice(1); // 包含 index 1 到結尾的元素

    // 隨機選擇 3 個其他元素
    const randomElements = [];
    while (randomElements.length < 3) {
      const randomIndex = Math.floor(Math.random() * remainingElements.length);
      const selectedElement = remainingElements[randomIndex];

      // 確保不重複添加元素
      if (!randomElements.includes(selectedElement)) {
        randomElements.push(selectedElement);
      }
    }

    // 組合成新的長度為 4 的陣列
    const finalArray = shuffleArray([firstElement, ...randomElements]);

    return finalArray;
  });

  // Fisher-Yates 洗牌算法來打亂陣列
  function shuffleArray(array) {
    for (let i = array.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1));
      [array[i], array[j]] = [array[j], array[i]];
    }
    return array;
  }

  const enterTest = () => {
    console.log("測驗模式啟動");
    isTest.value = !isTest.value;

    // 清空測驗結果列表
    isTestEnd.value = false;
    testResultList.value = [];
    testQno.value = 1;
    clickOpen = true;

    const originalList = verbList.value;
    const copiedList = JSON.parse(JSON.stringify(originalList)); // 深拷貝
    shuffledList.value = shuffleArray(copiedList); // 打亂順序
  };

  const checkAns = (event) => {
    if (!clickOpen) return;
    clickOpen = false;

    // 透過事件對象來獲取 data-qno 的值
    const dataQno = event.target.dataset.qno; // 獲取 data-qno 的值

    let audio;
    let isRight = false;
    if (shuffledList.value[0].seqno.toString() === dataQno.toString()) {
      // 答對
      audio = new Audio(audio_O);
      event.target.classList.add("animate-O");
      isRight = true;
    } else {
      // 答錯
      audio = new Audio(audio_X);
      event.target.classList.add("animate-X");
    }

    testResultList.value = [
      ...testResultList.value,
      {
        qText: QElement.value.textContent,
        ans:
          shuffledList.value[0].japanese + `\n(${shuffledList.value[0].tones})`,
        memberAns: event.target.textContent.replace(" (", "\n("),
        result: isRight,
      },
    ];

    audio.play().catch((error) => {
      console.error("播放音檔時發生錯誤:", error);
    });

    event.target.addEventListener(
      "animationend",
      function () {
        // 動畫結束後的操作
        // 移除動畫類
        event.target.classList.remove("animate-X", "animate-O");

        if (testQno.value == shuffledList.value.length) {
          // 題目出完了
          console.log("END");
          isTestEnd.value = true;
        } else {
          clickOpen = true;

          const firstElement = shuffledList.value.shift(); // 移除並獲取第一個元素
          shuffledList.value.push(firstElement); // 將該元素添加到陣列末尾
          testQno.value++;
        }
      },
      { once: true }
    );
  };
</script>

<template>
  <div>
    <!-- 日期下拉選單 -->
    <div class="div-day-select">
      <select id="date-select" v-model="selectedDate" v-show="!isTest">
        <option v-for="date in availableDates" :key="date" :value="date">
          {{ date }}
        </option>
      </select>
      <button id="btn-test" @click="enterTest" v-show="verbList.length > 0">
        {{ isTest ? "返回練習" : "測驗模式" }}
      </button>
    </div>

    <!-- 動詞練習 -->
    <div ref="QContentElement" class="div-day-verbs-test" v-if="isTest">
      <div class="div-day-verbs-test-qinfo" v-show="!isTestEnd">
        <p>{{ testQno }} / 10</p>
        <span ref="QElement" class="question-text">{{
          shuffledList[0].translation
        }}</span>
        <div class="div-options-list">
          <button
            v-for="item in shuffledOptionList"
            :key="'btn' + item.seqno"
            :data-qno="item.seqno"
            @click="checkAns"
          >
            {{ item.japanese }} ({{ item.tones }})
          </button>
        </div>
      </div>
      <div class="div-day-verbs-test-result" v-show="isTestEnd">
        <div class="result-item title">
          <span>題號</span>
          <span>題目</span>
          <span>正確答案</span>
          <span>你的答案</span>
          <span></span>
        </div>
        <div
          class="result-item"
          v-for="(testInfo, index) in testResultList"
          :key="'reslut' + index"
        >
          <span>{{ index + 1 }}</span>
          <span>{{ testInfo.qText }}</span>
          <span>{{ testInfo.ans }}</span>
          <span>{{ testInfo.memberAns }}</span>
          <span>{{ testInfo.result ? "O" : "X" }}</span>
        </div>
      </div>
    </div>

    <!-- 動詞列表 -->
    <div class="div-day-verbs-practice" v-else>
      <ol class="custom-ol" v-if="verbList.length > 0">
        <li v-for="item in verbList" :key="item.seqno">
          <b>{{ item.japanese }} ({{ item.tones }}) </b>:
          {{ item.translation }}
        </li>
      </ol>
      <p v-else>該日期沒有資料</p>
    </div>
  </div>
</template>
