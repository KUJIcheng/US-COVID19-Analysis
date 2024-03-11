<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import { csv } from 'd3-fetch';
  import { feature } from 'topojson-client';

  let svg1;
  let svg2;
  let svg3

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
    renderLineChartWithMortalityRateChange()
  });

//----------function 区域 ----------//

  // 绘制地图和染色的代码 <<---------
  function renderMap(us) {
    const width = svg1.clientWidth; // 获取SVG的宽度
    const height = svg1.clientHeight; // 获取SVG的高度

    // 根据SVG的大小动态计算缩放比例
    const scale = Math.min(width, height) * 2.05;
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
        .attr('stroke', '#AEB6BF')
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
    const margin = { top: 20, right: 80, bottom: 50, left: 80 },
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
        return day === 21; // 仅选择每月23号的数据
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

    // 添加横向的虚线
    // 绘制y轴网格线作为背景
    const yAxisGrid = d3.axisLeft(yScaleLeft)
      .tickSize(-width) // 使刻度线横跨整个图表宽度
      .tickFormat("") // 不显示刻度线的文本
      .ticks(10); // 控制网格线的数量

    // 将网格线添加到SVG中
    svg.append("g")
      .attr("class", "grid")
      .call(yAxisGrid)
      .attr("stroke-opacity", 0.2) // 设置虚线的透明度
      .attr("stroke-dasharray", "2,2"); // 设置虚线的样式

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

    
    const pathCases = svg.append("path")
        .datum(filteredData)
        .attr("class", "line cases-line")
        .style("stroke", "#CB4335")
        .style("fill", "none")
        .style("stroke-width", "3px")
        .attr("d", lineCases)
        .attr("stroke-dasharray", function() {
            const length = this.getTotalLength(); // 获取线条的总长度
            return length + " " + length; // 设置dasharray为线条长度
        })
        .attr("stroke-dashoffset", function() {
            return this.getTotalLength(); // 初始偏移量为线条长度
        });

    // 启动动画，使线条从左到右展开
    pathCases.transition()
        .duration(2000) // 动画持续时间
        .attr("stroke-dashoffset", 0); // 最终偏移量为0

    // 绘制mortalityRate线条并添加动画
    const pathMortalityRate = svg.append("path")
        .datum(filteredData)
        .attr("class", "line mortality-rate-line")
        .style("stroke", "#5B2C6F")
        .style("fill", "none")
        .style("stroke-width", "3px")
        .attr("d", lineMortalityRate)
        .attr("stroke-dasharray", function() {
            const length = this.getTotalLength();
            return length + " " + length;
        })
        .attr("stroke-dashoffset", function() {
            return this.getTotalLength();
        });

    // 启动动画
    pathMortalityRate.transition()
        .duration(2000) // 同样可以调整动画持续时间
        .attr("stroke-dashoffset", 0);

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
    // 在body中创建一个工具提示的div并设置基本的样式
    const tooltip = d3.select("body").append("div")
        .attr("class", "tooltip") // 可以通过CSS进一步样式化
        .style("position", "absolute")
        .style("background-color", "rgba(255, 255, 255, 0.8)")
        .style("padding", "5px")
        .style("border-radius", "5px")
        .style("opacity", 0) // 初始不可见
        .style("pointer-events", "none"); // 防止tooltip干扰鼠标事件
    
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

      // 更新工具提示的内容和位置
      tooltip.style("opacity", 1)
            .html(`Date: ${d3.timeFormat("%Y-%m-%d")(d.date)}<br>Cases: ${d.cases}<br>Mortality Rate: ${(d.mortalityRate * 100).toFixed(2)}%`)
            .style("left", (event.pageX + 20) + "px")
            .style("top", (event.pageY - 20) + "px");
    }
    
    // 为鼠标移出事件更新工具提示的隐藏逻辑
    function mouseout() {
        tooltip.style("opacity", 0);
        focusLine.style("opacity", 0);
        focusCircleCases.style("opacity", 0);
        focusCircleMortalityRate.style("opacity", 0);
    }

    svg.append("rect")
      .attr("width", width) // 使矩形的宽度与SVG的宽度相同
      .attr("height", height) // 使矩形的高度与SVG的高度相同
      .style("fill", "none") // 使矩形透明
      .style("pointer-events", "all") // 确保矩形可以捕获鼠标事件
      .on("mouseout", mouseout) // 绑定鼠标移出事件
      .on("mousemove", mousemove); // 绑定鼠标移动事件

    // 第一个事件
    // 将字符串日期转换为日期对象
    const targetDate = d3.timeParse("%Y-%m-%d")("2020-04-03");

    // 计算目标日期在x轴上的位置
    const targetX = xScale(targetDate);

    // 在SVG上绘制一条垂直虚线，位于2020年4月3日的位置
    svg.append("line")
        .attr("x1", targetX)
        .attr("x2", targetX)
        .attr("y1", 0)
        .attr("y2", height)
        .style("stroke", "cyan") // 设置线条颜色为蓝绿色
        .style("stroke-width", "2px") // 设置线条宽度
        .style("stroke-dasharray", "5,5") // 定义虚线模式，5px线段和5px间隔
        .style("opacity", 0.7); // 设置线条透明度为半透明
    
    // 图标的路径，根据你的项目结构调整
    const iconPath = 'icons8-protection-mask-64.png';
    const iconY = height + 17;

    // 添加图标到SVG中
    svg.append("image")
        .attr("xlink:href", iconPath)
        .attr("width", 32)
        .attr("height", 32)
        .attr("x", targetX - 16)
        .attr("y", iconY);
    
    // 选择工具提示元素
    const tooltipDiv = d3.select("#tooltip");

    // 为图标添加鼠标悬停事件
    svg.select("image")
        .on("mouseover", function(event) {
            tooltipDiv
                .style("display", "block") // 显示工具提示
                .style("left", (event.pageX - 20) + "px") // 根据鼠标位置定位，添加一些偏移
                .style("top", (event.pageY + 20) + "px")
                .html("2020-4-3: The White House Coronavirus Task Force and CDC recommended that persons wear a cloth face covering in public to slow the spread of COVID-19."); // 设置工具提示的内容
        })
        .on("mouseout", function() {
            tooltipDiv.style("display", "none"); // 鼠标移开时隐藏工具提示
        });
    
    // 第二个图标和事件
    // 将字符串日期转换为日期对象，针对2021年1月20日
    const targetDate2 = d3.timeParse("%Y-%m-%d")("2021-01-20");

    // 计算目标日期在x轴上的位置
    const targetX2 = xScale(targetDate2);

    // 在SVG上绘制一条垂直虚线，位于2021年1月20日的位置
    svg.append("line")
        .attr("x1", targetX2)
        .attr("x2", targetX2)
        .attr("y1", 0)
        .attr("y2", height)
        .style("stroke", "cyan")
        .style("stroke-width", "2px")
        .style("stroke-dasharray", "5,5")
        .style("opacity", 0.7);

    const iconPath2 = 'icons8-syringe-64.png';

    // 添加新图标到SVG中
    svg.append("image")
        .attr("xlink:href", iconPath2)
        .attr("width", 32)
        .attr("height", 32)
        .attr("x", targetX2 - 16)
        .attr("y", height + 17)
        .on("mouseover", function(event) {
            tooltipDiv
                .style("display", "block")
                .style("left", (event.pageX - 20) + "px")
                .style("top", (event.pageY + 20) + "px")
                .html("2021-1-20: President Joe Biden launched a COVID-19 plan focusing on vaccination, testing, and addressing health disparities. It included a $160 billion proposal for a national vaccination program and expanded testing.");
        })
        .on("mouseout", function() {
            tooltipDiv.style("display", "none");
        });

    // 第三个图标和事件
    // 将字符串日期转换为日期对象，针对2021年11月26日
    const targetDate3 = d3.timeParse("%Y-%m-%d")("2021-11-26");

    // 计算目标日期在x轴上的位置
    const targetX3 = xScale(targetDate3);

    // 在SVG上绘制一条垂直虚线，位于2021年11月26日的位置
    svg.append("line")
        .attr("x1", targetX3)
        .attr("x2", targetX3)
        .attr("y1", 0)
        .attr("y2", height)
        .style("stroke", "cyan")
        .style("stroke-width", "2px")
        .style("stroke-dasharray", "5,5")
        .style("opacity", 0.7);

    // 新图标的路径
    const iconPath3 = 'icons8-coronavirus-64.png';

    // 添加新图标到SVG中
    svg.append("image")
        .attr("xlink:href", iconPath3)
        .attr("width", 32)
        .attr("height", 32)
        .attr("x", targetX3 - 16) // 将图标中心对齐到虚线上
        .attr("y", height + 17) // 假设将图标放置在x轴下方20像素处
        .on("mouseover", function(event) {
            tooltipDiv
                .style("display", "block")
                .style("left", (event.pageX - 20) + "px")
                .style("top", (event.pageY + 30) + "px")
                .html("2021-11-26: WHO classified the Omicron variant, B.1.1.529, as a Variant of Concern due to its many mutations and potential higher reinfection risk, first identified in South Africa on November 24.​"); // 设置鼠标悬停时显示的文本
        })
        .on("mouseout", function() {
            tooltipDiv.style("display", "none");
        });
    
    // 第四个事件
    // 将字符串日期转换为日期对象，针对2022年2月25日
    const targetDate4 = d3.timeParse("%Y-%m-%d")("2022-02-25");

    // 计算目标日期在x轴上的位置
    const targetX4 = xScale(targetDate4);

    // 在SVG上绘制一条垂直虚线，位于2022年2月25日的位置
    svg.append("line")
        .attr("x1", targetX4)
        .attr("x2", targetX4)
        .attr("y1", 0)
        .attr("y2", height)
        .style("stroke", "cyan")
        .style("stroke-width", "2px")
        .style("stroke-dasharray", "5,5")
        .style("opacity", 0.7);

    // 图标的路径
    const iconPath4 = 'icons8-pcr-test-64.png';

    // 添加图标到SVG中
    svg.append("image")
        .attr("xlink:href", iconPath4)
        .attr("width", 32)
        .attr("height", 32)
        .attr("x", targetX4 - 16) // 将图标中心对齐到虚线上
        .attr("y", height + 17) // 假设将图标放置在x轴下方20像素处
        .on("mouseover", function(event) {
            tooltipDiv
                .style("display", "block")
                .style("left", (event.pageX - 20) + "px") 
                .style("top", (event.pageY + 20) + "px")
                .html("2022-2-25: CDC updated COVID-19 guidance to reflect reduced severity risks due to available treatments and vaccines, emphasizing the continued importance of vaccinations and updated protocols for exposure and infection management.​"); // 设置鼠标悬停时显示的文本
        })
        .on("mouseout", function() {
            tooltipDiv.style("display", "none");
        });

    // 第五个事件
    // 将字符串日期转换为日期对象，针对2022年8月11日
    const targetDate5 = d3.timeParse("%Y-%m-%d")("2022-08-11");

    // 计算目标日期在x轴上的位置
    const targetX5 = xScale(targetDate5);

    // 在SVG上绘制一条垂直虚线，位于2022年8月11日的位置
    svg.append("line")
        .attr("x1", targetX5)
        .attr("x2", targetX5)
        .attr("y1", 0)
        .attr("y2", height)
        .style("stroke", "cyan")
        .style("stroke-width", "2px")
        .style("stroke-dasharray", "5,5")
        .style("opacity", 0.7);

    // 图标的路径
    const iconPath5 = 'icons8-health-shield-64.png';

    // 添加图标到SVG中
    svg.append("image")
        .attr("xlink:href", iconPath5)
        .attr("width", 32)
        .attr("height", 32)
        .attr("x", targetX5 - 16) // 将图标中心对齐到虚线上
        .attr("y", height + 17) // 假设将图标放置在x轴下方20像素处
        .on("mouseover", function(event) {
            tooltipDiv
                .style("display", "block")
                .style("left", (event.pageX - 20) + "px") // 根据鼠标位置定位，添加一些偏移
                .style("top", (event.pageY + 20) + "px") // 根据鼠标位置定位，添加一些偏移
                .html("2022-8-11: CDC simplified COVID-19 guidance, emphasizing vaccines and updating protocols for exposure, without requiring quarantine but recommending masking and testing."); // 设置鼠标悬停时显示的文本
        })
        .on("mouseout", function() {
            tooltipDiv.style("display", "none"); // 鼠标移开时隐藏工具提示
        });
  }

  function renderLineChartWithMortalityRateChange() {
    const margin = { top: 20, right: 80, bottom: 20, left: 80 },
          width = svg3.clientWidth - margin.left - margin.right,
          height = svg3.clientHeight - margin.top - margin.bottom;
    
    const filteredData = Object.entries(dataByStateAndDate)
      .filter(([date]) => {
        const day = new Date(date).getDate();
        return day === 21; // 仅选择每月21号的数据
      })
      .map(([date, statesData]) => {
        // 计算所有州的mortalityRateChange的平均值
        const allStates = Object.values(statesData); // 获取所有州的数据
        const averageMortalityRateChange = allStates.reduce((sum, curr) => sum + (curr.mortalityRateChange || 0), 0) / allStates.length;

        return {
          date: d3.timeParse("%Y-%m-%d")(date),
          mortalityRateChange: averageMortalityRateChange // 使用计算出的平均死亡率变化率
        };
      });

    // 清空之前的绘图
    d3.select(svg3).selectAll('*').remove();

    const svg = d3.select(svg3)
                  .attr("viewBox", `0 0 ${width + margin.left + margin.right} ${height + margin.top + margin.bottom}`)
                  .append("g")
                  .attr("transform", `translate(${margin.left},${margin.top})`);

    // 假设filteredData是已经根据需要处理的数据
    const xScale = d3.scaleTime()
                    .range([0, width])
                    .domain(d3.extent(filteredData, d => d.date));

    const yScale = d3.scaleLinear()
                    .range([height, 0])
                    .domain([-0.028, 0.028]);

    // 定义x轴和y轴
    const xAxis = d3.axisBottom(xScale);
    const yAxis = d3.axisLeft(yScale)
                    .tickFormat(d3.format(".0%")); // 将刻度格式化为百分比

    // 绘制x轴
    svg.append("g")
      .attr("transform", `translate(0, ${height / 2})`) // x轴放在中间
      .call(xAxis);

    // 绘制y轴
    svg.append("g")
      .call(yAxis);
    
    // 添加横向虚线
    svg.append("g")
       .attr("class", "grid")
       .call(d3.axisLeft(yScale)
             .tickSize(-width) // 横跨整个图表的宽度
             .tickFormat("") // 不显示刻度文本
       )
       .attr("stroke-dasharray", "2,2") // 设置为虚线
       .attr("stroke-opacity", 0.7) // 可以调整虚线的透明度
       .selectAll(".tick line") // 选择所有的刻度线
       .attr("stroke", "lightgrey"); // 设置虚线的颜色

    // 定义线条生成器
    const lineGenerator = d3.line()
        .defined(d => !isNaN(d.mortalityRateChange)) // 确保数据有效
        .curve(d3.curveMonotoneX) // 使用MonotoneX曲线让线条平滑
        .x(d => xScale(d.date))
        .y(d => yScale(d.mortalityRateChange));

    // 绘制线条，并设置初始的 stroke-dasharray 和 stroke-dashoffset
    const path = svg.append("path")
        .datum(filteredData) // 绑定处理后的数据
        .attr("fill", "none")
        .attr("stroke", "#E74C3C") // 使用指定的颜色
        .attr("stroke-width", 3)
        .attr("d", lineGenerator)
        .attr("stroke-dasharray", function() {
            const length = this.getTotalLength(); // 获取线条的总长度
            return `${length} ${length}`; // 设置dasharray为线条长度
        })
        .attr("stroke-dashoffset", function() {
            return this.getTotalLength(); // 初始偏移量为线条长度
        });

    // 应用动画，逐渐将 stroke-dashoffset 减少到 0
    path.transition()
        .duration(3000) // 动画持续时间，可以根据需要调整
        .attr("stroke-dashoffset", 0);
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
<div class="backdrop"></div>

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

  <div class="controls">
    <select bind:value={selectedState} class="state-selector">
      {#each states as state}
        <option value="{state}">{state}</option>
      {/each}
    </select>
  </div>

  <div class="visualization">
    <svg bind:this={svg2} width="100%" height="98%"></svg>
  </div>
  
  <div class="text-box">
    <h2>Here we describe some of the content of the above line chart, and then lead to our next visualization below, which is a line chart about the rate of change of mortality.</h2>
    <h2>the text box background can be edit to have a better look</h2>
  </div>

  <div class="visualization">
    <svg bind:this={svg3} width="100%" height="98%"></svg>
  </div>

  <div class="text-box">
    <h1>Draw the conclusion here.</h1>
  </div>
</div>

<div id="tooltip" style="display: none; position: absolute; padding: 8px; background: white; border: none; border-radius: 3px; pointer-events: none;">Tooltip</div>

<style>

  :global(body) {
    background-color: #EAF2F8; /* 背景颜色 */
    background-image: url('/covid8.jpg'); /* 设置背景图片，根据你的实际路径调整 */
    background-size: cover; /* 保证背景图片铺满整个容器 */
    background-attachment: fixed; /* 背景图片不随滚动条滚动 */
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
    border-radius: 10px;
    padding: 1rem;
    margin-bottom: 2rem; /* 为滚动提供空间 */
    box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
    overflow: auto; /* 如果内容超出，则允许滚动 */
    
    background-color: rgba(234, 237, 237, 0.9); /* 设置半透明的底色 */
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
  gap: 20px; /* 控制内部元素之间的间距 */
  margin-top: -4%; /* 向上移动 */
  }

  .controls input[type="range"], .controls p {
    flex-grow: 1; /* 让进度条和日期标签填充剩余空间 */
  }

  .controls button {
    margin-right: 20px; /* 按钮和日期的间距 */
    border-radius: 10px; /* 圆角 */
    height: 40px;
    width: 90px;
    background-color: #EAEDED; /* 默认的浅色背景 */
    transition: background-color 0.3s; /* 平滑的背景色过渡效果 */
    border: 1px solid lightgrey; /* 设置边框颜色为lightgrey */
  }

  .controls button:hover {
    background-color: #D5D8DC; /* 鼠标悬停时的深色背景 */
  }

  .controls p {
    margin-right: 0;
    padding: 0.5rem 1rem; /* 增加一些内边距以提升可读性 */
    background-color: #EAEDED; /* 背景色 */
    border: 1px solid lightgrey; /* 边框颜色 */
    border-radius: 10px; /* 圆角 */
  }

  .controls p::before {
    content: "Date: ";
  }

  .state-selector {
    width: 100%; /* 放大选择框 */
    max-width: 400px; /* 最大宽度 */
    height: 40px; /* 增加高度 */
    border-radius: 5px;
    font-size: 16px; /* 字体大小 */
    margin: 20px auto; /* 居中显示 */
    margin-bottom: 0px;
    margin-top: 70px;
  }

  .backdrop {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    backdrop-filter: blur(6px);
    z-index: -1;
  }


  #tooltip {
    display: none; /* 初始不显示 */
    position: absolute; /* 使用绝对定位 */
    padding: 8px;
    background: white;
    border: 1px solid black;
    border-radius: 4px;
    pointer-events: none; /* 防止阻挡鼠标事件 */
    z-index: 100; /* 确保在上层 */
    max-width: 300px; /* 设置最大宽度 */
    white-space: normal; /* 允许文本换行 */
    overflow-wrap: break-word; /* 在需要的时候断词 */
  }

  /* 媒体查询，用于小屏幕的样式调整 */
  @media (max-width: 600px) {
    .visualization {
      width: 70%; /* 小屏幕上的宽度调整 */
    }
  }
</style>
