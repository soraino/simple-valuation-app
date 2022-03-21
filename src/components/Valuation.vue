<template>
  <section>
    <div style="margin-top: 10em; margin-bottom: 2em">
      <h1 class="title is-3">
        Company Name:
        <span v-if="quoteInfo != null">{{ quoteInfo.name }}</span>
      </h1>
      <p>
        Current Price:
        <b v-if="quoteInfo != null">${{ quoteInfo.price }}</b>
      </p>
    </div>
    <b-field label="Number of fiscal years shown">
      <b-numberinput
        :min="1"
        :max="10"
        icon-pack="fas"
        v-model="number"
      ></b-numberinput>
    </b-field>

    <div class="mt-5">
      <b-table
        class="mt-3"
        :data="data"
        :columns="columns"
        bordered
        striped
        hoverable
      >
        <template #empty>
          <div class="has-text-centered">No records</div>
        </template>
      </b-table>
    </div>

    <p>Price to Book actual price: {{ priceToBookActPrice }}</p>
    <p>Price to Sales actual price: {{ priceToSaleActPrice }}</p>
  </section>
</template>
<script>
export default {
  name: "Valuation",
  props: {
    symbol: String,
  },
  data() {
    return {
      number: 4,
      quoteInfo: null,
      ttmData: null,
      data: [],
    };
  },
  watch: {
    symbol: async function (newVal) {
      if (!newVal.length) return;
      try {
        const keyMetricsRes = await this.$http.get(
          `https://financialmodelingprep.com/api/v3/ratios/${newVal}?limit=10&apikey=${process.env.VUE_APP_ROOT_API_KEY}`
        );

        const { data } = keyMetricsRes;

        this.data = [];
        if (data.length == 0) {
          alert("No data available for this company");
          return;
        }
        const reducedData = data.reduce(
          (acc, curr) => {
            if (acc.ret.length == 0) {
              const divYieldObj = { type: "Dividend Yield" };
              const priceToBook = { type: "Price to Book" };
              const priceToSales = { type: "Price to Sales" };
              acc.ret.push(divYieldObj);
              acc.ret.push(priceToBook);
              acc.ret.push(priceToSales);
            } 
             acc.ret[0][curr.date.substring(0, 4)] =
                curr.dividendYield == null ? "-" : curr.dividendYield.toFixed(4);
              acc.ret[1][curr.date.substring(0, 4)] = curr.priceToBookRatio.toFixed(4);
              acc.ret[2][curr.date.substring(0, 4)] = curr.priceToSalesRatio.toFixed(4);
            acc.summ.pb += curr.priceToBookRatio;
            acc.summ.ps += curr.priceToSalesRatio;
            return acc;
          },
          { ret: [], summ: { pb: 0, ps: 0 } }
        );

        reducedData.ret[1]["totalSum"] = reducedData.summ.pb;
        reducedData.ret[2]["totalSum"] = reducedData.summ.ps;

        reducedData.ret[0]["fairPrice"] = "-";
        reducedData.ret[1]["fairPrice"] = (reducedData.summ.pb / this.number).toFixed(4);
        reducedData.ret[2]["fairPrice"] = (reducedData.summ.ps / this.number).toFixed(4);

        const quoteDataRes = await this.$http.get(
          `https://financialmodelingprep.com/api/v3/quote/${newVal}?limit=10&apikey=${process.env.VUE_APP_ROOT_API_KEY}`
        );

        this.quoteInfo = quoteDataRes.data[0];

        setTimeout(() => {
          const ttmData = ttmDataRes.data[0];
          this.ttmData = ttmData;
          reducedData.ret[0]["ttm"] =
            ttmData.dividendYieldPercentageTTM != null
              ? ttmData.dividendYieldPercentageTTM.toFixed(4) + "%"
              : "-";
          reducedData.ret[1]["ttm"] =
            ttmData.tangibleBookValuePerShareTTM != null
              ? (
                  this.quoteInfo.price / ttmData.tangibleBookValuePerShareTTM
                ).toFixed(4)
              : "-";
          reducedData.ret[2]["ttm"] =
            ttmData.priceToSalesRatioTTM != null
              ? ttmData.priceToSalesRatioTTM.toFixed(4)
              : "-";
          this.data = reducedData.ret;
        }, 500);
        const ttmDataRes = await this.$http.get(
          `https://financialmodelingprep.com/api/v3/key-metrics-ttm/${newVal}?limit=10&apikey=${process.env.VUE_APP_ROOT_API_KEY}`
        );
      } catch (error) {
        this.data = [];
        this.quoteInfo = null;
        if (error.response.status == 403) {
          alert("This stock's infomation is not available for the free plan.");
        }
      } finally {
        this.isFetching = false;
      }
    },
    number(newVal) {
      this.data[1]["fairPrice"] = (this.data[1]["totalSum"] / newVal).toFixed(
        4
      );
      this.data[2]["fairPrice"] = (this.data[2]["totalSum"] / newVal).toFixed(
        4
      );
    },
  },
  computed: {
    priceToBookActPrice() {
      if (this.data.length <= 0) return 0;
      return (
        this.data[1]["fairPrice"] * this.ttmData.tangibleBookValuePerShareTTM
      ).toFixed(4);
    },
    priceToSaleActPrice() {
      if (this.data.length <= 0) return 0;
      return (
        this.data[2]["fairPrice"] * this.ttmData.revenuePerShareTTM
      ).toFixed(4);
    },
    columns() {
      const columns = [
        {
          field: "fairPrice",
          label: "Fair Price",
        },
        {
          field: "ttm",
          label: "TTM",
        },
      ];
      const currDate = new Date();
      for (let i = 0; i < this.number; i++) {
        const yearString = (currDate.getFullYear() - i - 1).toString();
        columns.unshift({
          field: yearString,
          label: yearString,
        });
      }
      columns.unshift({
        field: "type",
        label: "Valuation Type",
      });
      return columns;
    },
  },
};
</script>