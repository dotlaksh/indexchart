<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import { createChart, ColorType } from 'lightweight-charts';
  import type { StockData, Interval } from '../types';

  export let data: StockData[] = [];
  export let stockName: string = '';
  export let interval: Interval = { label: 'D', value: '1d', range: '5y' };

  let chartContainer: HTMLElement;
  let legendContainer: HTMLElement;
  let chart: any;
  let barSeries: any;
  let maSeries: any;
  
  
  let resizeObserver: ResizeObserver;

  function formatPrice(price: number): string {
    return price.toFixed(2);
  }

  function formatPercentage(percentage: number): string {
    return percentage.toFixed(2) + '%';
  }

  

  function calculateMovingAverage(data: StockData[], period: number): { time: number, value: number }[] {
    let result: { time: number, value: number }[] = [];
    for (let i = period - 1; i < data.length; i++) {
      let sum = 0;
      for (let j = 0; j < period; j++) {
        sum += data[i - j].close;
      }
      result.push({ time: data[i].time, value: sum / period });
    }
    return result;
  }

  function updateLegend(param: any) {
    const barData = param.seriesData.get(barSeries);
    if (barData) {
      const dataPoint = data.find((d) => d.time === barData.time);
      if (!dataPoint) return;

      const previousDataPoint = data[data.indexOf(dataPoint) - 1];
      const previousClose = previousDataPoint ? previousDataPoint.close : dataPoint.open;

      const priceChange = barData.close - previousClose;
      const percentageChange = (priceChange / previousClose) * 100;
      const isPositive = priceChange >= 0;

      legendContainer.innerHTML = `
        <div class="flex items-center space-x-4">
          <div class="flex flex-col">
            <span class="text-sm font-bold text-gray-200">${stockName}</span>
          </div>
          <div class="flex flex-col">
            <span class="text-sm font-semibold text-gray-200">${formatPrice(barData.close)}</span>
          </div>
          <div class="flex flex-col">
            <span class="text-sm font-semibold" style="color: ${isPositive ? '#3AAB3A' : '#FF0000'}">
              ${isPositive ? '+' : ''}${formatPrice(priceChange)} (${formatPercentage(percentageChange)})
            </span>
          </div>
        </div>
      `;
    }
  }

  function initializeChart() {
    if (!chartContainer) return;

    chart = createChart(chartContainer, {
      layout: {
        background: {
          type: ColorType.Solid,
          color: '#000000'
        },
        textColor: '#CCCCCC',
      },
      grid: {
        vertLines: { visible: false },
        horzLines: { visible: false },
      },
      timeScale: {
        timeVisible: false,
        rightOffset: 5,
        minBarSpacing: 1,
        borderColor: '#444444',
      },
    });

    barSeries = chart.addBarSeries({
      upColor: '#3AAB3A',
      downColor: '#FF0000',
      thinBars: false,
      priceFormat: {
        type: 'price',
        precision: 2,
        minMove: 0.01,
      },
    });

    

    maSeries = chart.addLineSeries({ color: '#3b82f6', lineWidth: 1 });
  
    barSeries.priceScale().applyOptions({
      scaleMargins: {
        top: 0.2,
        bottom: 0.2,
      },
      borderColor: '#444444',
      mode: 1,
    });

    updateChartData();
    setInitialLegend();

    chart.subscribeCrosshairMove(updateLegend);

    chartContainer.addEventListener('mouseleave', () => {
      setInitialLegend();
    });
  }

  function updateChartData() {
    if (barSeries && data && data.length > 0) {
      const barData = data.map(({ time, high, low, close }, index) => {
        const previousClose = index > 0 ? data[index - 1].close : close;
        const isUp = close >= previousClose;
        return {
          time,
          open: close,
          high,
          low,
          close,
          color: isUp ? '#3AAB3A' : '#FF0000',
        };
      });

      
    

      barSeries.setData(barData);

      maSeries.setData(calculateMovingAverage(data, 10));
      
      

      chart.timeScale().fitContent();
      setInitialLegend();
    }
  }

  function setInitialLegend() {
    if (data && data.length > 0) {
      const lastDataPoint = data[data.length - 1];
      updateLegend({
        seriesData: new Map([
          [barSeries, lastDataPoint],
        ]),
      });
    }
  }

  function adjustChartSize() {
    if (chart && chartContainer) {
      requestAnimationFrame(() => {
        const newWidth = chartContainer.clientWidth;
        const newHeight = chartContainer.clientHeight;
        chart.applyOptions({
          width: newWidth,
          height: newHeight,
        });
        chart.timeScale().fitContent();
      });
    }
  }

  function handleResize() {
    requestAnimationFrame(() => {
      if (chartContainer) {
        adjustChartSize();
      }
    });
  }

  onMount(() => {
    initializeChart();

    resizeObserver = new ResizeObserver(() => {
      requestAnimationFrame(adjustChartSize);
    });

    if (chartContainer) {
      resizeObserver.observe(chartContainer);
    }

    window.addEventListener('orientationchange', () => {
      setTimeout(adjustChartSize, 100);
    });

    document.addEventListener('fullscreenchange', adjustChartSize);

    return () => {
      if (resizeObserver) {
        resizeObserver.disconnect();
      }
      window.removeEventListener('orientationchange', () => {
        setTimeout(adjustChartSize, 100);
      });
      document.removeEventListener('fullscreenchange', adjustChartSize);
      if (chart) {
        chart.remove();
      }
    };
  });

  $: if (chart && data) {
    updateChartData();
  }
</script>

<div class="chart-container relative w-full h-full min-h-[300px]">
  <div bind:this={chartContainer} class="w-full h-full"></div>
  <div 
    bind:this={legendContainer} 
    class="absolute top-1 left-1 z-10 font-sans p-1"
  ></div>
</div>
