<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import { csv } from 'd3-fetch';
  import { feature } from 'topojson-client';
  import { browser } from '$app/environment';

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

    if (browser) {
      // 处理.text-box的从左向右淡入
      const textObserver = new IntersectionObserver((entries) => {
        entries.forEach((entry) => {
          const target = entry.target;
          target.style.transition = 'opacity 0.5s ease-out, transform 0.5s ease-out';
          if (entry.isIntersecting) {
            target.style.opacity = 1;
            target.style.transform = 'translateX(0)';
          } else {
            target.style.opacity = 0;
            target.style.transform = 'translateX(-100%)';
          }
        });
      }, {threshold: 0.07});

      document.querySelectorAll('.text-box').forEach((element) => {
        textObserver.observe(element);
      });

      // 处理.title-box的从中心淡入
      const titleObserver = new IntersectionObserver((entries) => {
        entries.forEach((entry) => {
          const target = entry.target;
          // 由于我们只关注淡入，所以不需要transform
          target.style.transition = 'opacity 3s ease-out';
          if (entry.isIntersecting) {
            target.style.opacity = 1;
          } else {
            target.style.opacity = 0;
          }
        });
      }, {threshold: 0.2});

      document.querySelectorAll('.title-box').forEach((element) => {
        // 设置初始状态为不可见，这对于淡入效果是必要的
        element.style.opacity = 0;
        titleObserver.observe(element);
      });

      // 清理函数：断开两个observer
      return () => {
        textObserver.disconnect();
        titleObserver.disconnect();
      };
    }
  });

//----------function 区域 ----------//

  // 绘制地图和染色的代码 <<---------
  function renderMap(us) {
    const width = svg1.clientWidth; // 获取SVG的宽度
    const height = svg1.clientHeight; // 获取SVG的高度

    // 根据SVG的大小动态计算缩放比例
    const scale = Math.min(width, height) * 2.05;
    projection = d3.geoAlbersUsa().scale(scale).translate([width * 0.55, height / 2]);
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
    addColorLegend();
    addMortalityRateLegend();
    addNestedDeathCountCirclesOptimized();
    addRectangleAtTopLeft()
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
    d3.select(svg1).selectAll("circle.class-name").remove();

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
      const color = mortalityRate ? d3.scaleLinear().domain([0, 0.08]).range(["#46BF98", "#BF46BD"])(mortalityRate) : "lightgrey"; // 如果没有死亡率，使用默认颜色lightgrey

      d3.select(svg1).append("circle")
        .attr("class", "class-name")
        .attr("cx", center[0])
        .attr("cy", center[1])
        .attr("r", radius)
        .attr("fill", color)
        .attr("fill-opacity", 0.75) // 设置半透明
        .attr("stroke", "none"); // 无边框
    });
    addRectangleAtTopLeft()
  }

  function addColorLegend() {
    // 使用SVG的尺寸计算颜色比例尺的尺寸和位置参数
    const legendHeight = svg1.clientHeight * 0.25; // 比例尺高度为SVG高度的25%
    const legendWidth = svg1.clientWidth * 0.02; // 比例尺宽度为SVG宽度的2%
    const legendMargin = { top: svg1.clientHeight * 0.01, left: svg1.clientWidth * 0.002 }; // 边距基于SVG尺寸

    // 创建颜色比例尺的线性比例尺
    const colorScale = d3.scaleLinear()
      .domain([0, maxCases])
      .range(['#F9EBEA', '#641E16'])
      .nice();

    // 创建表示比例尺刻度的线性比例尺
    const legendScale = d3.scaleLinear()
      .domain([0, maxCases])
      .range([legendHeight, 0]);

    // 添加颜色比例尺的容器
    const legend = d3.select(svg1).append('g')
      .attr('class', 'color-legend')
      .attr('transform', `translate(${legendMargin.left}, ${svg1.clientHeight - legendHeight - legendMargin.top})`);

    // 添加颜色渐变
    legend.append('defs').append('linearGradient')
      .attr('id', 'gradient-color')
      .attr('x1', '0%')
      .attr('y1', '100%')
      .attr('x2', '0%')
      .attr('y2', '0%')
      .selectAll('stop')
      .data(colorScale.ticks().map((t, i, n) => ({ offset: `${100 * i / n.length}%`, color: colorScale(t) })))
      .enter().append('stop')
      .attr('offset', d => d.offset)
      .attr('stop-color', d => d.color);

    // 添加颜色渐变矩形
    legend.append('rect')
      .attr('width', legendWidth)
      .attr('height', legendHeight)
      .style('fill', 'url(#gradient-color)');

    // 添加比例尺刻度，使用.tickFormat来格式化刻度文本
    const legendAxis = d3.axisRight(legendScale)
      .ticks(5)
      .tickSize(legendWidth * 0.5) // 调整刻度大小
      .tickFormat(d => `${d / 1e6}M`); // 将感染人数转换为以百万（M）为单位的格式

    // 将比例尺刻度添加到图例
    const axisGroup = legend.append('g')
      .attr('class', 'color-axis')
      .attr('transform', `translate(${legendWidth}, 0)`)
      .call(legendAxis);

    // 设置刻度标签的样式，使其半透明
    axisGroup.selectAll('.tick text')
      .style('opacity', 0.5); // 设置半透明效果

    legend.append("text")
      .attr("class", "legend-title")
      .attr("x", 0)
      .attr("y", -10) // 调整这个值以在比例尺上方留出适当的空间
      .text("Infection Numbers (M)")
      .style("font-size", "10px") // 调整字体大小
      .attr("fill", "rgba(0,0,0,0.5)"); // 半透明的黑色文本
  }

  function addMortalityRateLegend() {
    const legendHeight = svg1.clientHeight * 0.25; // 比例尺高度为SVG高度的25%
    const legendWidth = svg1.clientWidth * 0.02; // 比例尺宽度为SVG宽度的2%
    const legendMargin = { top: svg1.clientHeight * 0.4, right: svg1.clientWidth * 0.002 }; // 调整到左上角

    const colorScale = d3.scaleLinear()
      .domain([0, 0.08]) // 对应于圆圈颜色的死亡率范围
      .range(["#46BF98", "#BF46BD"])
      .nice();

    const legendScale = d3.scaleLinear()
      .domain([0, 0.08]) // 死亡率范围
      .range([legendHeight, 0]);

    const legend = d3.select(svg1).append('g')
      .attr('class', 'mortality-rate-legend')
      .attr('transform', `translate(${legendMargin.right}, ${legendMargin.top})`);

    legend.append('defs').append('linearGradient')
      .attr('id', 'gradient-mortality-rate')
      .attr('x1', '0%')
      .attr('y1', '100%')
      .attr('x2', '0%')
      .attr('y2', '0%')
      .selectAll('stop')
      .data(colorScale.ticks().map((t, i, n) => ({ offset: `${100 * i / n.length}%`, color: colorScale(t) })))
      .enter().append('stop')
      .attr('offset', d => d.offset)
      .attr('stop-color', d => d.color);

    legend.append('rect')
      .attr('width', legendWidth)
      .attr('height', legendHeight)
      .style('fill', 'url(#gradient-mortality-rate)');

    const legendAxis = d3.axisRight(legendScale)
      .ticks(5)
      .tickFormat(d => `${d * 100}%`); // 将刻度转换为百分比格式

    legend.append('g')
      .attr('class', 'mortality-rate-axis')
      .attr('transform', `translate(${legendWidth}, 0)`)
      .call(legendAxis)
      .selectAll('.tick text')
      .style('opacity', 0.5); // 设置半透明效果

    legend.append("text")
      .attr("class", "legend-title")
      .attr("x", 0)
      .attr("y", -10) // 同样调整这个值以适当地放置文本
      .text("Mortality Rate (%)")
      .style("font-size", "10px") // 调整字体大小
      .attr("fill", "rgba(0,0,0,0.5)"); // 半透明的黑色文本
  }

  function addNestedDeathCountCirclesOptimized() {
    const svgWidth = svg1.clientWidth;
    const svgHeight = svg1.clientHeight;
    const deathCounts = [2000, 5000, 10000].reverse(); // 死亡数，逆序以便从大到小绘制
    const maxDeath = d3.max(deathCounts);

    // 定义半径的比例尺
    const radiusScale = d3.scaleSqrt().domain([0, maxDeath]).range([0, svgHeight * 0.06]);

    // 最大圆的圆心Y坐标，位于SVG底部适当位置
    let cy = svgHeight * 0.3 - radiusScale(maxDeath);

    deathCounts.forEach((death, index) => {
        const radius = radiusScale(death);
        const cx = svgWidth * 0.002 + radius; // 圆心X坐标，保持在左侧并根据最大半径调整

        // 绘制圆
        d3.select(svg1).append("circle")
            .attr("cx", cx)
            .attr("cy", cy)
            .attr("r", radius)
            .attr("fill", "#D6DBDF")
            .attr("stroke", "#5D6D7E")
            .attr("stroke-width", "0.5");

        // 添加死亡数标注
        d3.select(svg1).append("text")
            .attr("x", cx + radius + svgWidth * 0.0085)
            .attr("y", cy + radius * 0.002) // 在圆上方一点位置添加标注
            .text(`${death / 1000}k`)
            .attr("text-anchor", "middle")
            .style("font-size", "10px")
            .attr("fill", "rgba(0,0,0,0.5)");
    });

    // 添加表示这些圆代表死亡数的标题
    d3.select(svg1).append("text")
        .attr("x", svgWidth * 0.05)
        .attr("y", cy - radiusScale(maxDeath) - 12) // 在最大圆的上方添加标题
        .text("Deaths Count(K)")
        .attr("text-anchor", "middle")
        .style("font-size", "10px")
        .attr("fill", "rgba(0,0,0,0.5)");
  }

  function addRectangleAtTopLeft() {
    const svgWidth = svg1.clientWidth;
    const svgHeight = svg1.clientHeight;

    // 计算矩形的宽度和高度，基于SVG的尺寸
    const rectWidth = svgWidth * 0.05;
    const rectHeight = svgHeight * 0.05;

    // 计算矩形在SVG中的位置，位于左上角的1%位置
    const rectX = - svgWidth * 0.006;
    const rectY = 0;

    // 添加矩形到SVG
    d3.select(svg1).append("rect")
        .attr("x", rectX)
        .attr("y", rectY)
        .attr("width", rectWidth)
        .attr("height", rectHeight)
        .attr("fill", "rgba(230,234,235,255)") // 使用指定的rgba颜色填充
        .attr("stroke", "black") // 可选，如果需要矩形边框
        .attr("stroke-width", "0"); // 可选，设置矩形边框的宽度
  }



  // 第一个图的播放功能代码 <<---------
  let playing = false; // 用于追踪播放状态
  let intervalId; // 用于存储定时器ID

  // 播放或暂停功能
  function togglePlay() {
    if (!playing) {
      // 检查是否需要从头开始播放
      if (selectedDateIndex >= dates.length - 1) {
        // 如果当前是最后一个日期，先重置到初始状态
        resetPlay();
      }
      // 开始或继续播放
      playing = true;
      intervalId = setInterval(() => {
        if (selectedDateIndex < dates.length - 1) {
          selectedDateIndex += 1;
        } else {
          stopPlaying(); // 到达末尾时停止播放
        }
      }, 10);
    } else {
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

    const tooptiphight = svg2.clientHeight;

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
    
    const iconPath = 'icon/icons8-protection-mask-64.png';
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
                .style("left", (event.pageX - 50) + "px")
                .style("top", (event.pageY - tooptiphight*0.5) + "px")
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

    const iconPath2 = 'icon/icons8-syringe-64.png';

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
                .style("left", (event.pageX - 50) + "px")
                .style("top", (event.pageY - tooptiphight*0.5) + "px")
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
    const iconPath3 = 'icon/icons8-coronavirus-64.png';

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
                .style("left", (event.pageX - 50) + "px")
                .style("top", (event.pageY - tooptiphight*0.5) + "px")
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
    const iconPath4 = 'icon/icons8-pcr-test-64.png';

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
                .style("left", (event.pageX - 50) + "px") 
                .style("top", (event.pageY - tooptiphight*0.5) + "px")
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
    const iconPath5 = 'icon/icons8-health-shield-64.png';

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
                .style("left", (event.pageX - 50) + "px")
                .style("top", (event.pageY - tooptiphight*0.5) + "px")
                .html("2022-8-11: CDC simplified COVID-19 guidance, emphasizing vaccines and updating protocols for exposure, without requiring quarantine but recommending masking and testing."); // 设置鼠标悬停时显示的文本
        })
        .on("mouseout", function() {
            tooltipDiv.style("display", "none"); // 鼠标移开时隐藏工具提示
        });
    
    const targetPercentage = 0.0015; // 目标百分比，0.15%
    const iconPath6 = 'icon/icons8-virus-dna-64.png'; // 图标路径

    // 计算百分之0.15%在右侧Y轴上的像素位置
    const targetY = yScaleRight(targetPercentage);

    // 绘制横向的虚线
    svg.append("line")
      .attr("x1", 0)
      .attr("x2", width)
      .attr("y1", targetY)
      .attr("y2", targetY)
      .style("stroke", "#F1948A") // 设置线条颜色
      .style("stroke-width", "2px")
      .style("stroke-dasharray", "5,5") // 定义虚线模式
      .style("opacity", 0.7);

    // 添加图标到右侧Y轴对应位置
    svg.append("image")
      .attr("xlink:href", iconPath6)
      .attr("width", 32)
      .attr("height", 32)
      .attr("x", width + 27) // 略微偏移以防覆盖Y轴
      .attr("y", targetY - 16) // 调整图标位置居中对齐虚线
      .on("mouseover", function(event) {
        tooltipDiv
          .style("display", "block")
          .style("left", (event.pageX - 80) + "px")
          .style("top", (event.pageY - tooptiphight*0.25) + "px")
          .html("The mortality rate of common influenza is 0.15%"); // 设置工具提示内容
      })
      .on("mouseout", function() {
        tooltipDiv.style("display", "none");
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
        .duration(3000) // 动画持续时间
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


<!---------- HTML的构成部分 这里写文案！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！---------->
<div class="backdrop"></div>

<div class="container">

  <div class="title-box">
    <h1>Is the Threat of <br>COVID-19<br> DIMINISHING <br>in the United States?<br></h1>
    <h2>Exploration of COVID-19's Evolution in the United States and the Impact of "Big Events"</h2>
  </div>

  <div class="text-box" style="text-align: left;">
    <h2>Since March 23, 2023, USCIS announced the termination of COVID-related flexibilities, signaling that the United States will cease the collection and publication of data related to COVID-19 thereafter. <br> However, does this mean COVID-19 has truly left us behind, or is it no longer worthy of our attention and vigilance?</h2>
  </div>

  <div class="image-container">
    <div class="image-item">
      <img src="pictures/CDC.png" alt="Event 1">
      <br>
      <a href="https://www.uscis.gov/newsroom/alerts/uscis-announces-end-of-covid-related-flexibilities" target="_blank">USCIS Announces End of COVID-Related Flexibilities</a>
    </div>
  </div>

  <div class="text-box" style="text-align: left;">
    <h2>COVID-19 on Macro-Level over Time 🌎:</h2>
    <h3>Before we embark on our journey of exploration, let's first pause to examine the broader transformations COVID-19 is undergoing on a macroscopic scale. This will provide us with a preliminary overview of the evolving situation across individual states over time.</h3>
  </div>

  <div class="text-box" style="text-align: left;">
    <h2>How to Interact with the COVID-19 Map below:</h2>
    <h3>Click "▶️ Play" to run the COVID Map below. And you also can click "⏸️ Pause" and drag the time bar⌛ to observe changes in COVID-19 over a certain period. The map records the following data on COVID-19 in each state from March 21, 2020 to March 23, 2023:</h3>
    <h3>The color in each state represents the cumulative number of COVID-19 infections(M). 🤧
      <br>The size of the circle represents the number of deaths(K) caused by COVID-19 in each state. 💀
      <br>The color of the circle represents the Mortality Rate(%) in each state. ☣️</h3> 
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

  <div class="text-box" style="text-align: left;">
    <h2>What does the COVID-19 Map above Tell?</h2>
    <h3>Although the pandemic's trajectory seems chaotic across different states in the United States, we can still discern some common patterns among them. In the early stages of COVID-19's spread within the country, the mortality rate was exceptionally high in various states. However, as the number of infections increased, the mortality rate gradually decreased.</h3>
    <h3>It is noteworthy that the evolution of the pandemic does not seem to correlate with time. In other words, there appear to be certain "Big Events" influencing the progression of COVID-19 across the states, causing fluctuations in the pandemic's trajectory at certain periods on a macro level.</h3>
    <h4>Therefore, taking "Big Events" into consideration is essential!🔥</h4>
  </div>

  <div class="text-box" style="text-align: left;">
    <h2>So, What is "Big Events?🤔"</h2>
    <h3>"Big Events" include the U.S. government's macro policies on COVID-19, health recommendations from the World Health Organization (WHO), and the emergence of highly differentiated COVID variants.These “Big Events” often represent major changes regarding the COVID-19 virus. They may indirectly affect the changing trend of COVID-19 in the United States. Therefore, they were included in the scope of our study to provide a comprehensive understanding of how these events influenced public health responses and the trajectory of the pandemic. Here are two example of the "Big Events":</h3>
  </div>

  <div class="image-container">
    <div class="image-item">
      <img src="pictures/event1.jpg" alt="Event 1">
      <br>
      <a href="https://www.cdc.gov/mmwr/volumes/69/wr/mm6949e2.htm" target="_blank">CDC release anti-epidemic guidance</a>
    </div>
    <div class="image-item">
      <img src="pictures/event2.jpg" alt="Event 2">
      <br>
      <a href="https://www.cdc.gov/ncird/whats-new/covid-19-variant-update-2023-08-30.html" target="_blank">Omicron was discovered in US</a>
    </div>
  </div>

  <div class="text-box" style="text-align: left;">
    <h2>Let's Zoom in! COVID-19 on Micro-Level, What Happen in California🐻over Time?</h2>
    <h3>After observing the COVID-19 trend in the entire United States, it's time to discuss the info within the specific state. Let's choose California as our study subject to delve more deeply into the evolution of COVID-19 by using a Line Chart.📈, a visualization that allow us to explore the evolution and tendency of COVID-19 over time.</h3>
  </div>

  <div class="text-box" style="text-align: left;">
    <h2>How to Interact with the Line Chart below:</h2>
    <h3>We are primarily focusing on the trajectory of COVID-19 in California🐻, which is set as the default option. However, we also offer alternatives for tracking the pandemic in other states.</h3>
    <h3>🔎State Selector: Picking one state to explore its COVID-19's trajectory with the "Big Events" time line.
      <br>📅"Big Events" Icon: Some "Big Events" we have collected, hovering them to see more information.
      <br>👆🏻Hovering the mouse over the line chart allows observation of the infection numbers and mortality rate for corresponding dates.</h3>
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
  
  <div class="text-box" style="text-align: left;">
    <h2>Trajectory of COVID-19 in California🐻:</h2>
    <h3>XXX</h3>
  </div>

  <div class="text-box" style="text-align: left;">
    <h2>The Effect of the "Big Events":</h2>
    <h3>XXX</h3>
  </div>

  <div class="visualization">
    <svg bind:this={svg3} width="100%" height="98%"></svg>
  </div>

  <div class="text-box" style="text-align: left;">
    <h2>How Mortality Rate Change Over Time:</h2>
    <h3>The graph illustrates the rate of change in mortality over a timeline from 2021 to 2023. There was a significant spike in April 2020, indicating a sharp increase in mortality rate corresponding to the surge in COVID-19 deaths at that time. This is followed by fluctuations above and below the baseline, but the overall trend shows a decrease in mortality rate as the fluctuations become less pronounced over time.  </h3>
    
    <h3>The arrival of the Omicron variant in November 2021 correlated with an uptick in infection rates, yet mortality rates remained comparatively steady. This stability in mortality could suggest a lower severity of the Omicron variant compared to its predecessors.
      By October 2022, the rate appears to have stabilized near the baseline, suggesting that the mortality rate changes have significantly reduced or normalized compared to the initial spike.
    </h3>
  </div>

  <div class="text-box" style="text-align: left;">
    <h2>What does a reduced oscillation in the rate of change of mortality tell us?</h2>
    <h3>Analysis of the changing rate of the mortality rate</h3>

  <h3>The graph's trend of reduced oscillation suggests a stabilization of the situation; the mortality rate from COVID-19 is becoming more predictable and less volatile over time. It could indicate that, while infections may continue, the impact on mortality has lessened—potentially due to factors like widespread vaccination and booster shots, improved treatment protocols, or the emergence of less lethal virus variants like Omicron. This stabilization near the baseline by October 2022 could imply that mortality rates have reached a level consistent with endemic conditions.
  </h3>

  </div>
</div>

<div id="tooltip" style="display: none; position: absolute; padding: 8px; background: rgba(255, 255, 255, 0.85); border: none; border-radius: 5px; pointer-events: none;">Tooltip</div>


<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Oswald:wght@200..700&display=swap" rel="stylesheet">

<style>

  :global(body) {
    background-color: #EAF2F8; /* 背景颜色 */
    background-image: url('/covid8.jpg'); /* 设置背景图片 */
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

  .title-box {
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    height: 100vh; /* 设置高度占满整个视口高度 */
    width: 98vw; /* 设置宽度占满整个视口宽度 */
    margin: 0; /* 移除外边距 */
    padding: 0; /* 调整内边距，如果需要的话 */
    font-family: 'Oswald', sans-serif;
    opacity: 0;
    transition: opacity 1s ease-out; /* 如果你想要更长时间的淡入效果 */
  }

  .title-box h1 {
      font-size: 3.7rem; /* 根据需要调整标题的大小 */
      font-weight: 700; /* Oswald的粗体权重 */
      text-align: center
  }

  .title-box h2 {
      font-size: 1.5rem; /* 根据需要调整副标题的大小 */
      font-weight: 500; /* Oswald的常规权重 */
  }

  .title-box h3 {
      font-size: 1.5rem; /* 根据需要调整副标题的大小 */
      font-weight: 500; /* Oswald的常规权重 */
  }


  .text-box {
    width: 70%; /* 与可视化组件宽度一致 */
    text-align: center; /* 文本居中 */
    font-family: 'Oswald', sans-serif;
    opacity: 0;
    transform: translateX(-100%);
  }

  .text-box h1 {
      font-size: 1.5rem; /* 根据需要调整副标题的大小 */
      font-weight: 500; /* Oswald的常规权重 */
  }

  .text-box h2 {
      font-size: 1.5rem; /* 根据需要调整副标题的大小 */
      font-weight: 500; /* Oswald的常规权重 */
  }

  .text-box h3 {
      font-size: 1.3rem; /* 根据需要调整副标题的大小 */
      font-weight: 420; /* Oswald的常规权重 */
  }

  .text-box h4 {
      font-size: 1.3rem; /* 根据需要调整副标题的大小 */
      font-weight: 900; /* Oswald的常规权重 */
  }

  .controls {
    display: flex;
    flex-direction: row; /* 使内容水平排列 */
    align-items: center; /* 横向居中对齐 */
    padding: 1rem;
    width: 60%; /* 调整为所需的百分比 */
    gap: 40px; /* 控制内部元素之间的间距 */
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
    max-width: 300px; /* 最大宽度 */
    height: 40px; /* 增加高度 */
    border-radius: 5px;
    font-size: 16px; /* 字体大小 */
    margin: 20px auto; /* 居中显示 */
    margin-bottom: -20px;
    margin-top: 70px;
    background-color: #EAEDED; /* 默认的浅色背景 */
    transition: background-color 0.3s; /* 平滑的背景色过渡效果 */
    border: 1px solid lightgrey; /* 设置边框颜色为lightgrey */
  }

  .image-container {
    display: flex; /* 横向排列图片 */
    justify-content: space-around; /* 图片之间的间隔 */
    flex-wrap: wrap; /* 允许内容换行 */
    width: 70vw; 
    margin-bottom: 2rem; /* 为滚动提供空间 */
  }

  .image-item {
    text-align: center; /* 使图片和文字在其容器内居中 */
    width: auto;
  }

  .image-item img {
    width: auto; /* 根据需要调整宽度 */
    height: 35vh; /* 保持图片高度一致 */
    object-fit: cover; /* 裁剪图片以填充容器 */
    border-radius: 10px; /* 如需要，为图片添加圆角 */
    margin-bottom: 0.5rem; /* 图片与文字说明之间的间隔 */
  }

  .image-item p {
    margin: 0; /* 移除段落默认的外边距 */
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
