<template>
    <div>
      <!-- 금액 입력 및 유형 선택 폼 -->
      <div>
        <form @submit.prevent="registerAmount">
          <input type="date" v-model="whenMoneyUse"/>&nbsp;
          <input type="text" placeholder="용도를 입력해주세요" v-model="whereMoneyUse" /> &nbsp;
          <input type="text" placeholder="금액을 입력해주세요" v-model="amountMoneyUse" /> &nbsp;
          <select v-model="selectedType">
            <option>수입</option>
            <option>지출</option>
          </select>
          &nbsp;
          <button type="submit">등록</button>
        </form>
        &nbsp;<hr/>
      </div>
      
      <!-- 총 수입, 총 지출 및 순 총액 표시 -->
      <div>
        <span :style="{ color: 'green' }">총 입금 액수 : {{ totalIncome }}</span><br />
        <span :style="{ color: 'red' }">총 출금 액수 : {{ totalOutcome }}</span><br />
        <span :style="{ color: totalIncomeOutCome >= 0 ? 'blue' : 'red' }">월 총액 : {{ totalIncomeOutCome }}</span>
        <hr />
    </div>
  
      <!-- 월별로 그룹화된 지출 목록 출력 -->
      <div v-for="(items, month) in groupedSpending" :key="month">
        <h3>{{ month }}월</h3>
        <ol>
          <li v-for="usedMoney in items" :key="usedMoney.id">
            <span :style="usedMoney.type === 'outcome' ? { color: 'red' } : { color: 'green' }">
              {{ usedMoney.year }} {{ usedMoney.month }} {{ usedMoney.day }} : {{ usedMoney.description }} / 가격 : {{ usedMoney.price }}
            </span>
            &nbsp;
            <button @click.stop="deleteSpending(usedMoney.id, usedMoney.type, usedMoney.price)">삭제</button>
          </li>
        </ol>
        <hr />
      </div>
    </div>
  </template>
  
  <script setup>
  import { ref, reactive, computed } from 'vue';
  import axios from 'axios';
  
  // 지출 목록에 대한 반응형 상태
  const states = reactive({
    spending: [],
  });
  
  // API 호출을 위한 기본 URL
  const BASEURL = 'http://localhost:3000/spending';
  
  // API에서 지출 데이터 가져오기
  async function fetchSpending() {
    try {
      const fetchSpendingRes = await axios.get(BASEURL);
      states.spending = fetchSpendingRes.data;
      console.log(states.spending);
  
      // 데이터 가져온 후 총 수입과 총 지출 계산
      calculateTotals();
    } catch (e) {
      alert('spending 데이터 통신 Err 발생');
      console.log(e);
    }
  }
  
  // 폼 입력 및 총 계산에 대한 반응형 참조
  const whereMoneyUse = ref('');
  const amountMoneyUse = ref('');
  const selectedType = ref('수입');
  const totalIncome = ref(0);
  const totalOutcome = ref(0);
  const whenMoneyUse = ref('');
  
  // 순 총액 (수입 - 지출)에 대한 계산된 속성
  const totalIncomeOutCome = computed(() => totalIncome.value - totalOutcome.value);
  
  // 총 수입 및 총 지출 계산 함수
  function calculateTotals() {
    totalIncome.value = 0;
    totalOutcome.value = 0;
  
    states.spending.forEach((usedMoney) => {
      if (usedMoney.type === "income") {
        totalIncome.value += usedMoney.price;
      } else if (usedMoney.type === "outcome") {
        totalOutcome.value += usedMoney.price;
      }
    });
  }
  
  // 데이터 그룹화 함수
function groupByMonth(spending) {
  const grouped = {};

  spending.forEach((item) => {
    const month = item.month;
    if (!grouped[month]) {
      grouped[month] = [];
    }
    grouped[month].push(item);
  });

  // 일별로 정렬
  for (const month in grouped) {
    grouped[month].sort((a, b) => parseInt(a.day) - parseInt(b.day));
  }

  return grouped;
}

  
  // 월별 그룹화된 데이터 계산된 속성
  const groupedSpending = computed(() => groupByMonth(states.spending));
  
  // 폼 제출 시 금액 등록 함수
  async function registerAmount() {
    if (!amountMoneyUse.value || !whenMoneyUse.value) {
      alert('금액과 날짜를 입력해주세요');
      return;
    }
  
    const date = new Date(whenMoneyUse.value);
    const year = date.getFullYear().toString();
    const month = (date.getMonth() + 1).toString();
    const day = date.getDate().toString();
  
    try {
      const newSpending = {
        id: Date.now(),
        description: whereMoneyUse.value,
        price: parseInt(amountMoneyUse.value),
        type: selectedType.value === '수입' ? 'income' : 'outcome',
        year: year,
        month: month,
        day: day,
      };
  
      const addnewSpending = await axios.post(BASEURL, newSpending);
      if (addnewSpending.status !== 201) {
        return alert('newSpending 추가 실패');
      }
  
      if (newSpending.type === 'income') {
        totalIncome.value += newSpending.price;
      } else if (newSpending.type === 'outcome') {
        totalOutcome.value += newSpending.price;
      }
  
      // 폼 입력 초기화
      amountMoneyUse.value = '';
      whereMoneyUse.value = '';
      selectedType.value = '수입';
      whenMoneyUse.value = '';
  
      fetchSpending();
    } catch (e) {
      alert('가계부 추가중 err 발생');
      console.log(e);
    }
  }
  
  async function deleteSpending(id, type, price) {
    console.log(`${BASEURL}/${id}`);
  
    try {
      const deleteSpendingRes = await axios.delete(`${BASEURL}/${id}`);
      if (deleteSpendingRes.status !== 200) {
        alert('Spending 삭제 실패~');
        return;
      }
  
      if (type === 'income') {
        totalIncome.value -= price; // 총 수입에서 가격을 빼야 함
      } else if (type === 'outcome') {
        totalOutcome.value -= price; // 총 지출에서 가격을 빼야 함
      }
  
      fetchSpending();
    } catch (e) {
      if (e.response && e.response.status === 404) {
        alert('삭제하려는 항목을 찾을 수 없습니다 (404 Not Found).');
      } else {
        alert('Spending 삭제 중 오류 발생!');
      }
      console.log(e);
    }
  }
  
  fetchSpending();
  </script>
  
  <style scoped>
  ol {
    list-style: none;
  }
  </style>
  