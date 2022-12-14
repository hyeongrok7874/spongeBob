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
        const {
          data: {
            mealServiceDietInfo: {
              [1]: { row },
            },
          },
        } = await axios.get(
          `https://open.neis.go.kr/hub/mealServiceDietInfo?KEY=${process.env.VUE_APP_NEIS_API_KEY}&Type=json&ATPT_OFCDC_SC_CODE=${this.ATPT_OFCDC_SC_CODE}&SD_SCHUL_CODE=${this.SD_SCHUL_CODE}&MLSV_YMD=${this.today}`
        );
        this.mealExist = row ? true : false;
        const now = new Date();
        this.morning = [];
        this.lunch = [];
        this.dinner = [];
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
        case "조식":
          this.notExistPrevMeal();
          break;
        case "중식":
          this.morning
            ? ((this.meal = this.morning), (this.timing = "조식"))
            : this.notExistPrevMeal();
          break;
        case "석식":
          this.lunch
            ? ((this.meal = this.lunch), (this.timing = "중식"))
            : this.notExistPrevMeal();
          break;
      }
    },
    nextMeal() {
      console.log(this.dinner);
      switch (this.timing) {
        case "조식":
          this.lunch.length
            ? ((this.meal = this.lunch), (this.timing = "중식"))
            : this.notExistNextMeal();
          break;
        case "중식":
          this.dinner.length
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
        const {
          data: {
            mealServiceDietInfo: {
              [1]: { row },
            },
          },
        } = await axios.get(
          `https://open.neis.go.kr/hub/mealServiceDietInfo?KEY=${process.env.VUE_APP_NEIS_API_KEY}&Type=json&ATPT_OFCDC_SC_CODE=${this.ATPT_OFCDC_SC_CODE}&SD_SCHUL_CODE=${this.SD_SCHUL_CODE}&MLSV_YMD=${this.today}`
        );
        this.mealExist = row ? true : false;
        this.morning = [];
        this.lunch = [];
        this.dinner = [];
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
        this.meal =
          this.morning.length !== 0
            ? this.morning
            : this.lunch.length !== 0
            ? this.lunch
            : this.dinner;
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
        this.isLoading = false;
      } catch (e) {
        this.mealExist = false;
        this.isLoading = false;
      }
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
@font-face {
  font-family: "CookieRun-Regular";
  src: url("https://cdn.jsdelivr.net/gh/projectnoonnu/noonfonts_2001@1.1/CookieRun-Regular.woff")
    format("woff");
  font-weight: normal;
  font-style: normal;
}

body {
  margin: 0;
  padding: 0;
}

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
  font-family: "CookieRun-Regular", "Noto Sans KR", sans-serif;
}

#app {
  font-family: "CookieRun-Regular", "Noto Sans KR", sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  display: flex;
  justify-content: center;
  background: #fff75f;
  position: relative;
  overflow: hidden;
}

.view {
  width: 500px;
  height: 100vh;
  background: white;
  position: relative;
  z-index: 5;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}

.title {
  font-size: 30px;
}

.school-search {
  width: 100%;
  height: 500px;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  align-items: center;
}

.search-wrap {
  width: 400px;
  height: 110px;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
}

.school-input {
  width: 100%;
  height: 50px;
  padding: 0;
  margin: 0;
  border: 0;
  border-radius: 10px;
  font-size: 20px;
  padding: 0 10px;
  border: 1px black solid;
}

.school-button {
  width: 100%;
  height: 50px;
  padding: 0;
  margin: 0;
  border: 0;
  border-radius: 10px;
  background: black;
  color: white;
  font-size: 20px;
  cursor: pointer;
}

.result-wrap {
  width: 80%;
  height: 300px;
  overflow-y: scroll;
  position: relative;
}

.result-wrap::-webkit-scrollbar {
  width: 10px;
}

.result-wrap::-webkit-scrollbar-thumb {
  background: #cbcbcb;
  border-radius: 7px;
}

.result {
  width: 100%;
  height: 50px;
  background: #40b3d2;
  color: black;
  margin: 10px 0;
  border-radius: 30px;
  padding: 0 20px;
  cursor: pointer;
  display: flex;
  flex-direction: column;
  justify-content: center;
}

.result-name {
  width: 100%;
  white-space: nowrap;
  overflow: hidden;
  font-size: 20px;
  font-weight: bold;
}

.result-address {
  font-size: 15px;
  white-space: nowrap;
  overflow: hidden;
}

.main-wrap {
  width: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: space-between;
}

.school-name {
  font-size: 30px;
}

.today {
  font-size: 20px;
  margin: 10px 0;
}

.timing {
  font-size: 25px;
  margin: 10px 0;
}

.menu-wrap {
  width: 80%;
}

.menu {
  width: 100%;
  height: 50px;
  background: #40b3d2;
  margin: 10px 0;
  display: flex;
  justify-content: center;
  align-items: center;
  font-size: 20px;
  border-radius: 30px;
  text-decoration: none;
  color: black;
}

.change-school {
  width: 100px;
  height: 100px;
  border-radius: 100%;
  border: none;
  background: #eb1d36;
  color: white;
  font-size: 20px;
  cursor: pointer;
  margin-top: 20px;
}

.circle {
  position: absolute;
  border-radius: 100%;
  background: #b2c036;
  z-index: 2;
}

.is-loading {
  width: 100%;
  text-align: center;
  margin-top: 50px;
  font-size: 30px;
}

.school-not-exist {
  height: 300px;
  padding-top: 50px;
  font-size: 30px;
}

.nobab {
  text-align: center;
  font-size: 30px;
  margin: 100px 0;
}

.button {
  position: absolute;
  -webkit-user-select: none;
  -moz-user-select: none;
  -ms-user-select: none;
  user-select: none;
  background: transparent;
  border: none;
  cursor: pointer;
  padding: 10px;
}

.button > img {
  width: 30px;
  height: 30px;
}

.next-button {
  right: 0;
}

.prev-button {
  left: 0;
}

@media (max-width: 500px) {
  .button {
    padding: 7px;
  }
  .button > img {
    width: 25px;
    height: 25px;
  }
}

@media (max-width: 400px) {
  .school-name {
    font-size: 23px;
  }
  .menu {
    font-size: 17px;
  }
  .view {
    width: 100%;
  }
  .search-wrap {
    width: 90%;
  }
  .result-name {
    font-size: 17px;
  }
  .result-address {
    font-size: 12px;
  }
  .button {
    padding: 4px;
  }
  .button > img {
    width: 20px;
    height: 20px;
  }
}
</style>
