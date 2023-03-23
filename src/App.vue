<template>
  <div class="view">
    <div v-if="!registration" class="school-search">
      <h1 class="title">스폰지 '밥'</h1>
      <div class="search-wrap">
        <input
          class="school-input"
          placeholder="학교 이름을 입력해주세요"
          ref="schoolInput"
          @keyup.enter="search"
        />
        <button @click="search" class="school-button">검색</button>
      </div>
      <div class="school-not-exist" v-if="schoolsNotExist">
        검색을 실패 했습니다.
      </div>
      <div class="result-wrap" v-else>
        <div class="is-loading" v-if="isLoading">잠시만요...</div>
        <div
          class="result"
          v-for="(school, index) in schoolList"
          :key="index"
          @click="register(school)"
          v-else
        >
          <div class="result-name">{{ school.SCHUL_NM }}</div>
          <div class="result-address">{{ school.ORG_RDNMA }}</div>
        </div>
      </div>
    </div>
    <div class="main-wrap" v-else>
      <p class="school-name">{{ schoolName }}</p>
      <p class="today">
        {{ parseInt(today.substring(4, 6)) }}월
        {{ parseInt(today.substring(6, 8)) }}일
      </p>
      <p class="timing" v-if="mealExist">{{ timing }}</p>
      <div class="is-loading" v-if="isLoading">잠시만요...</div>
      <div class="menu-wrap" v-if="mealExist && !isLoading">
        <a
          v-for="(menu, index) in meal"
          :key="index"
          class="menu"
          :href="returnURL(menu)"
          target="_blank"
          rel="noopener noreferrer"
        >
          {{ menu }}
        </a>
      </div>
      <p v-else-if="!isLoading" class="nobab">오늘은 밥이 없네요..</p>
      <button class="change-school" @click="resetSchool">학교 변경</button>
    </div>
    <button
      class="button prev-button"
      @click="prevMeal"
      v-if="registration && !isLoading"
    >
      <img src="./assets/prevButton.svg" alt="prev" />
    </button>
    <button
      class="button next-button"
      @click="nextMeal"
      v-if="registration && !isLoading"
    >
      <img src="./assets/nextButton.svg" alt="next" />
    </button>
  </div>
  <BackgroundPattern />
</template>

<script>
import axios from "axios";
import BackgroundPattern from "./components/BackgroundPattern.vue";

export default {
  name: "App",
  components: {
    BackgroundPattern,
  },
  data() {
    return {
      registration: false,
      schoolName: "",
      schoolList: [],
      ATPT_OFCDC_SC_CODE: "",
      SD_SCHUL_CODE: "",
      morning: [],
      lunch: [],
      dinner: [],
      meal: [],
      timing: "",
      today: "",
      isLoading: false,
      schoolsNotExist: false,
      mealExist: false,
    };
  },
  methods: {
    register(school) {
      localStorage.setItem("ATPT_OFCDC_SC_CODE", school.ATPT_OFCDC_SC_CODE);
      localStorage.setItem("SD_SCHUL_CODE", school.SD_SCHUL_CODE);
      localStorage.setItem("schoolName", school.SCHUL_NM);
      window.location.reload();
    },
    registerCheck() {
      this.ATPT_OFCDC_SC_CODE = localStorage.getItem("ATPT_OFCDC_SC_CODE");
      this.SD_SCHUL_CODE = localStorage.getItem("SD_SCHUL_CODE");
      this.registration =
        this.ATPT_OFCDC_SC_CODE && this.SD_SCHUL_CODE ? true : false;
    },
    async search() {
      const name = this.$refs.schoolInput.value;
      this.isLoading = true;
      try {
        const {
          data: {
            schoolInfo: {
              [1]: { row },
            },
          },
        } = await axios.get(
          `https://open.neis.go.kr/hub/schoolInfo?KEY=${process.env.VUE_APP_NEIS_API_KEY}&Type=json&SCHUL_NM=${name}`
        );
        this.schoolList = row;
        this.schoolsNotExist = false;
        this.isLoading = false;
      } catch (e) {
        this.isLoading = false;
        this.schoolsNotExist = true;
      }
    },
    async getMeal() {
      try {
        this.isLoading = true;
        this.initMeals();
        const now = new Date();
        const row = await this.getMealApi();
        this.mealExist = row ? true : false;
        this.assignMeals(row);
        if (now.getHours() >= 13) {
          this.meal =
            this.dinner.length > 0
              ? this.dinner
              : this.lunch.length > 0
              ? this.lunch
              : this.morning;
        } else if (now.getHours() >= 8) {
          this.meal =
            this.lunch.length > 0
              ? this.lunch
              : this.dinner > 0
              ? this.dinner
              : this.morning;
        } else {
          this.meal =
            this.morning.length > 0
              ? this.morning
              : this.lunch.length > 0
              ? this.lunch
              : this.dinner;
        }
        this.timingAssign();
        this.isLoading = false;
        this.schoolName = row[0].SCHUL_NM;
      } catch (e) {
        this.mealExist = false;
        this.isLoading = false;
      }
    },
    toArray(meal) {
      let array = [];
      for (const item of meal.replace(/[*, .]/g, "").split("<br/>")) {
        array.push(item.replace(/^[^가-힣]/g, "").replace(/[^가-힣]$/g, ""));
      }
      return array;
    },
    getToday() {
      const date = new Date();
      const year = date.getFullYear();
      const month = ("0" + (1 + date.getMonth())).slice(-2);
      const day = ("0" + date.getDate()).slice(-2);

      this.today = year + month + day;
    },
    removeBracket(menu) {
      let result = "";
      let stack = [];
      let str = [];

      for (let character of menu) {
        stack.push(character);
      }

      for (let i = 0; i < menu.length; i++) {
        let temp = stack.pop();

        if (temp === "(") {
          while (str.pop() !== ")") {
            console.log();
          }
        } else {
          str.push(temp);
        }
      }

      result = str.reverse().join("");

      return result;
    },
    returnURL(menu) {
      return "https://www.google.com/search?q=" + menu;
    },
    resetSchool() {
      localStorage.clear();
      window.location.reload();
    },
    prevMeal() {
      switch (this.timing) {
        case "":
          this.notExistNextMeal();
          break;
        case "조식":
          this.notExistPrevMeal();
          break;
        case "중식":
          this.morning.length > 0
            ? ((this.meal = this.morning), (this.timing = "조식"))
            : this.notExistPrevMeal();
          break;
        case "석식":
          this.lunch.length > 0
            ? ((this.meal = this.lunch), (this.timing = "중식"))
            : this.notExistPrevMeal();
          break;
      }
    },
    nextMeal() {
      switch (this.timing) {
        case "":
          this.notExistNextMeal();
          break;
        case "조식":
          this.lunch.length > 0
            ? ((this.meal = this.lunch), (this.timing = "중식"))
            : this.notExistNextMeal();
          break;
        case "중식":
          this.dinner.length > 0
            ? ((this.meal = this.dinner), (this.timing = "석식"))
            : this.notExistNextMeal();
          break;
        case "석식":
          this.notExistNextMeal();
          break;
      }
    },
    notExistPrevMeal() {
      this.today = (parseInt(this.today) - 1).toString();
      this.getNewMeal();
    },
    notExistNextMeal() {
      this.today = (parseInt(this.today) + 1).toString();
      this.getNewMeal();
    },
    async getNewMeal() {
      try {
        this.isLoading = true;
        this.initMeals();
        const row = await this.getMealApi();
        this.mealExist = row ? true : false;
        this.assignMeals(row);
        this.meal =
          this.morning.length > 0
            ? this.morning
            : this.lunch.length > 0
            ? this.lunch
            : this.dinner;
        this.timingAssign();
        this.isLoading = false;
      } catch (e) {
        this.mealExist = false;
        this.isLoading = false;
      }
    },
    timingAssign() {
      switch (this.meal) {
        case this.morning:
          this.timing = "조식";
          break;
        case this.lunch:
          this.timing = "중식";
          break;
        case this.dinner:
          this.timing = "석식";
          break;
      }
    },
    async getMealApi() {
      try {
        const {
          data: {
            mealServiceDietInfo: {
              [1]: { row },
            },
          },
        } = await axios.get(
          `https://open.neis.go.kr/hub/mealServiceDietInfo?KEY=${process.env.VUE_APP_NEIS_API_KEY}&Type=json&ATPT_OFCDC_SC_CODE=${this.ATPT_OFCDC_SC_CODE}&SD_SCHUL_CODE=${this.SD_SCHUL_CODE}&MLSV_YMD=${this.today}`
        );

        return row;
      } catch (e) {
        return e;
      }
    },
    assignMeals(row) {
      for (const info of row) {
        const meal = info.MMEAL_SC_NM;
        switch (meal) {
          case "조식":
            this.morning = this.toArray(this.removeBracket(info.DDISH_NM));
            break;
          case "중식":
            this.lunch = this.toArray(this.removeBracket(info.DDISH_NM));
            break;
          case "석식":
            this.dinner = this.toArray(this.removeBracket(info.DDISH_NM));
            break;
        }
      }
    },
    initMeals() {
      this.morning = [];
      this.lunch = [];
      this.dinner = [];
    },
  },
  mounted() {
    this.schoolName = localStorage.getItem("schoolName") ?? "";
    this.getToday();
    this.registerCheck();
    this.getMeal();
  },
};
</script>

<style>
@import "css/app.css";
</style>
