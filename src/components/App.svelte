<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import { csv } from 'd3-fetch';
  import { feature } from 'topojson-client';

  let svg1;
  let svg2;

  let dataByStateAndDate = {};
  let dates = [];
  let selectedDateIndex = 0;
  let selectedDateString = '';
  let maxCases = 0;
  let lastColorByState = {};

  let maxDeaths = 0; // 数据集中的最大死亡人数
  let maxMortalityRate = 0; // 数据集中的最大死亡率 

  let us = null;
  let projection = null;

  //----------加载数据区域 ----------//

  onMount(async () => {
    us = await d3.json('states-10m.json');
    const data = await csv('daily_data.csv');
    const monthData = await csv('monthly_data.csv');

    data.forEach(d => {
      if (!dataByStateAndDate[d.date]) {
        dataByStateAndDate[d.date] = {};
      }
      // 确保将字符串转换为数字
      const cases = +d.cases;
      const deaths = +d.deaths;
      const mortalityRate = +d.mortality_rate;

      // 存储每个日期下每个州的详细信息
      dataByStateAndDate[d.date][d.state] = { cases, deaths, mortalityRate };

      // 更新最大值
      if (cases > maxCases) maxCases = cases;
      if (deaths > maxDeaths) maxDeaths = deaths;
      if (mortalityRate > maxMortalityRate) maxMortalityRate = mortalityRate;
    });

    dates = Object.keys(dataByStateAndDate).sort((a, b) => new Date(a) - new Date(b));
    selectedDateString = dates[selectedDateIndex];
    renderMap(us);
  });

//----------function 区域 ----------//

  // 绘制地图和染色的代码 <<---------
  function renderMap(us) {
    const width = svg1.clientWidth; // 获取SVG的宽度
    const height = svg1.clientHeight; // 获取SVG的高度

    // 根据SVG的大小动态计算缩放比例
    const scale = Math.min(width, height) * 2.2;
    projection = d3.geoAlbersUsa().scale(scale).translate([width / 2, height / 2]);
    const pathGenerator = d3.geoPath().projection(projection);
    const states = feature(us, us.objects.states).features;

    d3.select(svg1)
      .selectAll('.state')
      .data(states)
      .enter().append('path')
        .attr('class', 'state')
        .attr('d', pathGenerator)
        .attr('fill', '#F9EBEA')
        .attr('stroke', '#fff')
        .attr('stroke-width', '1');

    updateMapColors(); // 根据初始日期更新颜色
    updateCircles(); // 绘制圆圈
  }

  function updateMapColors() {
    const currentDate = dates[selectedDateIndex];
    const casesByState = dataByStateAndDate[currentDate] || {};

    d3.select(svg1).selectAll('.state')
      .attr('fill', function(d) {
        const stateName = d.properties.name;
        // 此处修改为从对象中提取cases
        const stateData = casesByState[stateName];
        if (stateData && stateData.cases !== undefined) {
          const color = calculateColor(stateData.cases);
          lastColorByState[stateName] = color; // 更新该州的颜色
          return color;
        }
        return lastColorByState[stateName] || '#F9EBEA'; // 没有新数据时保持颜色不变
      });
  }

  function calculateColor(cases) {
    const scale = d3.scaleLinear().domain([0, maxCases]).range(['#F9EBEA', '#641E16']);
    return scale(cases);
  }

  // 绘制地图上的小圆的代码 <<---------
  function updateCircles() {
    // 确保移除旧的圆圈
    d3.select(svg1).selectAll("circle").remove();

    const currentDateData = dataByStateAndDate[selectedDateString] || {};
    const states = feature(us, us.objects.states).features;

    states.forEach(state => {
      const center = d3.geoPath().projection(projection).centroid(state);
      const stateData = currentDateData[state.properties.name];

      // 如果当前日期的数据存在则使用它，否则使用默认值
      const deaths = stateData ? stateData.deaths : 0; // 默认死亡数为0
      const mortalityRate = stateData ? stateData.mortalityRate : 0; // 默认死亡率为0

      // 使用比例确定圆的半径和颜色
      const radius = deaths ? d3.scaleSqrt().domain([0, maxDeaths]).range([0, 30])(deaths) : 1; // 如果没有死亡数，使用默认大小1
      const color = mortalityRate ? d3.scaleLinear().domain([0, 6]).range(["lightgrey", "black"])(mortalityRate) : "lightgrey"; // 如果没有死亡率，使用默认颜色lightgrey

      d3.select(svg1).append("circle")
        .attr("cx", center[0])
        .attr("cy", center[1])
        .attr("r", radius)
        .attr("fill", color)
        .attr("fill-opacity", 0.5) // 设置半透明
        .attr("stroke", "none"); // 无边框
    });
  }

  // 第一个图的播放功能代码 <<---------
  let playing = false; // 用于追踪播放状态
  let intervalId; // 用于存储定时器ID

  // 播放或暂停功能
  function togglePlay() {
    if (!playing) {
      // 如果当前不在播放就开始播放
      playing = true;
      intervalId = setInterval(() => {
        if (selectedDateIndex < dates.length - 1) {
          selectedDateIndex += 1; // 前进到下一个日期
        } else {
          resetPlay(); // 如果到达末尾则重置
        }
      }, 15); // 更新频次
    } else {
      // 如果当前在播放则暂停
      stopPlaying();
    }
  }

  // 停止播放功能
  function stopPlaying() {
    clearInterval(intervalId);
    playing = false;
  }

  // 重置播放功能
  function resetPlay() {
    clearInterval(intervalId);
    selectedDateIndex = 0; // 重置到最开始的状态
    playing = false; // 更新播放状态
  }

  // 进度条的更新设置 <<---------
  $: selectedDateString = dates[selectedDateIndex]; // 先更新selectedDateString

  $: if (selectedDateString) {
      updateMapColors(); // 根据最新的selectedDateString更新颜色
      updateCircles();   // 更新圆
  }
</script>


<!---------- HTML的构成部分 ---------->
<div class="container">


  <div class="text-box">
    <h1>Title should be here, but still undecided</h1>
    <h2>Sub-title or additional information</h2>
  </div>


  <div class="visualization">
    <svg bind:this={svg1} width="100%" height="98%"></svg>
  </div>

  <!-- 时间拖动条和日期 -->
  <div class="controls">
    <button on:click={togglePlay}>{playing ? '⏸️ Pause' : '▶️ Play'}</button>
    <p>{selectedDateString}</p>
    <input type="range" bind:value={selectedDateIndex} min="0" max={dates.length - 1}>
  </div>

  <div class="text-box">
    <h2>Explain some trends and details in the map to users.</h2>
    <h2>Then lead to the next visualization, using a line chart to observe the changing trend of covid.</h2>
  </div>

  <div class="visualization">
    <h2>Here will be an interactive line chart to observe the changing trends of each state's covid data over time.</h2>
    <svg bind:this={svg2} width="100%" height="98%"></svg>
  </div>

  <div class="text-box">
    <h2>Here we describe some of the content of the above line chart, and then lead to our next visualization below, which is a line chart about the rate of change of mortality.</h2>
    <h2>the text box background can be edit to have a better look</h2>
  </div>

  <div class="visualization">
    <h2>Here will be a line chart here to show how the change rate of mortality changes over time to show the changing trend of the harm of COVID19.</h2>
    <!-- 可视化组件3将放置在这里 -->
  </div>

  <div class="text-box">
    <h1>Draw the conclusion here.</h1>
  </div>
</div>

<style>

  :global(body) {
    background-color: #EAF2F8; /* 背景颜色 */
  }

  .container {
    display: flex;
    flex-direction: column;
    align-items: center; /* 横向居中 */
    height: auto; /* 容器高度根据内容自适应 */
    padding: 1rem;
    gap: 2rem; /* 组件之间的间隔 */
  }

  .visualization {
    width: 70vw; /* 页面宽度的70% */
    height: 63vh; /* 视口高度的50% */
    border: 1px solid #ccc;
    border-radius: 5px;
    padding: 1rem;
    margin-bottom: 2rem; /* 为滚动提供空间 */
    box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
    overflow: auto; /* 如果内容超出，则允许滚动 */
    background-color: white;
  }

  .text-box {
    width: 70%; /* 与可视化组件宽度一致 */
    text-align: center; /* 文本居中 */
  }

  .controls {
  display: flex;
  flex-direction: row; /* 使内容水平排列 */
  align-items: center; /* 横向居中对齐 */
  padding: 1rem;
  width: 60%; /* 调整为所需的百分比 */
  margin: 0 auto; /* 水平居中 */
  margin-top: -4%; /* 向上移动 */
  }

  .controls input[type="range"], .controls p {
    flex-grow: 1; /* 让进度条和日期标签填充剩余空间 */
  }

  .controls button {
  margin-right: 3%; /* bottom和日期的间距 */
  }

  /* 媒体查询，用于小屏幕的样式调整 */
  @media (max-width: 600px) {
    .visualization {
      width: 90%; /* 小屏幕上的宽度调整 */
    }
  }
</style>
