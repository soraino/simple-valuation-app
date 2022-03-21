<template>
  <div id="app">
    <section class="hero is-light is-fullheight">
      <div class="hero-head">
        <div class="container has-text-centered">
          <h1 class="title is-2">Simple Stock Valuation App</h1>
        </div>
      </div>
      <div class="hero-body" style="align-items: inherit !important;">
        <div class="container">
          <b-field label="Company search bar">
            <b-autocomplete
              class="mb-5"
              :data="data"
              placeholder="e.g. Apple"
              field="name"
              :loading="isFetching"
              @typing="getAsyncData"
              @select="(option) => (selected = option)"
            >
              <template slot-scope="props">
                <div class="media">
                  <div class="media-content">
                    {{ props.option.name }}
                    <br />
                    <small>
                      {{ props.option.exchangeShortName }}:{{
                        props.option.symbol
                      }}
                    </small>
                  </div>
                </div>
              </template>
            </b-autocomplete>
          </b-field>
          <Valuation :symbol="selected == null ? '' : selected.symbol">
          </Valuation>
        </div>
      </div>
    </section>
  </div>
</template>

<script>
import { debounce } from "lodash";
import Valuation from "./components/Valuation.vue";
export default {
  name: "App",
  components: {
    Valuation,
  },
  data() {
    return {
      searchVal: "",
      data: [],
      isFetching: false,
      selected: null,
    };
  },
  methods: {
    getAsyncData: debounce(async function (name) {
      if (name.length < 2) {
        this.data = [];
        return;
      }
      this.isFetching = true;
      try {
        const info = await this.$http.get(
          `https://financialmodelingprep.com/api/v3/search?query=${name}&limit=10&apikey=${process.env.VUE_APP_ROOT_API_KEY}`
        );
        const { data } = info;
        this.data = [];
        data.forEach((item) => this.data.push(item));
      } catch (error) {
        this.data = [];
        throw error;
      } finally {
        this.isFetching = false;
      }
    }, 500),
  },
};
</script>

