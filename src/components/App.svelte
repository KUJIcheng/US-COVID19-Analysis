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

  let lastColorByState = {};
  let maxCases = 0; // 数据集中最大的case数
  let maxDeaths = 0; // 数据集中的最大死亡人数
  let maxMortalityRate = 0; // 数据集中的最大死亡率

  let us = null;
  let projection = null;

  let selectedState = 'California';
  let states = ['Alabama', 'Alaska', 'American Samoa', 'Arizona', 
  'Arkansas', 'California', 'Colorado', 'Connecticut', 'Delaware', 
  'District of Columbia', 'Florida', 'Georgia', 'Guam', 'Hawaii', 
  'Idaho', 'Illinois', 'Indiana', 'Iowa', 'Kansas', 'Kentucky', 
  'Louisiana', 'Maine', 'Maryland', 'Massachusetts', 'Michigan', 
  'Minnesota', 'Mississippi', 'Missouri', 'Montana', 'Nebraska', 
  'Nevada', 'New Hampshire', 'New Jersey', 'New Mexico', 'New York', 
  'North Carolina', 'North Dakota', 'Northern Mariana Islands', 'Ohio', 
  'Oklahoma', 'Oregon', 'Pennsylvania', 'Puerto Rico', 'Rhode Island', 
  'South Carolina', 'South Dakota', 'Tennessee', 'Texas', 'Utah', 'Vermont', 
  'Virgin Islands', 'Virginia', 'Washington', 'West Virginia', 'Wisconsin', 'Wyoming'];

  //----------加载数据区域 ----------//

  onMount(async () => {
    us = await d3.json('states-10m.json');
    const data = await csv('daily_data_complete.csv');
    // const monthData = await csv('monthly_data.csv');

    // 读取每日的数据以及处理部分 <<---------
    data.forEach(d => {
      if (!dataByStateAndDate[d.date]) {
        dataByStateAndDate[d.date] = {};
      }
      // 确保将字符串转换为数字
      const cases = +d.cases;
      const deaths = +d.deaths;
      const mortalityRate = +d.mortality_rate;
      const casesGrowthRate = +d.cases_growth_rate;
      const deathsGrowthRate = +d.deaths_growth_rate;
      const mortalityRateChange = +d.mortality_rate_change;

      // 存储每个日期下每个州的详细信息
      dataByStateAndDate[d.date][d.state] = {
        cases, 
        deaths, 
        mortalityRate,
        casesGrowthRate,
        deathsGrowthRate,
        mortalityRateChange
      };

      // 更新最大值
      if (cases > maxCases) maxCases = cases;
      if (deaths > maxDeaths) maxDeaths = deaths;
      if (mortalityRate > maxMortalityRate) maxMortalityRate = mortalityRate;
    });

    dates = Object.keys(dataByStateAndDate).sort((a, b) => new Date(a) - new Date(b));
    selectedDateString = dates[selectedDateIndex];

    renderMap(us);
    renderLineChart(selectedState);
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
      const radius = deaths ? d3.scaleSqrt().domain([0, maxDeaths]).range([0, svg1.clientHeight * 0.06])(deaths) : 0; // 如果没有死亡数，使用默认大小0
      const color = mortalityRate ? d3.scaleLinear().domain([0, 0.06]).range(["lightgrey", "black"])(mortalityRate) : "lightgrey"; // 如果没有死亡率，使用默认颜色lightgrey

      d3.select(svg1).append("circle")
        .attr("cx", center[0])
        .attr("cy", center[1])
        .attr("r", radius)
        .attr("fill", color)
        .attr("fill-opacity", 0.75) // 设置半透明
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

  //绘制折线图的代码 <<---------
  function renderLineChart(selectedState) {
    const margin = { top: 20, right: 80, bottom: 30, left: 80 },
          width = svg2.clientWidth - margin.left - margin.right,
          height = svg2.clientHeight - margin.top - margin.bottom;

    // 清空之前的绘图
    d3.select(svg2).selectAll('*').remove();

    const svg = d3.select(svg2)
                  .attr("viewBox", `0 0 ${width + margin.left + margin.right} ${height + margin.top + margin.bottom}`)
                  .append("g")
                  .attr("transform", `translate(${margin.left},${margin.top})`);

    // 数据处理，选择每个月23号的数据
    const filteredData = Object.entries(dataByStateAndDate)
      .filter(([date]) => {
        const day = new Date(date).getDate();
        return day === 23; // 仅选择每月23号的数据
      })
      .map(([date, statesData]) => {
        const { cases, mortalityRate } = statesData[selectedState] || { cases: 0, mortalityRate: 0 };
        return {
          date: d3.timeParse("%Y-%m-%d")(date),
          cases,
          mortalityRate
        };
      });

    // 创建比例尺
    const xScale = d3.scaleTime().range([0, width]).domain(d3.extent(filteredData, d => d.date));
    const yScaleLeft = d3.scaleLinear().range([height, 0]).domain([0, d3.max(filteredData, d => d.cases)]);
    const yScaleRight = d3.scaleLinear().range([height, 0]).domain([0, 0.1]);

    // 定义左侧y轴和右侧y轴
    const yAxisLeft = d3.axisLeft(yScaleLeft);
    const yAxisRight = d3.axisRight(yScaleRight).tickFormat(d3.format(".0%"));

    // 绘制轴
    svg.append("g")
      .attr("transform", `translate(0, ${height})`)
      .call(d3.axisBottom(xScale));

    svg.append("g")
      .call(yAxisLeft);

    svg.append("g")
      .attr("transform", `translate(${width}, 0)`)
      .call(yAxisRight);

    // 线条生成器 for cases
    const lineCases = d3.line()
                    .defined(d => !isNaN(d.cases))
                    .curve(d3.curveMonotoneX)
                    .x(d => xScale(d.date))
                    .y(d => yScaleLeft(d.cases));

    // 线条生成器 for mortalityRate
    const lineMortalityRate = d3.line()
                                .defined(d => !isNaN(d.mortalityRate))
                                .curve(d3.curveMonotoneX)
                                .x(d => xScale(d.date))
                                .y(d => yScaleRight(d.mortalityRate));

    // 绘制cases线条
    svg.append("path")
      .data([filteredData])
      .attr("class", "line cases-line")
      .style("stroke", "#CB4335")
      .style("fill", "none")
      .style("stroke-width", "3px")
      .attr("d", lineCases);

    // 绘制mortalityRate线条
    svg.append("path")
      .data([filteredData])
      .attr("class", "line mortality-rate-line")
      .style("stroke", "#5B2C6F")
      .style("fill", "none")
      .style("stroke-width", "3px")
      .attr("d", lineMortalityRate);

    // 做灰色的线的代码
    // 添加一个透明的矩形覆盖整个SVG来捕捉鼠标事件
    svg.append("rect")
      .attr("width", width)
      .attr("height", height)
      .attr("fill", "none")
      .attr("pointer-events", "all")
      .on("mousemove", mousemove)
      .on("mouseout", mouseout);

    // 添加垂直线并调整粗细
    const focusLine = svg.append("line")
      .style("stroke", "#5D6D7E")
      .style("stroke-width", "2px") // 线的粗细
      .style("opacity", 0);

    // 调整空心圆的大小
    const focusCircleCases = svg.append("circle")
      .style("fill", "none")
      .style("stroke", "#CB4335")
      .attr("r", 7) // 圆的半径
      .style("opacity", 0);

    const focusCircleMortalityRate = svg.append("circle")
      .style("fill", "none")
      .style("stroke", "#5B2C6F")
      .attr("r", 7) // 圆的半径
      .style("opacity", 0);
    
    // 鼠标在第二个图移动的时候的交互行为代码在这里 <<---------
    function mousemove(event) {
      const mouseX = d3.pointer(event, this)[0]; // 获取鼠标在SVG内的x坐标
      const date = xScale.invert(mouseX); // 将x坐标转换回日期
        
      // 找到最接近鼠标位置的日期对应的数据点
      const index = d3.bisector(d => d.date).left(filteredData, date, 1);
      const a = filteredData[index - 1];
      const b = filteredData[index];
      const d = b && (date - a.date > b.date - date) ? b : a;

      focusLine.attr("x1", xScale(d.date))
                .attr("x2", xScale(d.date))
                .attr("y1", 0)
                .attr("y2", height)
                .style("opacity", 1);

      focusCircleCases.attr("cx", xScale(d.date))
                      .attr("cy", yScaleLeft(d.cases))
                      .style("opacity", 1);

      focusCircleMortalityRate.attr("cx", xScale(d.date))
                              .attr("cy", yScaleRight(d.mortalityRate))
                              .style("opacity", 1);
      
      focusLine.style("opacity", 1);
      focusCircleCases.style("opacity", 1);
      focusCircleMortalityRate.style("opacity", 1);

      // 根据找到的数据点更新信息框内容和位置
      const formatDate = d3.timeFormat("%Y-%m");
      const infoContent = [
        `Date: ${formatDate(d.date)}`,
        `Cases: ${d.cases}`,
        `Mortality Rate: ${(d.mortalityRate * 100).toFixed(2)}%`
      ]; // 使用数组来组织每一行的内容
      infoText.text(""); // 清除旧文本内容
      infoContent.forEach((line, index) => {
        // 为每行文本添加tspan元素，确保没有多余的空格
        infoText.append("tspan")
                .attr("x", 10) // 控制文本靠左对齐，可以根据实际情况调整
                .attr("y", 20 + index * 20) // 调整行间距
                .text(line.trim()); // 使用trim()确保每行前后没有多余的空格
      });

      infoBox.style("display", null) // 显示信息框
           .attr("transform", `translate(${mouseX - 90}, ${svg1.clientHeight * 0.4})`); // 调整信息框位置
    }

    function mouseout() {
      focusLine.style("opacity", 0);
      focusCircleCases.style("opacity", 0);
      focusCircleMortalityRate.style("opacity", 0);
      infoBox.style("display", "none"); // 隐藏信息框
    }

    // 在SVG中添加一个分组容器用于显示信息框
    const infoBox = svg.append("g")
                      .style("display", "none"); // 初始状态不显示

    // 在信息框内添加白色半透明的矩形作为背景
    infoBox.append("rect")
          .attr("width", 180) // 根据需要调整大小
          .attr("height", 70) // 根据需要调整大小
          .attr("fill", "lightgrey")
          .attr("fill-opacity", 0.7)
          .attr("rx", 3) // 圆角大小
          .attr("ry", 5);

    // 在信息框内添加文本元素显示信息
    const infoText = infoBox.append("text")
                            .attr("x", 5) // 文本与矩形边缘的距离
                            .attr("y", 20) // 初始文本位置
                            .attr("fill", "black");
                            svg.append("g")
    .attr("class", "grid")
    .call(d3.axisLeft(yScaleLeft)
        .tickSize(-width)
        .tickFormat("")
    )
    .attr("stroke-opacity", 0.2)
    .attr("stroke-dasharray", "2,2");
  }

  

  // 进度条的更新设置 <<---------
  $: selectedDateString = dates[selectedDateIndex]; // 先更新selectedDateString

  $: if (selectedDateString) {
      updateMapColors(); // 根据最新的selectedDateString更新颜色
      updateCircles();   // 更新圆
  }

  // 选择不同的州呈现折线图的更新设置<<---------
  $: if (selectedState && svg2) {
  renderLineChart(selectedState);
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
    <svg bind:this={svg2} width="100%" height="98%"></svg>
  </div>

  <div class="controls">
    <select bind:value={selectedState} class="state-selector">
      {#each states as state}
        <option value="{state}">{state}</option>
      {/each}
    </select>
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
    height: 66vh; /* 视口高度的50% */
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

  .state-selector {
    width: 100%; /* 放大选择框 */
    max-width: 400px; /* 最大宽度 */
    height: 40px; /* 增加高度 */
    font-size: 16px; /* 字体大小 */
    margin: 20px auto; /* 居中显示 */
  }

  /* 媒体查询，用于小屏幕的样式调整 */
  @media (max-width: 600px) {
    .visualization {
      width: 70%; /* 小屏幕上的宽度调整 */
    }
  }
</style>
