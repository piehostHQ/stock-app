<template>
  <div class="lineChart">
    <div class="chartAndButtonsContainer">
      <form @submit.prevent="fetchAndPublishStockData" id="stockSymbolSearchForm">
        <input
          type="text"
          name="stockSymbolSearch"
          placeholder="Enter the symbol of the stock you want to search"
          id="stockSymbolSearchBar"
          v-model="stockSymbol"
        />
        <button type="submit" class="submit">Search</button>
      </form>
      <br />
      <p>Active Stock Symbol: {{ stockSymbol }}</p>
      <br />
      <div id="chartContainer">
        <canvas id="myChart" width="400px" height="400px"></canvas>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed } from "vue";
import Chart from "chart.js/auto";
import axios from "axios";
import PieSocket from "piesocket-js";

let myChart;
let channel;
export default {
  name: "LineChart",
  data() {
    return {
      stockSymbol: "IBM",
      stockMarketHistoryDates: [],
      stockMarketHistoryPrices: [],
    };
  },
  mounted() {
    this.createChart();
    this.connectToPieSocket();
  },
  methods: {
    createChart() {
      const ctx = document.getElementById("myChart");
      myChart = new Chart(ctx, {
        type: "line",
        data: {
          labels: this.stockMarketHistoryDates,
          datasets: [
            {
              label: "Stock Market Price",
              data: this.stockMarketHistoryPrices,
              fill: false,
              borderColor: "rgb(75, 192, 192)",
              tension: 0,
              options: {
                responsive: true,
                maintainAspectRatio: false,
              },
            },
          ],
        },
      });
    },
    updateChart() {
      myChart.data.labels = this.stockMarketHistoryDates;
      myChart.data.datasets[0].data = this.stockMarketHistoryPrices;
      myChart.update();
    },
    connectToPieSocket() {
      let piesocket = new PieSocket({
        clusterId: "YOUR CLUSTER ID",
        apiKey: "YOUR API KEY",
        notifySelf: true,
        presence: true,
        userId: "user_" + Math.floor(Math.random() * 1000),
      });

      piesocket.subscribe("stock-data").then((ch) => {
        channel = ch;
        ch.listen("new_stock_data", (data) => {
          this.stockSymbol = data.symbol;
          this.stockMarketHistoryDates = data.dates;
          this.stockMarketHistoryPrices = data.prices;
          this.updateChart();
          console.log(
            this.stockSymbol,
            this.stockMarketHistoryDates,
            this.stockMarketHistoryPrices
          );
        });

        // Fetch initial stock data
        this.fetchAndPublishStockData();
      });
    },
    fetchAndPublishStockData() {
      if (!channel) {
        console.error("Channel is not initialized yet.");
        return;
      }
      
      let AlphaVantageAPI_URL_LINK =
        "https://www.alphavantage.co/query?function=TIME_SERIES_MONTHLY_ADJUSTED&symbol=" +
        this.stockSymbol +
        "&apikey=6S3XWKYVUZIUJZEF";

      axios.get(AlphaVantageAPI_URL_LINK).then((response) => {
        let stockMarketHistory = response.data["Monthly Adjusted Time Series"];
        let stockMarketHistoryDates = [];
        let stockMarketHistoryPrices = [];

        for (const property in stockMarketHistory) {
          let closingPrice = stockMarketHistory[property]["4. close"];
          let closingDate = property;
          stockMarketHistoryDates.unshift(closingDate);
          stockMarketHistoryPrices.unshift(closingPrice);
        }

        channel.publish("new_stock_data", {
          symbol: this.stockSymbol,
          dates: stockMarketHistoryDates,
          prices: stockMarketHistoryPrices,
        });
      });
    },
  },
};
</script>

<style scoped>
#chartContainer {
  width: 1000px;
  height: 300px;
  margin: 0 auto;
}
#stockSymbolSearchBar {
  padding: 10px;
}
#stockSymbolSearchForm .submit {
  padding: 10px 20px;
}
</style>