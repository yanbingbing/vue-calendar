<template>
<div id="my-calendar" :class="{holidaystyle: isHolidayFestival}">
  <div class="calendar-left">
    <div class="controls">
      <el-select
        class="year-control"
        :value="cYear"
        @change="changeYear"
        placeholder="请选择"
        size="mini"
      >
        <el-option
          v-for="year in years"
          :key="year"
          :label="year + '年'"
          :value="year">
        </el-option>
      </el-select>
      <el-button type="mini" icon="el-icon-arrow-left" @click="prevMonth"></el-button>
      <el-select
        class="month-control"
        :value="cMonth"
        @change="changeMonth"
        placeholder="请选择"
        size="mini"
      >
        <el-option
          v-for="month in months"
          :key="month"
          :label="(month + 1) + '月'"
          :value="month">
        </el-option>
      </el-select>
      <el-button type="mini" icon="el-icon-arrow-right" @click="nextMonth"></el-button>
      <el-select
        class="holiday-control"
        :value="holiday"
        @change="selectHoliday"
        placeholder="假期安排"
        size="mini"
      >
        <el-option
          v-for="holiday in currentHolidayList"
          :key="holiday.startday"
          :label="holiday.name"
          :value="holiday.startday">
        </el-option>
      </el-select>
      <el-button class="returntoday" type="mini" @click="goToday">返回今天</el-button>
    </div>
    <div class="days-table">
      <div class="days-head">
        <span
          class="days-week"
          :class="{weekend: week.id === 0 || week.id === 6}"
          :key="week.id" v-for="week in weeks"
        >
          {{week.name}}
        </span>
      </div>
      <div class="days" :class="{rows6: currentDays.length > 35}">
        <div
          class="rili-cell"
          :key="item.fullDate"
          v-for="item in currentDays"
          :class="{
            festival: item.festival,
            rest: item.rest,
            work: item.work,
            weekend: item.weekend,
            other: item.other,
            today: item.fulldate === today,
            selected: item.fulldate === selected
          }"
          @click="changeDate(item)"
        >
          <span class="holiday-sign" v-if="item.rest">休</span>
          <span class="holiday-sign" v-else-if="item.work">班</span>
          <span class="rili-number">{{item.date}}</span>
          <span class="rili-almanac">{{item.almanac}}</span>
        </div>
      </div>
    </div>
  </div>
  <div class="calendar-right">
    <div class="format-date">{{selectedFormat}}</div>
    <div class="big-date">{{cDate}}</div>
    <div class="lunar-detail">
      <span>{{lunarDetail.lMonth}}月{{lunarDetail.lDate}}</span>
      <span>{{lunarDetail.gzYear}}年 【{{lunarDetail.animal}}年】</span>
      <span>{{lunarDetail.gzMonth}}月 {{lunarDetail.gzDate}}日</span>
    </div>
    <div class="almanac-detail">
      <div class="almanac-suit">
        <i>宜</i>
        <p :key="suit" v-for="suit in almanacDetail.suit.split('.', 5)" v-if="suit">
          {{suit}}
        </p>
      </div>
      <div class="almanac-avoid">
        <i>忌</i>
        <p :key="avoid" v-for="avoid in almanacDetail.avoid.split('.', 5)" v-if="avoid">
           {{avoid}}
        </p>
      </div>
    </div>
  </div>
</div>
</template>

<script>
import Vue from 'vue';
import $ from 'jquery';
import lunar from './lunar';

function queryExternalData(query, cb) {
  $.getJSON('http://test.yanbingbing.com/rili.php?cb=?', {
    query,
  }, cb);
}

function getExternalData(y, m) {
  const query = `${y}年${m + 1}月`;
  const data = localStorage.getItem(`calendarData:${query}`);
  if (data) {
    try {
      return Promise.resolve(JSON.parse(data));
    } catch (e) {
      // empty
    }
  }

  return new Promise((resolve) => {
    queryExternalData(query, (ret) => {
      localStorage.setItem(`calendarData:${query}`, JSON.stringify(ret));
      resolve(ret);
    });
  });
}

function pad(n) {
  let str = String(n);
  if (str.length < 2) {
    str = `0${str}`;
  }
  return str;
}

const almanacData = {};
const holidayStatus = {};
const holidayFestival = {};
const holidayList = {};
const prepared = {};
const WEEKS = [
  { name: '日', id: 0 },
  { name: '一', id: 1 },
  { name: '二', id: 2 },
  { name: '三', id: 3 },
  { name: '四', id: 4 },
  { name: '五', id: 5 },
  { name: '六', id: 6 },
];

function getDateData(y, m, d, other) {
  const dx = new Date(y, m, d);
  const day = dx.getDay();
  const fulldate = `${y}-${m + 1}-${d}`;
  const lunarRes = lunar(dx);
  let almanac = lunarRes.term || lunarRes.lDate;
  const festival = lunarRes.festival();
  if (festival.length > 0) {
    almanac = festival[0].desc;
  }
  return {
    fulldate,
    date: d,
    month: m,
    year: y,
    almanac,
    work: (fulldate in holidayStatus) && holidayStatus[fulldate] === '2',
    rest: (fulldate in holidayStatus) && holidayStatus[fulldate] === '1',
    festival: lunarRes.term || festival.length > 0,
    day,
    other,
    weekend: day === 0 || day === 6,
  };
}

function prepareExternalData(year, month, callback) {
  const fullmonth = `${year}-${month}`;
  if (prepared[fullmonth]) {
    return;
  }
  prepared[fullmonth] = true;
  getExternalData(year, month).then((json) => {
    if (!json || !json.data || !json.data[0]) return;
    if (Array.isArray(json.data[0].almanac)) {
      json.data[0].almanac.forEach((item) => {
        Vue.set(almanacData, item.date, item);
      });
    }
    if (json.data[0].holiday) {
      let holidays = json.data[0].holiday;
      if (!Array.isArray(holidays)) {
        holidays = [holidays];
      }
      holidays.forEach((item) => {
        Vue.set(holidayFestival, item.festival, item);
        item.list.forEach((k) => {
          holidayStatus[k.date] = k.status;
        });
      });
    }
    if (json.data[0].holidaylist) {
      Vue.set(holidayList, year, json.data[0].holidaylist);
    }

    callback(fullmonth);
  });
}

function fillExternal(fullmonth, target) {
  if (fullmonth in target) {
    target[fullmonth].forEach((item) => {
      if (item.fulldate in holidayStatus) {
        if (holidayStatus[item.fulldate] === '2') {
          Vue.set(item, 'work', true);
        } else if (holidayStatus[item.fulldate] === '1') {
          Vue.set(item, 'rest', true);
        }
      }
      if (item.fulldate in holidayFestival) {
        const festival = holidayFestival[item.fulldate];
        Vue.set(item, 'festival', true);
        Vue.set(item, 'almanac', festival.name);
      }
    });
  }
}

/**
 * get month data
 * @param {number} year 2018
 * @param {number} month 0 - 11
 */
function getMonthData(year, month, startDay) {
  const d = new Date(year, month, 1);
  const day = d.getDay();
  const diffWeek = day - (startDay % 7);
  const data = [];
  let datepos = (diffWeek < 0 ? -6 : 1) - diffWeek;
  let realDate;
  let realMonth;
  let realYear;
  let i = 0;
  do {
    d.setDate(datepos);
    realYear = d.getFullYear();
    realDate = d.getDate();
    realMonth = d.getMonth();
    if (realMonth === month + 1 && i % 7 === 0) {
      break;
    }

    data.push(getDateData(
      realYear, realMonth, realDate,
      realMonth !== month,
    ));

    datepos = realDate + 1;
    i += 1;
  } while (i < 42);

  return data;
}
export default {
  name: 'my-calendar',
  data() {
    const years = [];
    for (let year = 1900; year <= 2050; year += 1) {
      years.push(year);
    }
    const months = [];
    for (let month = 0; month < 12; month += 1) {
      months.push(month);
    }
    const startDay = 1;
    const today = new Date();
    const days = {};
    const cYear = today.getFullYear();
    const cMonth = today.getMonth();
    const cDate = today.getDate();
    const weeks = WEEKS.slice();
    if (startDay > 0) {
      const inserts = weeks.splice(0, startDay);
      weeks.splice(weeks.length, 0, ...inserts);
    }
    days[`${cYear}-${cMonth}`] = getMonthData(cYear, cMonth, startDay);
    prepareExternalData(cYear, cMonth, (f) => {
      fillExternal(f, days);
    });
    return {
      startDay,
      cYear,
      cMonth,
      cDate,
      today: `${cYear}-${cMonth + 1}-${cDate}`,
      years,
      months,
      days,
      weeks,
      almanacData,
      holidayFestival,
      holidayList,
    };
  },
  computed: {
    fullMonth() {
      return `${this.cYear}-${this.cMonth}`;
    },
    currentDays() {
      return this.days[this.fullMonth];
    },
    currentHolidayList() {
      return this.holidayList[this.cYear];
    },
    selected() {
      return `${this.cYear}-${this.cMonth + 1}-${this.cDate}`;
    },
    selectedFormat() {
      const day = (new Date(this.cYear, this.cMonth, this.cDate)).getDay();
      return `${this.cYear}-${pad(this.cMonth + 1)}-${pad(this.cDate)} 星期${WEEKS[day].name}`;
    },
    lunarDetail() {
      const dx = new Date(this.cYear, this.cMonth, this.cDate);
      const ret = lunar(dx);
      return {
        gzDate: ret.gzDate,
        gzMonth: ret.gzMonth,
        gzYear: ret.gzYear,
        animal: ret.animal,
        lDate: ret.lDate,
        lMonth: ret.lMonth,
      };
    },
    almanacDetail() {
      return this.almanacData[this.selected] || { avoid: '', suit: '' };
    },
    isHolidayFestival() {
      return (this.selected in this.holidayFestival);
    },
    holiday() {
      return this.isHolidayFestival ? this.selected : '';
    },
  },
  watch: {
    startDay(val) {
      prepareExternalData(this.cYear, this.cMonth, (f) => {
        fillExternal(f, this.days);
      });
      const weeks = WEEKS.slice();
      if (val > 0) {
        const inserts = weeks.splice(0, val);
        weeks.splice(weeks.length, 0, ...inserts);
      }
      this.weeks = weeks;
      this.days = {
        [this.fullMonth]: getMonthData(this.cYear, this.cMonth, val),
      };
    },
    cYear(val) {
      const fullMonth = `${val}-${this.cMonth}`;
      if (!(fullMonth in this.days)) {
        this.days[fullMonth] = getMonthData(val, this.cMonth, this.startDay);
      }
      prepareExternalData(val, this.cMonth, (f) => {
        fillExternal(f, this.days);
      });
    },
    cMonth(val) {
      const fullMonth = `${this.cYear}-${val}`;
      if (!(fullMonth in this.days)) {
        this.days[fullMonth] = getMonthData(this.cYear, val, this.startDay);
      }
      prepareExternalData(this.cYear, val, (f) => {
        fillExternal(f, this.days);
      });
    },
  },
  methods: {
    changeDate(item) {
      this.cYear = item.year;
      this.cMonth = item.month;
      this.cDate = item.date;
    },
    selectHoliday(fulldate) {
      const [y, m, d] = fulldate.split('-').map(k => parseInt(k, 10));
      const x = new Date(y, m - 1, d);
      this.cYear = x.getFullYear();
      this.cMonth = x.getMonth();
      this.cDate = x.getDate();
    },
    changeYear(year) {
      const d = new Date(year, this.cMonth, this.cDate);
      if (d.getDate() !== this.cDate) {
        d.setDate(0);
      }
      this.cYear = d.getFullYear();
      this.cMonth = d.getMonth();
      this.cDate = d.getDate();
    },
    changeMonth(month) {
      const d = new Date(this.cYear, month, this.cDate);
      if (d.getDate() !== this.cDate) {
        d.setDate(0);
      }
      this.cYear = d.getFullYear();
      this.cMonth = d.getMonth();
      this.cDate = d.getDate();
    },
    prevMonth() {
      const d = new Date(this.cYear, this.cMonth - 1, this.cDate);
      if (d.getDate() !== this.cDate) {
        d.setDate(0);
      }
      this.cYear = d.getFullYear();
      this.cMonth = d.getMonth();
      this.cDate = d.getDate();
    },
    nextMonth() {
      const d = new Date(this.cYear, this.cMonth + 1, this.cDate);
      if (d.getDate() !== this.cDate) {
        d.setDate(0);
      }
      this.cYear = d.getFullYear();
      this.cMonth = d.getMonth();
      this.cDate = d.getDate();
    },
    goToday() {
      const today = new Date();
      this.cYear = today.getFullYear();
      this.cMonth = today.getMonth();
      this.cDate = today.getDate();
    },
  },
};
</script>
