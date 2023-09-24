<template>
  <div class="charts-container">
    <div class="top-charts">
      <div ref="chart1" class="chart" style="width: 600px; height: 400px;"></div>
      <div ref="chart2" class="chart" style="width: 600px; height: 400px;"></div>
    </div>
    <div ref="chart3" class="chart large-chart" style="width: 1300px; height: 400px;"></div>
    <div class="search-section">
  <!-- 使用 el-autocomplete 替换普通的 input -->
  <el-autocomplete
    v-model="city"
    :fetch-suggestions="querySearchCitiesAsync"
    placeholder="请输入城市"
    @select="handleCitySelect"
    class="search-input"
  >
    <template slot-scope="{ item }">
      {{ item }}
    </template>
  </el-autocomplete>

  <!-- 第二个输入框: 景点 -->
  <el-autocomplete
    v-model="name"
    :fetch-suggestions="querySearchAttractionsAsync"
    placeholder="请输入景点名称"
    @select="handleAttractionSelect"
    class="search-input"
  >
    <template slot-scope="{ item }">
      {{ item }}
    </template>
  </el-autocomplete>
  
  <!-- 使用 el-button 替换普通的 button -->
  <el-button type="primary" icon="el-icon-search" @click="handleSearch">搜索</el-button>
</div>
<div class="result-section" v-if="searchResult">
  <!-- Display image -->
  <div class="result-image">
    <img :src="searchResult.viewUrl" alt="Search Result Image">
  </div>
  <!-- Display other information -->
  <div class="result-info">
    <h3>{{ searchResult.name }}</h3>
    <p><strong>City:</strong> {{ searchResult.city }}</p>
    <p><strong>Heat Score:</strong> {{ searchResult.heatScore }}</p>
    <p><strong>Rank:</strong> {{ searchResult.rank }}</p>
    <p><strong>Hot Tag:</strong> {{ searchResult.hotTag }}</p>
  </div>
</div>

    
  </div>

</template>


<script>
import * as echarts from 'echarts';
import axios from 'axios';

export default {
  name: 'CityRankings',
  data() {
    return {
      chart1: null,
      chart2: null,
      chart3: null,
      city: '',   // 添加 city 字段
      name: '',   // 添加 name 字段
      cityList: [],
      attractionList: [],
      searchResult: null  // 添加 searchResult 字段

    };
  },
  mounted() {
    this.chart1 = echarts.init(this.$refs.chart1);
    this.chart2 = echarts.init(this.$refs.chart2);
    this.chart3 = echarts.init(this.$refs.chart3);
    this.fetchData();
    this.fetchAllAverageHeatScoreData();
    this.fetchCityList();
  },
  methods: {
    fetchCityList() {
      axios.get('http://localhost:8080/CityList')
        .then(response => {
          this.cityList = response.data;
        })
        .catch(error => console.error('Error fetching city list:', error));
    },
    querySearchCitiesAsync(queryString, cb) {
      console.log("queryString: ", queryString); // 输出搜索的查询字符串
      const results = this.cityList.filter(city => city.toLowerCase().includes(queryString.toLowerCase()));
      console.log("filtered results: ", results); // 输出过滤后的结果
      cb(results);
    },
    handleCitySelect(item) {
    this.city = item;
    this.fetchAttractionList();
  },
  fetchAttractionList() {
    axios.get(`http://localhost:8080/NameList?city=${this.city}`)
      .then(response => {
        this.attractionList = response.data;
      })
      .catch(error => console.error('Error fetching attraction list:', error));
  },
  querySearchAttractionsAsync(queryString, cb) {
    // 首先，检查 queryString 是否定义
    if (typeof queryString !== 'string') {
        return cb([]); // 返回空数组，如果queryString是undefined或不是字符串
    }

    const results = this.attractionList.filter(attraction => {
        // 确保 attraction 是一个字符串
        return typeof attraction === 'string' && attraction.toLowerCase().includes(queryString.toLowerCase());
    });
    cb(results);
},handleAttractionSelect(item) {
    this.name = item;
    // 这里你可以添加处理选定景点的其他逻辑
  }
,
    handleSearch() {
      axios.get(`http://localhost:8080/findByCityAndName?city=${this.city}&name=${this.name}`)
        .then(response => {
          if (response.data && response.data.length > 0) {
            this.searchResult = response.data[0];
          } else {
            this.searchResult = null;
          }
        })
        .catch(error => console.error('Error fetching search data:', error));
    },
    fetchData() {
      axios.get('http://localhost:8080/analyzeAverageRank')
        .then(response => {
          this.updateChart(this.chart1, response.data, '评分前十');
        })
        .catch(error => console.error('Error fetching rank data:', error));

      axios.get('http://localhost:8080/analyzeAverageHeatScore')
        .then(response => {
          this.updateChart(this.chart2, response.data, '热度前十');
        })
        .catch(error => console.error('Error fetching heat score data:', error));
    },
    fetchAllAverageHeatScoreData() {
      axios.get('http://localhost:8080/analyzeAllAverageHeatScore')
        .then(response => {
          this.updateLargeBarChart(this.chart3, response.data);
        })
        .catch(error => console.error('Error fetching all average heat score data:', error));
    },
    updateChart(chart, data, title) {
      const option = {
        title: {
          text: title,
        },
        tooltip: {
          trigger: 'axis',
        },
        xAxis: {
          type: 'category',
          data: data.map(item => item.city),
          axisLabel: {
            interval: 0,
            rotate: 30,
          },
        },
        yAxis: {
          type: 'value',
          min: Math.min(...data.map(item => item.value)) - 0.05,
        },
        series: [
          {
            data: data.map(item => item.value),
            type: 'line',
            label: {
              show: true,
              position: 'top',
            },
          },
        ],
      };

      chart.setOption(option);
    },
    updateLargeBarChart(chart, data) {
  const option = {
    title: {
      text: data.length + ' Data',
      left: 10
    },
    toolbox: {
      feature: {
        dataZoom: {
          yAxisIndex: false
        },
        saveAsImage: {
          pixelRatio: 2
        }
      }
    },
    tooltip: {
      trigger: 'axis',
      axisPointer: {
        type: 'shadow'
      }
    },
    grid: {
      bottom: 90
    },
    dataZoom: [
      {
        type: 'inside'
      },
      {
        type: 'slider'
      }
    ],
    xAxis: {
      data: data.map(item => item.city), // Use item.city here
      silent: false,
      splitLine: {
        show: false
      },
      splitArea: {
        show: false
      }
    },
    yAxis: {
      splitArea: {
        show: false
      }
    },
    series: [
      {
        type: 'bar',
        data: data.map(item => item.value), // Use item.value here
        large: true
      }
    ]
  };

  chart.setOption(option);
}

  },
  
};
</script>

<style>
body, html {
  margin: 0;
  padding: 0;
  width: 100%;
  height: 100%;
}

body::before {
  content: "";
  position: fixed; /* Use fixed position so it covers the entire screen */
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-image: url('https://img.tukuppt.com/bg_grid/00/04/90/It4mH8sMep.jpg!/fh/350');
  background-size: cover;
  background-repeat: no-repeat;
  z-index: -1; /* Make sure it stays below the content */
  opacity: 0.5; /* Adjust this value for background opacity */
}

.charts-container {
  position: relative;
  display: flex;
  flex-direction: column; /* stack the charts vertically */
  align-items: center;
  justify-content: center;
  height: 100%;
}

.top-charts {
  display: flex;
  margin-bottom: 20px; /* Add some space between the top charts and the large chart below */
}

.chart {
  margin-right: 20px;
  position: relative;
  background-color: rgba(255, 255, 255, 0.3); /* Optional: give a slight white background to charts */
  z-index: 1;
}

.large-chart {
  margin-right: 0; /* No margin on the right for the large chart */
}


/* 省略原来的样式 */

/* 添加搜索部分的样式 */
.search-section {
  display: flex;
    align-items: center;
    gap: 10px; /* 这个是元素之间的间隙，你可以根据需要调整 */
}

.result-section {
  display: flex;
  align-items: stretch; /* Make sure the children of the result section stretch to the same height */
  justify-content: center;
  margin-top: 20px;
  background-color: rgba(255, 255, 255, 0.3); /* Optional background color for the result section */
  border-radius: 10px; /* Optional border radius for the result section */
}

.result-image {
  display: flex;
  align-items: center; /* Center the image vertically */
  justify-content: center; /* Center the image horizontally */
  padding: 10px; /* Add some padding around the image */
  flex:1; /* Distribute space evenly */
}

.result-image img {
  max-height: 100%; /* Limit the maximum height of the image */
  object-fit: contain; /* Make sure the image is contained within its box */
}

.result-info {
  display: flex;
  flex-direction: column;
  justify-content: center; /* Center the text content vertically */
  padding: 10px; /* Add some padding around the text content */
  flex: 2; /* Distribute space, allow info section to take more space */
  min-width: 200px; /* Set a minimum width for the info section */
}

.search-input {
  margin-right: 10px;
  max-width: 200px; /* 你可以根据需要调整这个值 */
}

</style>

